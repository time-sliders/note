# 主要内容

本文主要分析 zookeeper 流程

* 新服务器加入时的流程 && 全部重新启动时的流程
* Leader 选举成功之后，Follower Diff更新流程
* Leader 宕机重新选举流程

本地环境是基于 zookeeper 单机伪集群模式。

我们从 zkServer.sh 启动脚本开始分析：

```sh
ZOOMAIN="org.apache.zookeeper.server.quorum.QuorumPeerMain"
......
case $1 in
start)
    echo  -n "Starting zookeeper ... "
    nohup "$JAVA" $ZOO_DATADIR_AUTOCREATE "-Dzookeeper.log.dir=${ZOO_LOG_DIR}" \
    "-Dzookeeper.log.file=${ZOO_LOG_FILE}" "-Dzookeeper.admin.serverPort=8088"  "-Dzookeeper.root.logger=${ZOO_LOG4J_PROP}" \
    -XX:+HeapDumpOnOutOfMemoryError -XX:OnOutOfMemoryError='kill -9 %p' \
    -cp "$CLASSPATH" $JVMFLAGS $ZOOMAIN "$ZOOCFG" > "$_ZOO_DAEMON_OUT" 2>&1 < /dev/null &
```

从启动脚本中，可以看到 zookeeper 的启动类为 `org.apache.zookeeper.server.quorum.QuorumPeerMain` 我们进入该类开始分析

```java
protected QuorumPeer quorumPeer;

/**
 * To start the replicated server specify the configuration file name on
 * the command line.
 * @param args path to the configfile
 */
public static void main(String[] args) {
    QuorumPeerMain main = new QuorumPeerMain();
    try {
        //  进入
        main.initializeAndRun(args);
    } catch (IllegalArgumentException e) {
        LOG.error("Invalid arguments, exiting abnormally", e);
        LOG.info(USAGE);
        System.err.println(USAGE);
        System.exit(2);
    } catch (ConfigException e) {
        LOG.error("Invalid config, exiting abnormally", e);
        System.err.println("Invalid config, exiting abnormally");
        System.exit(2);
    } catch (DatadirException e) {
        LOG.error("Unable to access datadir, exiting abnormally", e);
        System.err.println("Unable to access datadir, exiting abnormally");
        System.exit(3);
    } catch (AdminServerException e) {
        LOG.error("Unable to start AdminServer, exiting abnormally", e);
        System.err.println("Unable to start AdminServer, exiting abnormally");
        System.exit(4);
    } catch (Exception e) {
        LOG.error("Unexpected exception, exiting abnormally", e);
        System.exit(1);
    }
    LOG.info("Exiting normally");
    System.exit(0);
}

protected void initializeAndRun(String[] args)
    throws ConfigException, IOException, AdminServerException
{
		// 解析命令行参数  这里就是从命令行中解析诸如端口，数据文件地址之类的配置
    QuorumPeerConfig config = new QuorumPeerConfig();
    if (args.length == 1) {
        config.parse(args[0]);
    }

		// 启动一个定时清理 dataLogDir 等目录下过期文件的 Timer，非重点不细看了
    // Start and schedule the the purge task
    DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
            .getDataDir(), config.getDataLogDir(), config
            .getSnapRetainCount(), config.getPurgeInterval());
    purgeMgr.start();

    if (args.length == 1 && config.isDistributed()) {
     		// 集群模式，由于我们要看集群模式的选举与协同算法，所以从这里进去
        runFromConfig(config);
    } else {
        LOG.warn("Either no config or no quorum defined in config, running "
                + " in standalone mode");
				// 单机模式，无视
        // there is only server in the quorum -- run as standalone
        ZooKeeperServerMain.main(args);
    }
}

public void runFromConfig(QuorumPeerConfig config)
        throws IOException, AdminServerException
{
  try {
   		// 日志的一些东西，不用管
      ManagedUtil.registerLog4jMBeans();
  } catch (JMException e) {
      LOG.warn("Unable to register log4j JMX control", e);
  }

  LOG.info("Starting quorum peer");
  try {
  		// 这俩个类应该是用来管理与其他服务器连接的，暂时不用看
      ServerCnxnFactory cnxnFactory = null;
      ServerCnxnFactory secureCnxnFactory = null;

      if (config.getClientPortAddress() != null) {
          cnxnFactory = ServerCnxnFactory.createFactory();
          cnxnFactory.configure(config.getClientPortAddress(),
                  config.getMaxClientCnxns(),
                  false);
      }

      if (config.getSecureClientPortAddress() != null) {
          secureCnxnFactory = ServerCnxnFactory.createFactory();
          secureCnxnFactory.configure(config.getSecureClientPortAddress(),
                  config.getMaxClientCnxns(),
                  true);
      }

			// 核心代码   QuorumPeer 是 zk 的核心算法处理的类，这里根据启动配置，初始化对象
      quorumPeer = new QuorumPeer();
      quorumPeer.setTxnFactory(new FileTxnSnapLog(
                  config.getDataLogDir(),
                  config.getDataDir()));
      quorumPeer.enableLocalSessions(config.areLocalSessionsEnabled());
      quorumPeer.enableLocalSessionsUpgrading(
          config.isLocalSessionsUpgradingEnabled());
      //quorumPeer.setQuorumPeers(config.getAllMembers());
      quorumPeer.setElectionType(config.getElectionAlg());
      quorumPeer.setMyid(config.getServerId());  //  当前 Sever 的 Id
      quorumPeer.setTickTime(config.getTickTime());
      quorumPeer.setMinSessionTimeout(config.getMinSessionTimeout());
      quorumPeer.setMaxSessionTimeout(config.getMaxSessionTimeout());
      quorumPeer.setInitLimit(config.getInitLimit());
      quorumPeer.setSyncLimit(config.getSyncLimit());
      quorumPeer.setConfigFileName(config.getConfigFilename());
      quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory()));
      quorumPeer.setQuorumVerifier(config.getQuorumVerifier(), false);
      if (config.getLastSeenQuorumVerifier()!=null) {
          quorumPeer.setLastSeenQuorumVerifier(config.getLastSeenQuorumVerifier(), false);
      }
      quorumPeer.initConfigInZKDatabase();
      quorumPeer.setCnxnFactory(cnxnFactory);
      quorumPeer.setSecureCnxnFactory(secureCnxnFactory);
      quorumPeer.setLearnerType(config.getPeerType());
      quorumPeer.setSyncEnabled(config.getSyncEnabled());
      quorumPeer.setQuorumListenOnAllIPs(config.getQuorumListenOnAllIPs());
      
      quorumPeer.start();// 启动 进入quorumPeer 查看
      quorumPeer.join();
  } catch (InterruptedException e) {
      // warn, but generally this is ok
      LOG.warn("Quorum Peer interrupted", e);
  }
}
```

