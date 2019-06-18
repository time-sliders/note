继第一篇分析的代码，我们继续回到 main loop 循环来看

```java
/*
 * Main loop
 */
while (running) {
    switch (getPeerState()) {
        case LOOKING:
					....
        case OBSERVING:
					...
        case FOLLOWING:
          ...
        case LEADING:
            LOG.info("LEADING");
            try {
              	// 创建 leader 实例
                setLeader(makeLeader(logFactory));
              	// 核心方法，执行 lead 逻辑
                leader.lead();
                setLeader(null);
            } catch (Exception e) {
                LOG.warn("Unexpected exception", e);
            } finally {
                if (leader != null) {
                    leader.shutdown("Forcing shutdown");
                    setLeader(null);
                }
                updateServerState();
            }
            break;
    }
    start_fle = Time.currentElapsedTime();
}
```

```java
/**
 * This method is main function that is called to lead
 *
 * @throws IOException
 * @throws InterruptedException
 */
void lead() throws IOException, InterruptedException {
    self.end_fle = Time.currentElapsedTime();
    LOG.info("LEADING - LEADER ELECTION TOOK - " +
          (self.end_fle - self.start_fle) + " " + QuorumPeer.FLE_TIME_UNIT);
    self.start_fle = 0;
    self.end_fle = 0;

    zk.registerJMX(new LeaderBean(this, zk), self.jmxLocalPeerBean);

    try {
        self.tick.set(0);
      	// 从本地文件恢复数据  
        zk.loadData();
				// leader 的状态信息
        leaderStateSummary = new StateSummary(self.getCurrentEpoch(), zk.getLastProcessedZxid());

        // Start thread that waits for connection requests from
        // new followers.
      	/*这里是启动一个 ServerSocket 监听来自 follower 的请求并处理*/
        cnxAcceptor = new LearnerCnxAcceptor();
        cnxAcceptor.start();

      	// 等待足够多的 follower 进来，代表自己确实是 leader，此处 lead 线程可能会等待
        long epoch = getEpochToPropose(self.getId(), self.getAcceptedEpoch());
```

 等待 Follower 连接

```java
public long getEpochToPropose(long sid, long lastAcceptedEpoch) throws InterruptedException, IOException {
    synchronized(connectingFollowers) {
        if (!waitingForNewEpoch) {
            return epoch;
        }
        if (lastAcceptedEpoch >= epoch) {
            epoch = lastAcceptedEpoch+1;
        }
        // 将自己加入连接队伍中，方便后续判断 lead 是否有效  
        connectingFollowers.add(sid);
        QuorumVerifier verifier = self.getQuorumVerifier();
      	// 如果足够多的 follower 进入，选举有效，则无需等待，并通知其他的等待线程，类似于 Barrier  
        if (connectingFollowers.contains(self.getId()) &&
							verifier.containsQuorum(connectingFollowers)) {
            waitingForNewEpoch = false;
            self.setAcceptedEpoch(epoch);
            connectingFollowers.notifyAll();
        } else {
          	// 如果进入的 follower 不够，则进入等待，超时即为 initLimit 时间，  
            long start = Time.currentElapsedTime();
            long cur = start;
            long end = start + self.getInitLimit()*self.getTickTime();
            while(waitingForNewEpoch && cur < end) {
                connectingFollowers.wait(end - cur);
                cur = Time.currentElapsedTime();
            }
          	// 超时了，退出 lead 过程，重新发起选举  
            if (waitingForNewEpoch) {
                throw new InterruptedException("Timeout while waiting for epoch from quorum");
            }
        }
        return epoch;
    }
}
```

好的，这个时候我们假设其他 Follower 还没连接进来，那 Leader 就会在此等待。再来看 Follower 的初始化过程

```java
case FOLLOWING:
    try {
        LOG.info("FOLLOWING");
      	// 初始化 Follower 对象  
        setFollower(makeFollower(logFactory));
        // follow 动作
        follower.followLeader();
    } catch (Exception e) {
        LOG.warn("Unexpected exception", e);
    } finally {
        follower.shutdown();
        setFollower(null);
        updateServerState();
    }
    break;
```

```java
/**
 * the main method called by the follower to follow the leader
 *
 * @throws InterruptedException
 */
void followLeader() throws InterruptedException {
    ...
    try {
         // 根据 sid 找到对应 leader，拿到 lead 连接信息  
        InetSocketAddress addr = findLeader();            
        try {
          	// 连接 leader  
            connectToLeader(addr);
          	// 注册 follower，根据 Leader和 Follower 协议，主要是同步选举轮数  
            long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);
            if (self.isReconfigStateChange())
               throw new Exception("learned about role change");
            //check to see if the leader zxid is lower than ours
            //this should never happen but is just a safety check
            long newEpoch = ZxidUtils.getEpochFromZxid(newEpochZxid);
            if (newEpoch < self.getAcceptedEpoch()) {
                LOG.error("Proposed leader epoch " + ZxidUtils.zxidToString(newEpochZxid)
                        + " is less than our accepted epoch " + ZxidUtils.zxidToString(self.getAcceptedEpoch()));
                throw new IOException("Error: Epoch of leader is lower");
            }
          	// 同步数据
            syncWithLeader(newEpochZxid);                
            QuorumPacket qp = new QuorumPacket();
          	// 接受 Leader 消息，执行并反馈给 Leader，线程在此自旋  
            while (this.isRunning()) {
                readPacket(qp);
                processPacket(qp);
            }
...
```

```java
/**
 * Establish a connection with the Leader found by findLeader. Retries
 * until either initLimit time has elapsed or 5 tries have happened. 
 * @param addr - the address of the Leader to connect to.
 * @throws IOException - if the socket connection fails on the 5th attempt
 * @throws ConnectException
 * @throws InterruptedException
 */
protected void connectToLeader(InetSocketAddress addr) 
throws IOException, ConnectException, InterruptedException {
    sock = new Socket();        
    sock.setSoTimeout(self.tickTime * self.initLimit);
		// 设置读超时时间为 initLimit 对应时间  
    int initLimitTime = self.tickTime * self.initLimit;
    int remainingInitLimitTime = initLimitTime;
    long startNanoTime = nanoTime();
		// 重试5次，失败后退出 follower 角色，重新选举  
    for (int tries = 0; tries < 5; tries++) {
        try {
            // recalculate the init limit time because retries sleep for 1000 milliseconds
            remainingInitLimitTime = initLimitTime - (int)((nanoTime() - startNanoTime) / 1000000);
            if (remainingInitLimitTime <= 0) {
                LOG.error("initLimit exceeded on retries.");
                throw new IOException("initLimit exceeded on retries.");
            }

            sockConnect(sock, addr, Math.min(self.tickTime * self.syncLimit, remainingInitLimitTime));
            sock.setTcpNoDelay(nodelay);
            break;
        } catch (IOException e) {
          	// 连接超时处理
            remainingInitLimitTime = initLimitTime - (int)((nanoTime() - startNanoTime) / 1000000);

            if (remainingInitLimitTime <= 1000) {
                LOG.error("Unexpected exception, initLimit exceeded. tries=" + tries +
                         ", remaining init limit=" + remainingInitLimitTime +
                         ", connecting to " + addr,e);
                throw e;
            } else if (tries >= 4) {
                LOG.error("Unexpected exception, retries exceeded. tries=" + tries +
                         ", remaining init limit=" + remainingInitLimitTime +
                         ", connecting to " + addr,e);
                throw e;
            } else {
                LOG.warn("Unexpected exception, tries=" + tries +
                        ", remaining init limit=" + remainingInitLimitTime +
                        ", connecting to " + addr,e);
                sock = new Socket();
                sock.setSoTimeout(self.tickTime * self.initLimit);
            }
        }
        Thread.sleep(1000);
    }
    leaderIs = BinaryInputArchive.getArchive(new BufferedInputStream(
            sock.getInputStream()));
    bufferedOutput = new BufferedOutputStream(sock.getOutputStream());
    leaderOs = BinaryOutputArchive.getArchive(bufferedOutput);
}   
```

 假设这里 Follower 顺利连上了 Leader，此时 Leader 端会为每个 Follower 启动单独IO线程，请看 LearnerCnxAcceptor 代码

```java
@Override
public void run() {
    try {
        while (!stop) {
            try{
                 // 线程在此等待连接  
                Socket s = ss.accept();
                // start with the initLimit, once the ack is processed
                // in LearnerHandler switch to the syncLimit
              	// 读超时设为 initLimit 时间  
                s.setSoTimeout(self.tickTime * self.initLimit);
                s.setTcpNoDelay(nodelay);
              	// 为每个 Follower 启动单独线程，处理 IO  
                LearnerHandler fh = new LearnerHandler(s, Leader.this);
                fh.start();
            } catch (SocketException e) {
                if (stop) {
                    LOG.info("exception while shutting down acceptor: "
                            + e);

                    // When Leader.shutdown() calls ss.close(),
                    // the call to accept throws an exception.
                    // We catch and set stop to true.
                    stop = true;
                } else {
                    throw e;
                }
            }
        }
    } catch (Exception e) {
        LOG.warn("Exception while accepting follower", e.getMessage());
        handleException(this.getName(), e);
    }
}
```

Leader 端为 Follower 建立 IO 线程，其处理过程和 Follower 自身的主线程根据协议相互交互，以下将通过数据交换场景式分析这个过程，Leader 端 IO 线程 LearnerHandler 启动

**LearnerHandler**#run

```java
ia = BinaryInputArchive.getArchive(new BufferedInputStream(sock.getInputStream()));
bufferedOutput = new BufferedOutputStream(sock.getOutputStream());
oa = BinaryOutputArchive.getArchive(bufferedOutput);
// IO线程等待 Follower 发送包  
QuorumPacket qp = new QuorumPacket();
ia.readRecord(qp, "packet");
```

Follower 端进入 registerWithLeader 处理

```java
   /*
    * Send follower info, including last zxid and sid
    */
	 long lastLoggedZxid = self.getLastLoggedZxid();
   QuorumPacket qp = new QuorumPacket();                
   qp.setType(pktType);
   qp.setZxid(ZxidUtils.makeZxid(self.getAcceptedEpoch(), 0));
   
   /*
    * Add sid to payload
    */
   LearnerInfo li = new LearnerInfo(self.getId(), 0x10000, self.getQuorumVerifier().getVersion());
   ByteArrayOutputStream bsid = new ByteArrayOutputStream();
   BinaryOutputArchive boa = BinaryOutputArchive.getArchive(bsid);
   boa.writeRecord(li, "LearnerInfo");
   qp.setData(bsid.toByteArray());
   
   writePacket(qp, true);
   readPacket(qp);        
```

LearnerHandler 端收到包处理

```java
byte learnerInfoData[] = qp.getData();
if (learnerInfoData != null) {
  //反序列化 LearnerInfo  
  ByteBuffer bbsid = ByteBuffer.wrap(learnerInfoData);
  if (learnerInfoData.length >= 8) {
    this.sid = bbsid.getLong();
  } 
  if (learnerInfoData.length >= 12) {
    this.version = bbsid.getInt(); // protocolVersion
  }
  if (learnerInfoData.length >= 20) {
    long configVersion = bbsid.getLong();
    if (configVersion > leader.self.getQuorumVerifier().getVersion()) {
      throw new IOException("Follower is ahead of the leader (has a later activated configuration)");                     
    }
  }
} else {
  this.sid = leader.followerCounter.getAndDecrement();
}

if (leader.self.getView().containsKey(this.sid)) {
  LOG.info("Follower sid: " + this.sid + " : info : "
           + leader.self.getView().get(this.sid).toString());
} else {
  LOG.info("Follower sid: " + this.sid + " not in the current config " + Long.toHexString(leader.self.getQuorumVerifier().getVersion()));
}

if (qp.getType() == Leader.OBSERVERINFO) {
  learnerType = LearnerType.OBSERVER;
}

//follower的选举轮数  
long lastAcceptedEpoch = ZxidUtils.getEpochFromZxid(qp.getZxid());

long peerLastZxid;
StateSummary ss = null;
long zxid = qp.getZxid();
// 将 Follower加入到 connectingFollowers 中，因为满足半数机器的条件，此时在此等待的 leader 主线程会退出等待，继续往下处理  
long newEpoch = leader.getEpochToPropose(this.getSid(), lastAcceptedEpoch);
long newLeaderZxid = ZxidUtils.makeZxid(newEpoch, 0);

if (this.getVersion() < 0x10000) {
  // we are going to have to extrapolate the epoch information
  long epoch = ZxidUtils.getEpochFromZxid(zxid);
  ss = new StateSummary(epoch, zxid);
  // fake the message
  leader.waitForEpochAck(this.getSid(), ss);
} else {
  //发一个Leader.LEADERINFO 包，带上新的 epoch id  
  byte ver[] = new byte[4];
  ByteBuffer.wrap(ver).putInt(0x10000);
  QuorumPacket newEpochPacket = new QuorumPacket(Leader.LEADERINFO, newLeaderZxid, ver, null);
  oa.writeRecord(newEpochPacket, "packet");
  bufferedOutput.flush();
  QuorumPacket ackEpochPacket = new QuorumPacket();
  //等待 Follower 响应  
  ia.readRecord(ackEpochPacket, "packet");
  if (ackEpochPacket.getType() != Leader.ACKEPOCH) {
    LOG.error(ackEpochPacket.toString()
              + " is not ACKEPOCH");
    return;
  }
  ByteBuffer bbepoch = ByteBuffer.wrap(ackEpochPacket.getData());
  ss = new StateSummary(bbepoch.getInt(), ackEpochPacket.getZxid());
  leader.waitForEpochAck(this.getSid(), ss);
}
```