接下来，我们进入 quorumPeer 类，查看 zk 的核心逻辑

```java
@Override
public synchronized void start() {
    if (!getView().containsKey(myid)) {
        throw new RuntimeException("My id " + myid + " not in the peer list");
     }
    // 加载数据库 初始化一些数据
    loadDataBase();
  	// 
    startServerCnxnFactory();
    try {
        adminServer.start();
    } catch (AdminServerException e) {
        LOG.warn("Problem starting AdminServer", e);
        System.out.println(e);
    }
    // 这里一步里面，实际上还没有开始群首选举, 只是做了一些创建链接/开始异步消息处理线程的准备工作
    startLeaderElection();
    // 核心方法：Main Loop，我们从这一步开始分析
    super.start();
}
```

# 新服务器加入时的流程 && 全部重新启动时的流程

我们本地搭建三台zk的集群环境，先启动其中2台，由于已经达到法定数量，所以，在这已经启动的2台机器中，一台是 Leader 一台是 Follower。接下来，我们启动第三台机器，并进行 debug 跟踪，查看新机器加入时，整个集群是如何工作的。

QuorumPeer 类实际上是 zookeeper 的主线程类，上面代码最后一行，调用了 start ，即线程的 run 方法，我们从这里开始分析

```java
@Override
public void run() {
    updateThreadName(); // 设置线程名称，无视

    LOG.debug("Starting quorum peer");
    try {
        // 注册一些信息，非重点，无视
        jmxQuorumBean = new QuorumBean(this);
        MBeanRegistry.getInstance().register(jmxQuorumBean, null);
        for (QuorumServer s : getView().values()) {
            ZKMBeanInfo p;
            if (getId() == s.id) {
                p = jmxLocalPeerBean = new LocalPeerBean(this);
                try {
                    MBeanRegistry.getInstance().register(p, jmxQuorumBean);
                } catch (Exception e) {
                    LOG.warn("Failed to register with JMX", e);
                    jmxLocalPeerBean = null;
                }
            } else {
                RemotePeerBean rBean = new RemotePeerBean(s);
                try {
                    MBeanRegistry.getInstance().register(rBean, jmxQuorumBean);
                    jmxRemotePeerBean.put(s.id, rBean);
                } catch (Exception e) {
                    LOG.warn("Failed to register with JMX", e);
                }
            }
        }
    } catch (Exception e) {
        LOG.warn("Failed to register with JMX", e);
        jmxQuorumBean = null;
    }

    try {
        /*
         * Main loop  主循环，zookeeper 状态机核心代码
         */
        while (running) {
            switch (getPeerState()) { // 一开始的时候，PeerState 是 LOOKING
                case LOOKING:
                    LOG.info("LOOKING");

                    if (Boolean.getBoolean("readonlymode.enabled")) {// 我们不是 readonlyMode，这里不管
                        LOG.info("Attempting to start ReadOnlyZooKeeperServer");
                        ......
                    } else {
                        try {
                            reconfigFlagClear(); //
                            if (shuttingDownLE) {// false
                                shuttingDownLE = false;
                                startLeaderElection();
                            }
                          	
                            setCurrentVote(
                              makeLEStrategy()// 构建选举策略，我们这里是之前已经创建好的 FastLeaderElection
                              	.lookForLeader());// 核心选举代码，从这里进入
                        } catch (Exception e) {
                            LOG.warn("Unexpected exception", e);
                            // 出现异常继续重新选举
                            setPeerState(ServerState.LOOKING);
                        }
                    }
                    break;
                case OBSERVING:
                    ...
                case FOLLOWING:
                    ...
                case LEADING:
                    ...
            }
            start_fle = Time.currentElapsedTime();
        }
    } finally {
        LOG.warn("QuorumPeer main thread exited");
        MBeanRegistry instance = MBeanRegistry.getInstance();
        instance.unregister(jmxQuorumBean);
        instance.unregister(jmxLocalPeerBean);

        for (RemotePeerBean remotePeerBean : jmxRemotePeerBean.values()) {
            instance.unregister(remotePeerBean);
        }

        jmxQuorumBean = null;
        jmxLocalPeerBean = null;
        jmxRemotePeerBean = null;
    }
}
```