此时 Follower 收到 LEADERINFO 包处理：

```java
final long newEpoch = ZxidUtils.getEpochFromZxid(qp.getZxid());
if (qp.getType() == Leader.LEADERINFO) {
  // we are connected to a 1.0 server so accept the new epoch and read the next packet
  leaderProtocolVersion = ByteBuffer.wrap(qp.getData()).getInt();
  byte epochBytes[] = new byte[4];
  final ByteBuffer wrappedEpochBytes = ByteBuffer.wrap(epochBytes);
  // 将自己的 epoch 发给 leader
  if (newEpoch > self.getAcceptedEpoch()) {
    wrappedEpochBytes.putInt((int)self.getCurrentEpoch());
    self.setAcceptedEpoch(newEpoch);
  } else if (newEpoch == self.getAcceptedEpoch()) {
    // since we have already acked an epoch equal to the leaders, we cannot ack
    // again, but we still need to send our lastZxid to the leader so that we can
    // sync with it if it does assume leadership of the epoch.
    // the -1 indicates that this reply should not count as an ack for the new epoch
    wrappedEpochBytes.putInt(-1);
  } else {
    throw new IOException("Leaders epoch, " + newEpoch + " is less than accepted epoch, " + self.getAcceptedEpoch());
  }
  // 发送一个 Leader.ACKEPOCH 包，带上自己的最大 zxid  
  QuorumPacket ackNewEpoch = new QuorumPacket(Leader.ACKEPOCH, lastLoggedZxid, epochBytes, null);
  writePacket(ackNewEpoch, true);
  return ZxidUtils.makeZxid(newEpoch, 0);
```

LearnerHandler 收到 Leader.ACKEPOCH 后进入 waitForEpochAck 处理

```java
public void waitForEpochAck(long id, StateSummary ss) throws IOException, InterruptedException {
    synchronized(electingFollowers) {
        if (electionFinished) {
            return;
        }
        if (ss.getCurrentEpoch() != -1) {
            if (ss.isMoreRecentThan(leaderStateSummary)) {
                throw new IOException("Follower is ahead of the leader, leader summary: " 
                                                + leaderStateSummary.getCurrentEpoch()
                                                + " (current epoch), "
                                                + leaderStateSummary.getLastZxid()
                                                + " (last zxid)");
            }
          	// 将 follower 添加到等待集合 
            electingFollowers.add(id);
        }
        QuorumVerifier verifier = self.getQuorumVerifier();
      	// 判断是否满足选举条件，如果不满足进入等待，满足则通知其他等待线程，类似于Barrier  
        if (electingFollowers.contains(self.getId()) && verifier.containsQuorum(electingFollowers)) {
          	// 满足法定仲裁，唤醒其他等待
            electionFinished = true;
            electingFollowers.notifyAll();
        } else {
          	// 进入等待
            long start = Time.currentElapsedTime();
            long cur = start;
            long end = start + self.getInitLimit()*self.getTickTime();
            while(!electionFinished && cur < end) {
                electingFollowers.wait(end - cur);
                cur = Time.currentElapsedTime();
            }
            if (!electionFinished) {
                throw new InterruptedException("Timeout while waiting for epoch to be acked by quorum");
            }
        }
    }
}
```

假设 IO 线程在此等待，此时 leader 主线程在 getEpochToPropose 恢复后继续执行

```java
long epoch = getEpochToPropose(self.getId(), self.getAcceptedEpoch());

zk.setZxid(ZxidUtils.makeZxid(epoch, 0));

synchronized(this){
    lastProposed = zk.getZxid();
}
// 发起一个 NEWLEADER 投票  
newLeaderProposal.packet = new QuorumPacket(NEWLEADER, zk.getZxid(),
       null, null);


if ((newLeaderProposal.packet.getZxid() & 0xffffffffL) != 0) {
    LOG.info("NEWLEADER proposal has Zxid of "
            + Long.toHexString(newLeaderProposal.packet.getZxid()));
}

QuorumVerifier lastSeenQV = self.getLastSeenQuorumVerifier();
QuorumVerifier curQV = self.getQuorumVerifier();
if (curQV.getVersion() == 0 && curQV.getVersion() == lastSeenQV.getVersion()) {
    // This was added in ZOOKEEPER-1783. The initial config has version 0 (not explicitly
    // specified by the user; the lack of version in a config file is interpreted as version=0). 
    // As soon as a config is established we would like to increase its version so that it
    // takes presedence over other initial configs that were not established (such as a config
    // of a server trying to join the ensemble, which may be a partial view of the system, not the full config). 
    // We chose to set the new version to the one of the NEWLEADER message. However, before we can do that
    // there must be agreement on the new version, so we can only change the version when sending/receiving UPTODATE,
    // not when sending/receiving NEWLEADER. In other words, we can't change curQV here since its the committed quorum verifier, 
    // and there's still no agreement on the new version that we'd like to use. Instead, we use 
    // lastSeenQuorumVerifier which is being sent with NEWLEADER message
    // so its a good way to let followers know about the new version. (The original reason for sending 
    // lastSeenQuorumVerifier with NEWLEADER is so that the leader completes any potentially uncommitted reconfigs
    // that it finds before starting to propose operations. Here we're reusing the same code path for 
    // reaching consensus on the new version number.)
    
    // It is important that this is done before the leader executes waitForEpochAck,
    // so before LearnerHandlers return from their waitForEpochAck
    // hence before they construct the NEWLEADER message containing
    // the last-seen-quorumverifier of the leader, which we change below
   try {
       QuorumVerifier newQV = self.configFromString(curQV.toString());
       newQV.setVersion(zk.getZxid());
       self.setLastSeenQuorumVerifier(newQV, true);    
   } catch (Exception e) {
       throw new IOException(e);
   }
}

newLeaderProposal.addQuorumVerifier(self.getQuorumVerifier());
if (self.getLastSeenQuorumVerifier().getVersion() > self.getQuorumVerifier().getVersion()){
   newLeaderProposal.addQuorumVerifier(self.getLastSeenQuorumVerifier());
}

// We have to get at least a majority of servers in sync with
// us. We do this by waiting for the NEWLEADER packet to get
// acknowledged
//等待 follower 与 Leader sync
waitForEpochAck(self.getId(), leaderStateSummary);
```

由于之前已经有 follower 进来，满足选举条件，则 IO 线程和 leader 主线程都继续往下执行，先看 leader 主线程