这里可以看到，新机器加入时，先将自己的状态设置为 LOOKING，然后调用选举算法进行选举，我们接下来看下这个选举算法到底做了什么。

```java
/**
 * Starts a new round of leader election. Whenever our QuorumPeer
 * changes its state to LOOKING, this method is invoked, and it
 * sends notifications to all other peers.
 */
public Vote lookForLeader() throws InterruptedException {
    ...
    try {
        HashMap<Long, Vote> recvset = new HashMap<Long, Vote>();
        HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>();

        int notTimeout = finalizeWait;

        synchronized(this){
						/*记录此server当前选举的轮次*/
            logicalclock.incrementAndGet();
            /*更新当前机器的提案：选自己为Leader*/
            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
        }

        LOG.info("New election. My id =  " + self.getId() +
                ", proposed zxid=0x" + Long.toHexString(proposedZxid));
        /*向所有机器发送选票：选自己为Leader，具体代码后续分析*/
        sendNotifications();

        /*
         * Loop in which we exchange notifications until we find a leader
         */
        while ((self.getPeerState() == ServerState.LOOKING) &&
                (!stop)){
            /*
             * Remove next notification from queue, times out after 2 times
             * the termination time  从接收消息队列中获取其他机器的选票信息
             */
            Notification n = recvqueue.poll(notTimeout,
                    TimeUnit.MILLISECONDS);

            /*
             * Sends more notifications if haven't received enough.
             * Otherwise processes new notification.
             * 如果没有收集到足够的选票，就重新发送 Notification，继续选举
             */
            if(n == null){
                if(manager.haveDelivered()){
                    sendNotifications();
                } else {
                    manager.connectAll();
                }

                /*
                 * Exponential backoff
                 */
                int tmpTimeOut = notTimeout*2;
                notTimeout = (tmpTimeOut < maxNotificationInterval?
                        tmpTimeOut : maxNotificationInterval);
                LOG.info("Notification time out: " + notTimeout);
            } 
            else if (self.getCurrentAndNextConfigVoters().contains(n.sid)) {
                /*
                 * Only proceed if the vote comes from a replica in the current or next
                 * voting view.  如果收到的选票，是合法的机器的选票才进行处理
                 */
                switch (n.state) {
                /*如果目标服务器也是 LOOKING 这里有2种情况，
                	1. 这个选票是当前服务器自己发送给自己的（在sendNotifications 方法内部的确把发给自己的消息也放到队列了）
                	2. 有多台机器同时在 LOOKING（很少会出现这种情况，但是还是有概率出现的，所以也要处理）*/
                case LOOKING:  
                    // If notification > current, replace and send messages out
                    // 这里请参考下面的 附录：electionEpoch / logicalclock / peerEpoch的区别
                    if (n.electionEpoch > logicalclock.get()) {
                        // 如果收到的返回结果比当前机器要大，则重新组织选票
                        logicalclock.set(n.electionEpoch);
                        recvset.clear();
                        if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                            // 如果目标服务器的选票比当前机器的选票大，则更新当前机器的选票跟目标服务器一致
                            updateProposal(n.leader, n.zxid, n.peerEpoch);
                        } else {
                            updateProposal(getInitId(),
                                    getInitLastLoggedZxid(),
                                    getPeerEpoch());
                        }
                        // 重新发送选票
                        sendNotifications();
                    } else if (n.electionEpoch < logicalclock.get()) {
                      	/* 如果目标机器的选票比我们这台机器要小，说明发送这个投票的机器丢失了一部分选举消息，
                      	可能是网络分片等原因，这里直接忽视掉这张投票就好了*/
                        if(LOG.isDebugEnabled()){
                            LOG.debug("Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x"
                                    + Long.toHexString(n.electionEpoch)
                                    + ", logicalclock=0x" + Long.toHexString(logicalclock.get()));
                        }
                        break;
                    } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                            proposedLeader, proposedZxid, proposedEpoch)) {
                      	/*走到这里说明目标机器 Epoch 与自己相同，是一张有效的投票，这里判断：
                      	如果目标选票比自己大，则更新自己的选票为目标选票，然后重新发起投票变更的通知*/
                        updateProposal(n.leader, n.zxid, n.peerEpoch);
                        sendNotifications();
                    }

                    if(LOG.isDebugEnabled()){
                        LOG.debug("Adding vote: from=" + n.sid +
                                ", proposed leader=" + n.leader +
                                ", proposed zxid=0x" + Long.toHexString(n.zxid) +
                                ", proposed election epoch=0x" + Long.toHexString(n.electionEpoch));
                    }
										// 将选票放入 recvset 中
                    recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
	
                    // 判断是否需要中断选举
                    if (termPredicate(recvset,
                            new Vote(proposedLeader, proposedZxid,
                                    logicalclock.get(), proposedEpoch))) {

                        // Verify if there is any change in the proposed leader
                        while((n = recvqueue.poll(finalizeWait,
                                TimeUnit.MILLISECONDS)) != null){
                            if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                    proposedLeader, proposedZxid, proposedEpoch)){
                                recvqueue.put(n);
                                break;
                            }
                        }

                        /*
                         * This predicate is true once we don't read any new
                         * relevant message from the reception queue
                         */
                        if (n == null) {
                            self.setPeerState((proposedLeader == self.getId()) ?
                                    ServerState.LEADING: learningState());

                            Vote endVote = new Vote(proposedLeader,
                                    proposedZxid, proposedEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }
                    break;
                case OBSERVING:
                    LOG.debug("Notification from observer: " + n.sid);
                    break;
                case FOLLOWING:
                case LEADING:
                    /* 如果收到的选票是  FOLLOWING || LEADING 表示目标机器已经完成了选举
                    绝大部分 zookeeper 启动加入集群时，大部分走的都是这一部分的代码*/
                    /*
                     * Consider all notifications from the same epoch
                     * together.
                     */
                    if(n.electionEpoch == logicalclock.get()){ 
                      /*目标机器也正在开始选举，这是一个极端场景，目标机器从 FOLLOWING || LEADING 
                      刚刚进入选举状态，可能是由于分片等原因*/
                        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
                        if(termPredicate(recvset, new Vote(n.leader,
                                        n.zxid, n.electionEpoch, n.peerEpoch, n.state))
                                        && checkLeader(outofelection, n.leader, n.electionEpoch)) {
                            self.setPeerState((n.leader == self.getId()) ?
                                    ServerState.LEADING: learningState());

                            Vote endVote = new Vote(n.leader, n.zxid, n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }

                    /*
                     * Before joining an established ensemble整体, verify that
                     * a majority are following the same leader.
                     * Only peer epoch is used to check that the votes come
                     * from the same ensemble. This is because there is at
                     * least one corner case in which the ensemble can be
                     * created with inconsistent zxid and election epoch
                     * info. However, given that only one ensemble can be
                     * running at a single point in time and that each 
                     * epoch is used only once, using only the epoch to 
                     * compare the votes is sufficient.
                     * 
                     * @see https://issues.apache.org/jira/browse/ZOOKEEPER-1732
                     */
                    // 收集目标已经完成选举的机器的选票到 outofelection 集合
                    outofelection.put(n.sid, new Vote(n.leader, 
                            IGNOREVALUE, IGNOREVALUE, n.peerEpoch, n.state));
                    /* 判断是否已经结束选举  这里是根据已经响应的数量是否大于 1/2  ，代码见下面分析
                    	 这里直接判断的是outofelection集合，但是实际上，可能有一部分机器是处理LOOKING状态
                       他们的选票是保存在recvset中，但是没关系，这里只要半数以上机器已经投了相同的 Vote，
                       则表示集群已经选举结束，其他机器直接 Follower 就好了*/
                    if (termPredicate(outofelection, new Vote(n.leader,
                            IGNOREVALUE, IGNOREVALUE, n.peerEpoch, n.state))
                        		/*有一个问题是，由于Leader和Follower有各自的超时机制，Leader如果在一段时间内未收
                        		到大多数Follower的响应，就会认为它已经失去了集群的支持，就会进入LOOKING状态，而Follower
                        		如果在一段时间内未收到来自Leader的PING命令，会认为它已经和Leader失去了响应，从而进
                        		入LOOKING状态，重新查找Leader．
                        		但是，这个时间是物理时间，它在各个机器上并不是严格相等的，中间可能存在一定的误差，即使
                        		我们知道只要误差小于一定的值，基本上就可以达到同步，但是Zab的实现中可不是这么做的．这就
                        		会导致可能Leader已经超时，进入了LOOKING状态，而Follower只能过一段时间才能检测到这一
                        		事件，并且也进入LOOKING状态．
                        		如果此时有一台Server想要加入集群，此时Follower都会告诉这台Server此时的leader时那台
                        		已经宕机的Server．那么如何解决这个问题呢？所以，在Zab的实现中，有这么一段代码，用来检
                        		测Follower告诉这台Server此时的leader是否给这台Server发送过消息，以及Follower认为
                        		的leader此时是否确实处于LEADING状态*/
                            && checkLeader(outofelection, n.leader, IGNOREVALUE)) {
                        synchronized(this){
                            logicalclock.set(n.electionEpoch);// 重置时钟
                          	// 设置当前机器的状态，下一次循环就会进入 FOLLOWER 的逻辑了
                            self.setPeerState((n.leader == self.getId()) ?
                                    ServerState.LEADING: learningState());
                        }
                        Vote endVote = new Vote(n.leader, n.zxid, n.peerEpoch);
                        leaveInstance(endVote);// 退出选举
                        return endVote;
                    }
                    break;
                default:
                    LOG.warn("Notification state unrecoginized: " + n.state
                          + " (n.state), " + n.sid + " (n.sid)");
                    break;
                }
            } else {
                LOG.warn("Ignoring notification from non-cluster member " + n.sid);
            }
        }
        return null;
    } finally {
        try {
            if(self.jmxLeaderElectionBean != null){
                MBeanRegistry.getInstance().unregister(
                        self.jmxLeaderElectionBean);
            }
        } catch (Exception e) {
            LOG.warn("Failed to unregister with JMX", e);
        }
        self.jmxLeaderElectionBean = null;
    }
}
```