```java
// 当前选票轮数
self.setCurrentEpoch(epoch);    

try {
  	// 这里是等待 NEWLEADER 的 ACK 达到法定人数
    waitForNewLeaderAck(self.getId(), zk.getZxid(), LearnerType.PARTICIPANT);
} catch (InterruptedException e) {
  	// 异常中断，重新选举
    shutdown("Waiting for a quorum of followers, only synced with sids: [ "
            + newLeaderProposal.ackSetsToString() + " ]");
    HashSet<Long> followerSet = new HashSet<Long>();

    for(LearnerHandler f : getLearners()) {
        if (self.getQuorumVerifier().getVotingMembers().containsKey(f.getSid())){
            followerSet.add(f.getSid());
        }
    }    
    boolean initTicksShouldBeIncreased = true;
    for (Proposal.QuorumVerifierAcksetPair qvAckset:newLeaderProposal.qvAcksetPairs) {
        if (!qvAckset.getQuorumVerifier().containsQuorum(followerSet)) {
            initTicksShouldBeIncreased = false;
            break;
        }
    }                  
    if (initTicksShouldBeIncreased) {
        LOG.warn("Enough followers present. "+
                "Perhaps the initTicks need to be increased.");
    }
    return;
}
```

这个时候 LearnerHandler 线程继续执行

```java
/**
 * Determine if we need to sync with follower using DIFF/TRUNC/SNAP
 * and setup follower to receive packets from commit processor
 *
 * @param peerLastZxid
 * @param db
 * @param leader
 * @return true if snapshot transfer is needed.
 */
public boolean syncFollower(long peerLastZxid, ZKDatabase db, Leader leader) {
    /*
     * When leader election is completed, the leader will set its
     * lastProcessedZxid to be (epoch < 32). There will be no txn associated
     * with this zxid.
     *
     * The learner will set its lastProcessedZxid to the same value if
     * it get DIFF or SNAP from the leader. If the same learner come
     * back to sync with leader using this zxid, we will never find this
     * zxid in our history. In this case, we will ignore TRUNC logic and
     * always send DIFF if we have old enough history
     */
    boolean isPeerNewEpochZxid = (peerLastZxid & 0xffffffffL) == 0;
    // Keep track of the latest zxid which already queued
    long currentZxid = peerLastZxid;
    boolean needSnap = true;
    boolean txnLogSyncEnabled = (db.getSnapshotSizeFactor() >= 0);
    ReentrantReadWriteLock lock = db.getLogLock();
    ReadLock rl = lock.readLock();
    try {
        rl.lock();
        long maxCommittedLog = db.getmaxCommittedLog();
        long minCommittedLog = db.getminCommittedLog();
        long lastProcessedZxid = db.getDataTreeLastProcessedZxid();

        LOG.info("Synchronizing with Follower sid: {} maxCommittedLog=0x{}"
                + " minCommittedLog=0x{} lastProcessedZxid=0x{}"
                + " peerLastZxid=0x{}", getSid(),
                Long.toHexString(maxCommittedLog),
                Long.toHexString(minCommittedLog),
                Long.toHexString(lastProcessedZxid),
                Long.toHexString(peerLastZxid));

        if (db.getCommittedLog().isEmpty()) {
            /*
             * It is possible that commitedLog is empty. In that case
             * setting these value to the latest txn in leader db
             * will reduce the case that we need to handle
             *
             * Here is how each case handle by the if block below
             * 1. lastProcessZxid == peerZxid -> Handle by (2)
             * 2. lastProcessZxid < peerZxid -> Handle by (3)
             * 3. lastProcessZxid > peerZxid -> Handle by (5)
             */
            minCommittedLog = lastProcessedZxid;
            maxCommittedLog = lastProcessedZxid;
        }

        /*
         * Here are the cases that we want to handle
         *
         * 1. Force sending snapshot (for testing purpose) 强制发送快照（测试目的）
         * 2. Peer and leader is already sync, send empty diff （follower 与 leader 已经同步，发送空的 diff指令）
         * 3. Follower has txn that we haven't seen. This may be old leader
         *    so we need to send TRUNC. However, if peer has newEpochZxid,
         *    we cannot send TRUC since the follower has no txnlog
         *    followers 有一个当前 leader 没有的 txn，可能是因为之前是旧的 leader
         * 4. Follower is within committedLog range or already in-sync.
         *    We may need to send DIFF or TRUNC depending on follower's zxid
         *    We always send empty DIFF if follower is already in-sync
         * 5. Follower missed the committedLog. We will try to use on-disk
         *    txnlog + committedLog to sync with follower. If that fail,
         *    we will send snapshot
         */

        if (forceSnapSync) {
            // Force leader to use snapshot to sync with follower
            LOG.warn("Forcing snapshot sync - should not see this in production");
        } else if (lastProcessedZxid == peerLastZxid) {
            // Follower is already sync with us, send empty diff
            LOG.info("Sending DIFF zxid=0x" + Long.toHexString(peerLastZxid) +
                     " for peer sid: " +  getSid());
            queueOpPacket(Leader.DIFF, peerLastZxid);
            needOpPacket = false;
            needSnap = false;
        } else if (peerLastZxid > maxCommittedLog && !isPeerNewEpochZxid) {
            // Newer than commitedLog, send trunc and done
            // 如果follower超前了，则发送TRUNC包，让其和leader同步  
            LOG.debug("Sending TRUNC to follower zxidToSend=0x" +
                      Long.toHexString(maxCommittedLog) +
                      " for peer sid:" +  getSid());
            queueOpPacket(Leader.TRUNC, maxCommittedLog);
            currentZxid = maxCommittedLog;
            needOpPacket = false;
            needSnap = false;
        } else if ((maxCommittedLog >= peerLastZxid)
                && (minCommittedLog <= peerLastZxid)) {
            // Follower is within commitLog range
            LOG.info("Using committedLog for peer sid: " +  getSid());
            Iterator<Proposal> itr = db.getCommittedLog().iterator();
            currentZxid = queueCommittedProposals(itr, peerLastZxid,
                                                 null, maxCommittedLog);
            needSnap = false;
        } else if (peerLastZxid < minCommittedLog && txnLogSyncEnabled) {
            // Use txnlog and committedLog to sync

            // Calculate sizeLimit that we allow to retrieve txnlog from disk
            long sizeLimit = db.calculateTxnLogSizeLimit();
            // This method can return empty iterator if the requested zxid
            // is older than on-disk txnlog
            Iterator<Proposal> txnLogItr = db.getProposalsFromTxnLog(
                    peerLastZxid, sizeLimit);
            if (txnLogItr.hasNext()) {
                LOG.info("Use txnlog and committedLog for peer sid: " +  getSid());
                currentZxid = queueCommittedProposals(txnLogItr, peerLastZxid,
                                                     minCommittedLog, maxCommittedLog);

                LOG.debug("Queueing committedLog 0x" + Long.toHexString(currentZxid));
                Iterator<Proposal> committedLogItr = db.getCommittedLog().iterator();
                currentZxid = queueCommittedProposals(committedLogItr, currentZxid,
                                                     null, maxCommittedLog);
                needSnap = false;
            }
            // closing the resources
            if (txnLogItr instanceof TxnLogProposalIterator) {
                TxnLogProposalIterator txnProposalItr = (TxnLogProposalIterator) txnLogItr;
                txnProposalItr.close();
            }
        } else {
            LOG.warn("Unhandled scenario for peer sid: " +  getSid());
        }
        LOG.debug("Start forwarding 0x" + Long.toHexString(currentZxid) +
                  " for peer sid: " +  getSid());
        leaderLastZxid = leader.startForwarding(this, currentZxid);
    } finally {
        rl.unlock();
    }

    if (needOpPacket && !needSnap) {
        // This should never happen, but we should fall back to sending
        // snapshot just in case.
        LOG.error("Unhandled scenario for peer sid: " +  getSid() +
                 " fall back to use snapshot");
        needSnap = true;
    }

    return needSnap;
}
```