```java
/**
 * Send notifications to all peers upon a change in our vote
 */
private void sendNotifications() {
    /*遍历所有的投票者*/
    for (long sid : self.getCurrentAndNextConfigVoters()) {
        /*组装各个投票者的通知消息*/
        QuorumVerifier qv = self.getQuorumVerifier();
        ToSend notmsg = new ToSend(ToSend.mType.notification,
                proposedLeader,
                proposedZxid,
                logicalclock.get(),
                QuorumPeer.ServerState.LOOKING,
                sid,
                proposedEpoch, qv.toString().getBytes());
        if(LOG.isDebugEnabled()){
            LOG.debug("Sending Notification: " + proposedLeader + " (n.leader), 0x"  +
                  Long.toHexString(proposedZxid) + " (n.zxid), 0x" + Long.toHexString(logicalclock.get())  +
                  " (n.round), " + sid + " (recipient), " + self.getId() +
                  " (myid), 0x" + Long.toHexString(proposedEpoch) + " (n.peerEpoch)");
        }
        // 将消息放到 sendqueue
        sendqueue.offer(notmsg);
    }
}
```

该方法表示向其他所有机器发送通知，通知的内容是：选当前机器为Leader

在执行时的日志如下

> [0614 11:28:45 696 INFO ] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.QuorumPeer - **LOOKING**
> [0614 11:28:45 697 DEBUG] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.QuorumPeer - **Initializing leader election protocol...**
> [0614 11:28:45 699 DEBUG] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.FastLeaderElection - **Updating proposal: 1 (newleader), 0x200000004 (newzxid), -1 (oldleader), 0xffffffffffffffff (oldzxid)**
> [0614 11:28:45 699 INFO ] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.FastLeaderElection - **New election. My id =  1, proposed zxid=0x200000004**
> [0614 11:28:45 700 DEBUG] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.FastLeaderElection - **Sending Notification: 1 (n.leader), 0x200000004 (n.zxid), 0x1 (n.round), 1 (recipient), 1 (myid), 0x3 (n.peerEpoch)**
> [0614 11:28:45 701 DEBUG] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.FastLeaderElection - **Sending Notification: 1 (n.leader), 0x200000004 (n.zxid), 0x1 (n.round), 2 (recipient), 1 (myid), 0x3 (n.peerEpoch)**
> [0614 11:28:45 701 DEBUG] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.FastLeaderElection - **Sending Notification: 1 (n.leader), 0x200000004 (n.zxid), 0x1 (n.round), 3 (recipient), 1 (myid), 0x3 (n.peerEpoch)**

全序断言，下面这个方法用来比较2张选票的大小

```java
/**
 * Check if a pair (server id, zxid) succeeds our
 * current vote.
 *
 * @param id    Server identifier
 * @param zxid  Last zxid observed by the issuer of this vote
 */
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
    LOG.debug("id: " + newId + ", proposed id: " + curId + ", zxid: 0x" +
            Long.toHexString(newZxid) + ", proposed zxid: 0x" + Long.toHexString(curZxid));
    if(self.getQuorumVerifier().getWeight(newId) == 0){
        return false;
    }

    /*
     * We return true if one of the following three cases hold:
     * 1- New epoch is higher
     * 2- New epoch is the same as current epoch, but new zxid is higher
     * 3- New epoch is the same as current epoch, new zxid is the same
     *  as current zxid, but server id is higher.
     * 这里可以看到比较顺序:  epoch > zxid > sid
     */
    return ((newEpoch > curEpoch) ||
            ((newEpoch == curEpoch) &&
            ((newZxid > curZxid) || ((newZxid == curZxid) && (newId > curId)))));
}
```

termPredicate 判断是否已经选举结束

```java
/**
 * Termination predicate. Given a set of votes, determines if have
 * sufficient to declare the end of the election round.
 * 
 * @param votes
 *            Set of votes
 * @param vote
 *            Identifier of the vote received last
 */
private boolean termPredicate(HashMap<Long, Vote> votes, Vote vote) {
    SyncedLearnerTracker voteSet = new SyncedLearnerTracker();
    // 第一步，添加 QuorumVerifier，该类里面包含集群总量，也就是我们3台机器信息  
    voteSet.addQuorumVerifier(self.getQuorumVerifier());
    if (self.getLastSeenQuorumVerifier() != null
            && self.getLastSeenQuorumVerifier().getVersion() > self
                    .getQuorumVerifier().getVersion()) {
        voteSet.addQuorumVerifier(self.getLastSeenQuorumVerifier());
    }

    /*
     * First make the views consistent. Sometimes peers will have different
     * zxids for a server depending on timing.
     * 2. 将已经响应的机器放入 QuorumVerifier 的 Ack 列表里面
     */
    for (Map.Entry<Long, Vote> entry : votes.entrySet()) {
        if (vote.equals(entry.getValue())) {
            voteSet.addAck(entry.getKey());
        }
    }
		
  	/* 判断 ack 列表是否大于 half：1，即至少要2台机器响应*/
    return voteSet.hasAllQuorums();
}
```