在我们这个集群里。由于是刚启动的，所以leader会直接发送DIFF包，然后再发送一个NEWLEADER包

接着 follower 收到包处理，在 syncWithLeader 中

```java
/**
 * Finally, synchronize our history with the Leader. 
 * @param newLeaderZxid
 * @throws IOException
 * @throws InterruptedException
 */
protected void syncWithLeader(long newLeaderZxid) throws Exception{
    QuorumPacket ack = new QuorumPacket(Leader.ACK, 0, null, null);
    QuorumPacket qp = new QuorumPacket();
    long newEpoch = ZxidUtils.getEpochFromZxid(newLeaderZxid);
    
    QuorumVerifier newLeaderQV = null;
    
    readPacket(qp);   
    LinkedList<Long> packetsCommitted = new LinkedList<Long>();
    LinkedList<PacketInFlight> packetsNotCommitted = new LinkedList<PacketInFlight>();
    synchronized (zk) {
      	// DIFF 包  
        if (qp.getType() == Leader.DIFF) {
            LOG.info("Getting a diff from the leader 0x" + Long.toHexString(qp.getZxid()));                
        }
      	// 如果是 SNAP 包，则从 leader 复制一份镜像数据到本地内存  
        else if (qp.getType() == Leader.SNAP) {
            LOG.info("Getting a snapshot from leader");
            // The leader is going to dump the database
            // db is clear as part of deserializeSnapshot()
            zk.getZKDatabase().deserializeSnapshot(leaderIs);
            String signature = leaderIs.readString("signature");
            if (!signature.equals("BenWasHere")) {
                LOG.error("Missing signature. Got " + signature);
                throw new IOException("Missing signature");                   
            }
        // TRUNC 包，回滚到对应事务
        } else if (qp.getType() == Leader.TRUNC) {
            //we need to truncate the log to the lastzxid of the leader
            LOG.warn("Truncating log to get in sync with the leader 0x"
                    + Long.toHexString(qp.getZxid()));
            boolean truncated=zk.getZKDatabase().truncateLog(qp.getZxid());
            if (!truncated) {
                // not able to truncate the log
                LOG.error("Not able to truncate the log "
                        + Long.toHexString(qp.getZxid()));
                System.exit(13);
            }

        }
        else {
            LOG.error("Got unexpected packet from leader: {}, exiting ... ",
                      LearnerHandler.packetToString(qp));
            System.exit(13);

        }
        zk.getZKDatabase().initConfigInZKDatabase(self.getQuorumVerifier());
      	// 最新的事务id  
        zk.getZKDatabase().setlastProcessedZxid(qp.getZxid());
        // 启动过期 session 检查  
        zk.createSessionTracker();            
        
        long lastQueued = 0;

        // in V1.0 we take a snapshot when we get the NEWLEADER message, but in pre V1.0
        // we take the snapshot at the UPDATE, since V1.0 also gets the UPDATE (after the NEWLEADER)
        // we need to make sure that we don't take the snapshot twice.
        boolean snapshotTaken = false;
        // we are now going to start getting transactions to apply followed by an UPTODATE
        //同步完数据后，准备执行投票 
        outerLoop:
        while (self.isRunning()) {
            readPacket(qp);
            switch(qp.getType()) {
            // 将投票添加到待处理列表
            case Leader.PROPOSAL:
                PacketInFlight pif = new PacketInFlight();
                pif.hdr = new TxnHeader();
                pif.rec = SerializeUtils.deserializeTxn(qp.getData(), pif.hdr);
                if (pif.hdr.getZxid() != lastQueued + 1) {
                LOG.warn("Got zxid 0x"
                        + Long.toHexString(pif.hdr.getZxid())
                        + " expected 0x"
                        + Long.toHexString(lastQueued + 1));
                }
                lastQueued = pif.hdr.getZxid();
                
                if (pif.hdr.getType() == OpCode.reconfig){                
                    SetDataTxn setDataTxn = (SetDataTxn) pif.rec;       
                   QuorumVerifier qv = self.configFromString(new String(setDataTxn.getData()));
                   self.setLastSeenQuorumVerifier(qv, true);                               
                }
                
                packetsNotCommitted.add(pif);
                break;
            // COMMIT 则将事务交给 Server 处理掉  
            case Leader.COMMIT:
            case Leader.COMMITANDACTIVATE:
                pif = packetsNotCommitted.peekFirst();
                if (pif.hdr.getZxid() == qp.getZxid() && qp.getType() == Leader.COMMITANDACTIVATE) {
                    QuorumVerifier qv = self.configFromString(new String(((SetDataTxn) pif.rec).getData()));
                    boolean majorChange = self.processReconfig(qv, ByteBuffer.wrap(qp.getData()).getLong(),
                            qp.getZxid(), true);
                    if (majorChange) {
                        throw new Exception("changes proposed in reconfig");
                    }
                }
                if (!snapshotTaken) {
                    if (pif.hdr.getZxid() != qp.getZxid()) {
                        LOG.warn("Committing " + qp.getZxid() + ", but next proposal is " + pif.hdr.getZxid());
                    } else {
                        zk.processTxn(pif.hdr, pif.rec);
                        packetsNotCommitted.remove();
                    }
                } else {
                    packetsCommitted.add(qp.getZxid());
                }
                break;
            case Leader.INFORM:
            case Leader.INFORMANDACTIVATE:
                PacketInFlight packet = new PacketInFlight();
                packet.hdr = new TxnHeader();

                if (qp.getType() == Leader.INFORMANDACTIVATE) {
                    ByteBuffer buffer = ByteBuffer.wrap(qp.getData());
                    long suggestedLeaderId = buffer.getLong();
                    byte[] remainingdata = new byte[buffer.remaining()];
                    buffer.get(remainingdata);
                    packet.rec = SerializeUtils.deserializeTxn(remainingdata, packet.hdr);
                    QuorumVerifier qv = self.configFromString(new String(((SetDataTxn)packet.rec).getData()));
                    boolean majorChange =
                            self.processReconfig(qv, suggestedLeaderId, qp.getZxid(), true);
                    if (majorChange) {
                        throw new Exception("changes proposed in reconfig");
                    }
                } else {
                    packet.rec = SerializeUtils.deserializeTxn(qp.getData(), packet.hdr);
                    // Log warning message if txn comes out-of-order
                    if (packet.hdr.getZxid() != lastQueued + 1) {
                        LOG.warn("Got zxid 0x"
                                + Long.toHexString(packet.hdr.getZxid())
                                + " expected 0x"
                                + Long.toHexString(lastQueued + 1));
                    }
                    lastQueued = packet.hdr.getZxid();
                }

                if (!snapshotTaken) {
                    // Apply to db directly if we haven't taken the snapshot
                    zk.processTxn(packet.hdr, packet.rec);
                } else {
                    packetsNotCommitted.add(packet);
                    packetsCommitted.add(qp.getZxid());
                }

                break;
            // UPTODATE 包，说明同步成功，退出循环 
            case Leader.UPTODATE:
                LOG.info("Learner received UPTODATE message");                                      
                if (newLeaderQV!=null) {
                   boolean majorChange =
                       self.processReconfig(newLeaderQV, null, null, true);
                   if (majorChange) {
                       throw new Exception("changes proposed in reconfig");
                   }
                }
                if (!snapshotTaken) { // true for the pre v1.0 case
                   zk.takeSnapshot();
                    self.setCurrentEpoch(newEpoch);
                }
                self.setZooKeeperServer(zk);
                self.adminServer.setZooKeeperServer(zk);
                break outerLoop;
            // NEWLEADER 包，说明之前残留的投票已经处理完了，则将内存中数据写文件，并发送ACK包
            case Leader.NEWLEADER: // it will be NEWLEADER in v1.0        
               LOG.info("Learner received NEWLEADER message");
               if (qp.getData()!=null && qp.getData().length > 1) {
                   try {                       
                       QuorumVerifier qv = self.configFromString(new String(qp.getData()));
                       self.setLastSeenQuorumVerifier(qv, true);
                       newLeaderQV = qv;
                   } catch (Exception e) {
                       e.printStackTrace();
                   }
               }
               
                zk.takeSnapshot();
                self.setCurrentEpoch(newEpoch);
                snapshotTaken = true;
                writePacket(new QuorumPacket(Leader.ACK, newLeaderZxid, null, null), true);
                break;
            }
        }
    }
    ack.setZxid(ZxidUtils.makeZxid(newEpoch, 0));
    writePacket(ack, true);
    sock.setSoTimeout(self.tickTime * self.syncLimit);
    zk.startup();
    /*
     * Update the election vote here to ensure that all members of the
     * ensemble report the same vote to new servers that start up and
     * send leader election notifications to the ensemble.
     * 
     * @see https://issues.apache.org/jira/browse/ZOOKEEPER-1732
     */
    self.updateElectionVote(newEpoch);

    // We need to log the stuff that came in between the snapshot and the uptodate
    if (zk instanceof FollowerZooKeeperServer) {
        FollowerZooKeeperServer fzk = (FollowerZooKeeperServer)zk;
        for(PacketInFlight p: packetsNotCommitted) {
            fzk.logRequest(p.hdr, p.rec);
        }
        for(Long zxid: packetsCommitted) {
            fzk.commit(zxid);
        }
    } else if (zk instanceof ObserverZooKeeperServer) {
        // Similar to follower, we need to log requests between the snapshot
        // and UPTODATE
        ObserverZooKeeperServer ozk = (ObserverZooKeeperServer) zk;
        for (PacketInFlight p : packetsNotCommitted) {
            Long zxid = packetsCommitted.peekFirst();
            if (p.hdr.getZxid() != zxid) {
                // log warning message if there is no matching commit
                // old leader send outstanding proposal to observer
                LOG.warn("Committing " + Long.toHexString(zxid)
                        + ", but next proposal is "
                        + Long.toHexString(p.hdr.getZxid()));
                continue;
            }
            packetsCommitted.remove();
            Request request = new Request(null, p.hdr.getClientId(),
                    p.hdr.getCxid(), p.hdr.getType(), null, null);
            request.setTxn(p.rec);
            request.setHdr(p.hdr);
            ozk.commitRequest(request);
        }
    } else {
        // New server type need to handle in-flight packets
        throw new UnsupportedOperationException("Unknown server type");
    }
}
```

Follower 在这里同步 leader 数据，在拿到 NEWLEADER 包之后序列化到文件，发送 ACK 包，Leader 的 LeanerHandler 线程处理

```javascript
public void waitForNewLeaderAck(long sid, long zxid, LearnerType learnerType)
        throws InterruptedException {

    synchronized (newLeaderProposal.qvAcksetPairs) {

        if (quorumFormed) {
            return;
        }

        long currentZxid = newLeaderProposal.packet.getZxid();
        if (zxid != currentZxid) {
            LOG.error("NEWLEADER ACK from sid: " + sid
                    + " is from a different epoch - current 0x"
                    + Long.toHexString(currentZxid) + " receieved 0x"
                    + Long.toHexString(zxid));
            return;
        }

        /*
         * Note that addAck already checks that the learner
         * is a PARTICIPANT.
         */
        newLeaderProposal.addAck(sid);

        if (newLeaderProposal.hasAllQuorums()) {
            quorumFormed = true;
            newLeaderProposal.qvAcksetPairs.notifyAll();
        } else {
            long start = Time.currentElapsedTime();
            long cur = start;
            long end = start + self.getInitLimit() * self.getTickTime();
            while (!quorumFormed && cur < end) {
                newLeaderProposal.qvAcksetPairs.wait(end - cur);
                cur = Time.currentElapsedTime();
            }
            if (!quorumFormed) {
                throw new InterruptedException(
                        "Timeout while waiting for NEWLEADER to be acked by quorum");
            }
        }
    }
}
```

这一部分就是等法定机器返回 ACK

由于Follower进来已经满足投票条件，则 Leader 的 Server启动，如下