SyncedLearnerTracker

```java
public boolean hasAllQuorums() {
    for (QuorumVerifierAcksetPair qvAckset : qvAcksetPairs) {
        if (!qvAckset.getQuorumVerifier().containsQuorum(qvAckset.getAckset()))
            return false;
    }
    return true;
}
```

QuorumMaj

```java
/**
 * Verifies if a set is a majority. Assumes that ackSet contains acks only
 * from votingMembers
 */
public boolean containsQuorum(Set<Long> ackSet) { 
    return (ackSet.size() > half); // if 3：half = 1 这里可以看到至少大于 1/2即，3台中至少要有2台
}
```

QuorumCnxManager 会与其他服务器交互，将 Notification 发送过去，之后各个服务器返回响应。**这里有一点很奇怪，各个服务器，分别返回了2次Notification，需要确认原因！！**

> [0614 11:28:45 702 DEBUG] [WorkerSender[myid=1]] server.quorum.QuorumPeer - Resolved address for 127.0.0.1: /127.0.0.1
> [0614 11:28:45 702 DEBUG] [WorkerSender[myid=1]] server.quorum.QuorumPeer - Resolved address for 127.0.0.1: /127.0.0.1
> [0614 11:28:45 702 DEBUG] [WorkerSender[myid=1]] server.quorum.QuorumCnxManager - Opening channel to server 2
> [0614 11:28:45 703 DEBUG] [WorkerSender[myid=1]] server.quorum.QuorumCnxManager - Connected to server 2
> [0614 11:28:45 705 INFO ] [WorkerSender[myid=1]] server.quorum.QuorumCnxManager - Have smaller server identifier, so dropping the connection: (2, 1)
> [0614 11:28:45 706 INFO ] [/127.0.0.1:3001] server.quorum.QuorumCnxManager - Received connection request /127.0.0.1:53595
> [0614 11:28:45 707 DEBUG] [WorkerSender[myid=1]] server.quorum.QuorumPeer - Resolved address for 127.0.0.1: /127.0.0.1
> [0614 11:28:45 707 DEBUG] [WorkerReceiver[myid=1]] server.quorum.FastLeaderElection - **Receive new notification message. My id = 1**
> [0614 11:28:45 707 DEBUG] [WorkerSender[myid=1]] server.quorum.QuorumPeer - Resolved address for 127.0.0.1: /127.0.0.1
> [0614 11:28:45 707 DEBUG] [WorkerSender[myid=1]] server.quorum.QuorumCnxManager - Opening channel to server 3
> [0614 11:28:45 708 DEBUG] [WorkerSender[myid=1]] server.quorum.QuorumCnxManager - Connected to server 3
> [0614 11:28:45 708 INFO ] [WorkerSender[myid=1]] server.quorum.QuorumCnxManager - Have smaller server identifier, so dropping the connection: (3, 1)
> [0614 11:28:45 708 INFO ] [WorkerReceiver[myid=1]] server.quorum.FastLeaderElection - **Notification: 2 (message format version), 1 (n.leader), 0x200000004 (n.zxid), 0x1 (n.round), LOOKING (n.state), 1 (n.sid), 0x3 (n.peerEPoch), LOOKING (my state)100000000 (n.config version)**
> [0614 11:28:45 708 DEBUG] [/127.0.0.1:3001] server.quorum.QuorumCnxManager - Address of remote peer: 2
> [0614 11:28:45 709 DEBUG] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.FastLeaderElection - id: 1, proposed id: 1, zxid: 0x200000004, proposed zxid: 0x200000004
> [0614 11:28:45 709 DEBUG] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.FastLeaderElection - Adding vote: from=1, proposed leader=1, proposed zxid=0x200000004, proposed election epoch=0x1
> [0614 11:28:45 710 INFO ] [/127.0.0.1:3001] server.quorum.QuorumCnxManager - Received connection request /127.0.0.1:53597
> [0614 11:28:45 710 DEBUG] [/127.0.0.1:3001] server.quorum.QuorumCnxManager - Address of remote peer: 3
> [0614 11:28:45 711 DEBUG] [WorkerReceiver[myid=1]] server.quorum.FastLeaderElection - **Receive new notification message. My id = 1**
> [0614 11:28:45 711 INFO ] [WorkerReceiver[myid=1]] server.quorum.FastLeaderElection - **Notification: 2 (message format version), 3 (n.leader), 0x200000004 (n.zxid), 0xffffffffffffffff (n.round), FOLLOWING (n.state), 2 (n.sid), 0x3 (n.peerEPoch), LOOKING (my state)100000000 (n.config version)**
> [0614 11:28:45 720 DEBUG] [WorkerReceiver[myid=1]] server.quorum.FastLeaderElection - **Receive new notification message. My id = 1**
> [0614 11:28:45 720 INFO ] [WorkerReceiver[myid=1]] server.quorum.FastLeaderElection - **Notification: 2 (message format version), 3 (n.leader), 0x200000004 (n.zxid), 0xffffffffffffffff (n.round), FOLLOWING (n.state), 2 (n.sid), 0x3 (n.peerEPoch), LOOKING (my state)100000000 (n.config version)**
> [0614 11:28:45 721 DEBUG] [WorkerReceiver[myid=1]] server.quorum.FastLeaderElection - **Receive new notification message. My id = 1**
> [0614 11:28:45 721 INFO ] [WorkerReceiver[myid=1]] server.quorum.FastLeaderElection - **Notification: 2 (message format version), 3 (n.leader), 0x200000004 (n.zxid), 0xffffffffffffffff (n.round), LEADING (n.state), 3 (n.sid), 0x3 (n.peerEPoch), LOOKING (my state)100000000 (n.config version)**
> [0614 11:28:45 721 DEBUG] [WorkerReceiver[myid=1]] server.quorum.FastLeaderElection - **Receive new notification message. My id = 1**
> [0614 11:28:45 721 DEBUG] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.FastLeaderElection - I'm a participant: 1
> [0614 11:28:45 721 INFO ] [WorkerReceiver[myid=1]] server.quorum.FastLeaderElection - **Notification: 2 (message format version), 3 (n.leader), 0x200000004 (n.zxid), 0xffffffffffffffff (n.round), LEADING (n.state), 3 (n.sid), 0x3 (n.peerEPoch), LOOKING (my state)100000000 (n.config version)**