```java
    /**
     * Start up Leader ZooKeeper server and initialize zxid to the new epoch
     */
    private synchronized void startZkServer() {
        // Update lastCommitted and Db's zxid to a value representing the new epoch
        lastCommitted = zk.getZxid();
        LOG.info("Have quorum of supporters, sids: [ "
                + newLeaderProposal.ackSetsToString()
                + " ]; starting up and setting last processed zxid: 0x{}",
                Long.toHexString(zk.getZxid()));
        
        /*
         * ZOOKEEPER-1324. the leader sends the new config it must complete
         *  to others inside a NEWLEADER message (see LearnerHandler where
         *  the NEWLEADER message is constructed), and once it has enough
         *  acks we must execute the following code so that it applies the
         *  config to itself.
         */
        QuorumVerifier newQV = self.getLastSeenQuorumVerifier();
        
        Long designatedLeader = getDesignatedLeader(newLeaderProposal, zk.getZxid());                                         

        self.processReconfig(newQV, designatedLeader, zk.getZxid(), true);
        if (designatedLeader != self.getId()) {
            allowedToCommit = false;
        }
        
        zk.startup();
        /*
         * Update the election vote here to ensure that all members of the
         * ensemble report the same vote to new servers that start up and
         * send leader election notifications to the ensemble.
         * 
         * @see https://issues.apache.org/jira/browse/ZOOKEEPER-1732
         */
        self.updateElectionVote(getEpoch());

        zk.getZKDatabase().setlastProcessedZxid(zk.getZxid());
    }
```

```java
public synchronized void startup() {
    if (sessionTracker == null) {
        createSessionTracker();
    }
    startSessionTracker();
  	// 设置请求处理器
    setupRequestProcessors();

    registerJMX();

    setState(State.RUNNING);
    notifyAll();
}
```

```java
@Override
protected void setupRequestProcessors() {
  	// 4.后 final 处理器
    RequestProcessor finalProcessor = new FinalRequestProcessor(this);
    RequestProcessor toBeAppliedProcessor = new Leader.ToBeAppliedRequestProcessor(finalProcessor, getLeader());
  	// 3.commit 处理器
    commitProcessor = new CommitProcessor(toBeAppliedProcessor,
            Long.toString(getServerId()), false,
            getZooKeeperServerListener());
    commitProcessor.start();
  	// 2.proposal 提案处理器
    ProposalRequestProcessor proposalProcessor = new ProposalRequestProcessor(this,
            commitProcessor);
    proposalProcessor.initialize();
  	// 1.预处理器
    prepRequestProcessor = new PrepRequestProcessor(this, proposalProcessor);
    prepRequestProcessor.start();
    firstProcessor = new LeaderRequestProcessor(this, prepRequestProcessor);

    setupContainerManager();
}
```



Leader 启动后，LeanerHandler 发送一个 UPTODATE 包

```java
/*
 * Wait until leader starts up
 */
synchronized(leader.zk){
    while(!leader.zk.isRunning() && !this.isInterrupted()){
        leader.zk.wait(20);
    }
}
// Mutation packets will be queued during the serialize,
// so we need to mark when the peer can actually start
// using the data
//
LOG.debug("Sending UPTODATE message to " + sid);      
queuedPackets.add(new QuorumPacket(Leader.UPTODATE, -1, null, null));
```

Leaner 接收到包之后处理

```java
// 退出同步数据循环
case Leader.UPTODATE:
    LOG.info("Learner received UPTODATE message");                                      
    if (newLeaderQV!=null) {
       boolean majorChange =
           self.processReconfig(newLeaderQV, null, null, true);
       if (majorChange) {
           throw new Exception("changes proposed in reconfig");
       }
    }
    if (!snapshotTaken) { // true for the pre v1.0 case
       zk.takeSnapshot();
        self.setCurrentEpoch(newEpoch);
    }
    self.setZooKeeperServer(zk);
    self.adminServer.setZooKeeperServer(zk);
    break outerLoop;

...
// 再发ACK包 
ack.setZxid(ZxidUtils.makeZxid(newEpoch, 0));
writePacket(ack, true);
sock.setSoTimeout(self.tickTime * self.syncLimit);
zk.startup();  
```

leader 的 IO线程 LearnerHandler 进入主循环，收到 ACK 包处理

```java
while (true) {
    qp = new QuorumPacket();
    ia.readRecord(qp, "packet");

    long traceMask = ZooTrace.SERVER_PACKET_TRACE_MASK;
    if (qp.getType() == Leader.PING) {
        traceMask = ZooTrace.SERVER_PING_TRACE_MASK;
    }
    if (LOG.isTraceEnabled()) {
        ZooTrace.logQuorumPacket(LOG, traceMask, 'i', qp);
    }
    tickOfNextAckDeadline = leader.self.tick.get() + leader.self.syncLimit;


    ByteBuffer bb;
    long sessionId;
    int cxid;
    int type;

    switch (qp.getType()) {
		// ACK 包，看看之前的投票是否结束  
    case Leader.ACK:
        if (this.learnerType == LearnerType.OBSERVER) {
            if (LOG.isDebugEnabled()) {
                LOG.debug("Received ACK from Observer  " + this.sid);
            }
        }
        syncLimitCheck.updateAck(qp.getZxid());
        leader.processAck(this.sid, qp.getZxid(), sock.getLocalSocketAddress());
        break;
    // PING包更新下 session 的超时时间，往前推
    case Leader.PING:
        // Process the touches
        ByteArrayInputStream bis = new ByteArrayInputStream(qp.getData());
        DataInputStream dis = new DataInputStream(bis);
        while (dis.available() > 0) {
            long sess = dis.readLong();
            int to = dis.readInt();
            leader.zk.touch(sess, to);
        }
        break;
    // REVALIDATE 包，检查 session 是否还有效
    case Leader.REVALIDATE:
        bis = new ByteArrayInputStream(qp.getData());
        dis = new DataInputStream(bis);
        long id = dis.readLong();
        int to = dis.readInt();
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        DataOutputStream dos = new DataOutputStream(bos);
        dos.writeLong(id);
        boolean valid = leader.zk.checkIfValidGlobalSession(id, to);
        if (valid) {
            try {
                //set the session owner
                // as the follower that
                // owns the session
                leader.zk.setOwner(id, this);
            } catch (SessionExpiredException e) {
                LOG.error("Somehow session " + Long.toHexString(id) + " expired right after being renewed! (impossible)", e);
            }
        }
        if (LOG.isTraceEnabled()) {
            ZooTrace.logTraceMessage(LOG,
                                     ZooTrace.SESSION_TRACE_MASK,
                                     "Session 0x" + Long.toHexString(id)
                                     + " is valid: "+ valid);
        }
        dos.writeBoolean(valid);
        qp.setData(bos.toByteArray());
        queuedPackets.add(qp);
        break;
    // REQUEST 包，事务请求，follower 会将事务请求转发给 leader 处理
    case Leader.REQUEST:
        bb = ByteBuffer.wrap(qp.getData());
        sessionId = bb.getLong();
        cxid = bb.getInt();
        type = bb.getInt();
        bb = bb.slice();
        Request si;
        if(type == OpCode.sync){
            si = new LearnerSyncRequest(this, sessionId, cxid, type, bb, qp.getAuthinfo());
        } else {
            si = new Request(null, sessionId, cxid, type, bb, qp.getAuthinfo());
        }
        si.setOwner(this);
        leader.zk.submitLearnerRequest(si);
        break;
    default:
        LOG.warn("unexpected quorum packet, type: {}", packetToString(qp));
        break;
    }
}
```

这个时候 LearnerHandler 线程已经启动完成，Follower(Leaner) 发完ACK包后

```java
ack.setZxid(ZxidUtils.makeZxid(newEpoch, 0));
writePacket(ack, true);
// 读超时为 syncLimit 时间
sock.setSoTimeout(self.tickTime * self.syncLimit);
// 启动 follower 的 zookeeper server 
zk.startup();
/*
 * Update the election vote here to ensure that all members of the
 * ensemble report the same vote to new servers that start up and
 * send leader election notifications to the ensemble.
 * 
 * @see https://issues.apache.org/jira/browse/ZOOKEEPER-1732
 */
self.updateElectionVote(newEpoch);

// We need to log the stuff that came in between the snapshot and the uptodate
if (zk instanceof FollowerZooKeeperServer) {
    FollowerZooKeeperServer fzk = (FollowerZooKeeperServer)zk;
    for(PacketInFlight p: packetsNotCommitted) {
        fzk.logRequest(p.hdr, p.rec);
    }
    for(Long zxid: packetsCommitted) {
        fzk.commit(zxid);
    }
}
```

Follower(Leaner) 的 zookeeper server 启动

**FollowerZooKeeperServer**

```java
@Override
protected void setupRequestProcessors() {
  	// final处理器
    RequestProcessor finalProcessor = new FinalRequestProcessor(this);
  	// commit处理器
    commitProcessor = new CommitProcessor(finalProcessor,
            Long.toString(getServerId()), true, getZooKeeperServerListener());
    commitProcessor.start();
  	// first处理器
    firstProcessor = new FollowerRequestProcessor(this, commitProcessor);
    ((FollowerRequestProcessor) firstProcessor).start();
  	// 独立的 syncProcessor ？？？
    syncProcessor = new SyncRequestProcessor(this,
            new SendAckRequestProcessor((Learner)getFollower()));
    syncProcessor.start();
}
```

之后 Follower 进入主处理

```java
QuorumPacket qp = new QuorumPacket();
while (this.isRunning()) {
    readPacket(qp);
    processPacket(qp);
}
```

```java
/**
 * Examine the packet received in qp and dispatch based on its contents.
 * @param qp
 * @throws IOException
 */
protected void processPacket(QuorumPacket qp) throws Exception{
    switch (qp.getType()) {
    // PING 包，写回 session 数据
    case Leader.PING:    
        ping(qp);            
        break;
    // PROPOSAL 包，投票处理
    case Leader.PROPOSAL:           
        TxnHeader hdr = new TxnHeader();
        Record txn = SerializeUtils.deserializeTxn(qp.getData(), hdr);
        if (hdr.getZxid() != lastQueued + 1) {
            LOG.warn("Got zxid 0x"
                    + Long.toHexString(hdr.getZxid())
                    + " expected 0x"
                    + Long.toHexString(lastQueued + 1));
        }
        lastQueued = hdr.getZxid();
        
        if (hdr.getType() == OpCode.reconfig){
           SetDataTxn setDataTxn = (SetDataTxn) txn;       
           QuorumVerifier qv = self.configFromString(new String(setDataTxn.getData()));
           self.setLastSeenQuorumVerifier(qv, true);                               
        }
        
        fzk.logRequest(hdr, txn);
        break;
    // COMMIT 包，提交事务    
    case Leader.COMMIT:
        fzk.commit(qp.getZxid());
        break;
        
    case Leader.COMMITANDACTIVATE:
       // get the new configuration from the request
       Request request = fzk.pendingTxns.element();
       SetDataTxn setDataTxn = (SetDataTxn) request.getTxn();                                                                                                      
       QuorumVerifier qv = self.configFromString(new String(setDataTxn.getData()));                                

       // get new designated leader from (current) leader's message
       ByteBuffer buffer = ByteBuffer.wrap(qp.getData());    
       long suggestedLeaderId = buffer.getLong();
        boolean majorChange = 
               self.processReconfig(qv, suggestedLeaderId, qp.getZxid(), true);
       // commit (writes the new config to ZK tree (/zookeeper/config)                     
       fzk.commit(qp.getZxid());
        if (majorChange) {
           throw new Exception("changes proposed in reconfig");
       }
       break;
    case Leader.UPTODATE:
        LOG.error("Received an UPTODATE message after Follower started");
        break;
    case Leader.REVALIDATE:
        revalidate(qp);
        break;
    case Leader.SYNC:
        fzk.sync();
        break;
    default:
        LOG.warn("Unknown packet type: {}", LearnerHandler.packetToString(qp));
        break;
    }
}
```

这个时候 Follower 也初始化完成，再看 Leader 主线程，Leader 主线程之前在等待 Follower 同步结束，结束之后，Leader 主线程进入主循环，检查Follower 是否 down 掉

```java
while (true) {
    synchronized (this) {
        long start = Time.currentElapsedTime();
        long cur = start;
        long end = start + self.tickTime / 2;
        while (cur < end) {
            wait(end - cur);
            cur = Time.currentElapsedTime();
        }

        if (!tickSkip) {
            self.tick.incrementAndGet();
        }

        // We use an instance of SyncedLearnerTracker to
        // track synced learners to make sure we still have a
        // quorum of current (and potentially next pending) view.
        SyncedLearnerTracker syncedAckSet = new SyncedLearnerTracker();
        syncedAckSet.addQuorumVerifier(self.getQuorumVerifier());
        if (self.getLastSeenQuorumVerifier() != null
                && self.getLastSeenQuorumVerifier().getVersion() > self
                        .getQuorumVerifier().getVersion()) {
            syncedAckSet.addQuorumVerifier(self
                    .getLastSeenQuorumVerifier());
        }

        syncedAckSet.addAck(self.getId());

        for (LearnerHandler f : getLearners()) {
            if (f.synced()) {
                syncedAckSet.addAck(f.getSid());
            }
        }

        // check leader running status
        if (!this.isRunning()) {
            shutdown("Unexpected internal error");
            return;
        }

      	// 如果有 follower 挂掉导致投票不通过，则退出 lead 流程，重新选举
        if (!tickSkip && !syncedAckSet.hasAllQuorums()) {
            // Lost quorum of last committed and/or last proposed
            // config, set shutdown flag
            shutdownMessage = "Not sufficient followers synced, only synced with sids: [ "
                    + syncedAckSet.ackSetsToString() + " ]";
            break;
        }
        tickSkip = !tickSkip;
    }
    // 检查每个 follower 是否还活着
    for (LearnerHandler f : getLearners()) {
        f.ping();
    }
}
```