这部分日志可以看到，其他2台服务器分别返回了各自的状态，之后当前服务器根据其他服务器的状态，确定已经存在 Leader 了，所以设置自己的状态为 Follower，然后发送 diff 更新的请求给 Leader，更新本地数据。

> [0614 11:28:45 721 DEBUG] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.FastLeaderElection - **About to leave FLE instance: leader=3, zxid=0x200000004, my id=1, my state=FOLLOWING**
> [0614 11:28:45 721 INFO ] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] zookeeper.jmx.MBeanRegistry - Unregister MBean [org.apache.ZooKeeperService:name0=ReplicatedServer_id1,name1=replica.1,name2=LeaderElection]
> [0614 11:28:45 722 INFO ] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.QuorumPeer - **FOLLOWING**
[0614 11:28:45 764 INFO ] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.Learner - FOLLOWING - LEADER ELECTION TOOK - 42 MS
[0614 11:28:45 783 INFO ] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.Learner - **Getting a diff from the leader 0x200000004**
[0614 11:28:45 799 INFO ] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.Learner - Learner received NEWLEADER message
[0614 11:28:45 800 INFO ] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.persistence.FileTxnSnapLog - Snapshotting: 0x200000004 to /data/www/zk_cluster/server1/version-2/snapshot.200000004
[0614 11:28:45 811 INFO ] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.Learner - **Learner received UPTODATE message**
[0614 11:28:45 812 DEBUG] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.QuorumPeer - 1 setQuorumVerifier called with known or old config 4294967296. Current version: 4294967296
[0614 11:28:45 832 INFO ] [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:2181)(secure=disabled)] server.quorum.CommitProcessor - Configuring CommitProcessor with 4 worker threads.


# 附录

## electionEpoch / logicalclock / peerEpoch的区别

**electionEpoch **和 **logicalclock** 的区别在于，**electionEpoch **指的是发出 Notification 的server 的 **logicalclock**，而 **logicalclock** 则指的是当前 Server 所处的选举的轮次，每次调用 **lookForLeader()** 方法，它的值都会加一． 而对于一个新加入的机器来说，logicalclock 初始化为 0，如果加入成功，则会被设置为跟集群的机器保持一致。**electionEpoch** 和 **peerEpoch** 的区别在于，**electionEpoch** 记录的选举的轮次，而 **peerEpoch** 则指的是当前leader 的任期。