leader的实现类为LeaderZooKeeperServer，它间接继承自标准ZookeeperServer。它规定了请求到达leader时需要经历的路径：
PrepRequestProcessor -> ProposalRequestProcessor ->CommitProcessor -> Leader.ToBeAppliedRequestProcessor ->*FinalRequestProcessor*
该处理器链在 上一篇 Leader 初始化阶段进行设置：

```java
@Override
protected void setupRequestProcessors() {
    RequestProcessor finalProcessor = new FinalRequestProcessor(this);
    RequestProcessor toBeAppliedProcessor = new Leader.ToBeAppliedRequestProcessor(finalProcessor, getLeader());
    commitProcessor = new CommitProcessor(toBeAppliedProcessor,
            Long.toString(getServerId()), false,
            getZooKeeperServerListener());
    commitProcessor.start();
    ProposalRequestProcessor proposalProcessor = new ProposalRequestProcessor(this,
            commitProcessor);
    proposalProcessor.initialize();
    prepRequestProcessor = new PrepRequestProcessor(this, proposalProcessor);
    prepRequestProcessor.start();
    firstProcessor = new LeaderRequestProcessor(this, prepRequestProcessor);

    setupContainerManager();
}
```



# LeaderRequestProcessor

第一个消息服务器处理器预消息服务器 LeaderRequestProcessor

只有当 request 直接提交到 leader 的时候，才会被该处理器处理

```java
@Override
public void processRequest(Request request)
        throws RequestProcessorException {
    // Check if this is a local session and we are trying to create
    // an ephemeral node, in which case we upgrade the session
    Request upgradeRequest = null;
    try {
        upgradeRequest = lzks.checkUpgradeSession(request);
    } catch (KeeperException ke) {
        if (request.getHdr() != null) {
            LOG.debug("Updating header");
            request.getHdr().setType(OpCode.error);
            request.setTxn(new ErrorTxn(ke.code().intValue()));
        }
        request.setException(ke);
        LOG.info("Error creating upgrade request " + ke.getMessage());
    } catch (IOException ie) {
        LOG.error("Unexpected error in upgrade", ie);
    }
  
    if (upgradeRequest != null) {
      	// 提交一个升级session 的请求处理，这里是更新 session 的超时信息，非重点，不细看了
        nextProcessor.processRequest(upgradeRequest);
    }
		
  	// 提交 request 给下一个 processor 处理
    nextProcessor.processRequest(request);
}
```

# PrepRequestProcessor

下一个 processor 是 PrepRequestProcessor，这个处理器是一个异步线程在处理提交到队列里面的信息

```java
/**
 * This method will be called inside the ProcessRequestThread, which is a
 * singleton, so there will be a single thread calling this code.
 *
 * @param type
 * @param zxid
 * @param request
 * @param record
 */
protected void pRequest2Txn(int type, long zxid, Request request,
                            Record record, boolean deserialize)
    throws KeeperException, IOException, RequestProcessorException
{
    request.setHdr(new TxnHeader(request.sessionId, request.cxid, zxid,
            Time.currentWallTime(), type));

    switch (type) {
        case OpCode.create:
        case OpCode.create2:
        case OpCode.createTTL:
        case OpCode.createContainer: {
            pRequest2TxnCreate(type, request, record, deserialize);
            break;
        }
				...
            ...
        default:
            LOG.warn("unknown type " + type);
            break;
    }
}
```

该方法是处理不同类型的request，根据type选择一个处理分支，ProcessRequestThread内部调用该方法，它是单例的，因此只有一个单线程调用此代码。以create请求为例，了解工作机制：

```java
private void pRequest2TxnCreate(int type, Request request, Record record, boolean deserialize) throws IOException, KeeperException {
    if (deserialize) {
        ByteBufferInputStream.byteBuffer2Record(request.request, record);
    }

    int flags;
    String path;
    List<ACL> acl;
    byte[] data;
    long ttl;
    if (type == OpCode.createTTL) {
        CreateTTLRequest createTtlRequest = (CreateTTLRequest)record;
        flags = createTtlRequest.getFlags();
        path = createTtlRequest.getPath();
        acl = createTtlRequest.getAcl();
        data = createTtlRequest.getData();
        ttl = createTtlRequest.getTtl();
    } else {
        CreateRequest createRequest = (CreateRequest)record;
        flags = createRequest.getFlags();
        path = createRequest.getPath();
        acl = createRequest.getAcl();
        data = createRequest.getData();
        ttl = 0;
    }
    CreateMode createMode = CreateMode.fromFlag(flags);
  	// 验证 session 是否超时/合法之类的
    validateCreateRequest(createMode, request);
  	// 验证 需要创建的 path 的路径是否合法，然后返回 parrentPath
    String parentPath = validatePathForCreate(path, request.sessionId);

    List<ACL> listACL = fixupACL(path, request.authInfo, acl);
    ChangeRecord parentRecord = getRecordForPath(parentPath);// 给这个path创建一个ChangeRecord
  	// acl 验证，无视
    checkACL(zks, request.cnxn, parentRecord.acl, ZooDefs.Perms.CREATE, request.authInfo, path, listACL);

    int parentCVersion = parentRecord.stat.getCversion();
    if (createMode.isSequential()) {
        path = path + String.format(Locale.ENGLISH, "%010d", parentCVersion);
    }
    validatePath(path, request.sessionId);// 验证路径合法性
    try {
        if (getRecordForPath(path) != null) {
            throw new KeeperException.NodeExistsException(path);
        }
    } catch (KeeperException.NoNodeException e) {
        // ignore this one
    }
    boolean ephemeralParent = EphemeralType.get(parentRecord.stat.getEphemeralOwner()) == EphemeralType.NORMAL;
    if (ephemeralParent) {
        throw new KeeperException.NoChildrenForEphemeralsException(path);
    }
    int newCversion = parentRecord.stat.getCversion()+1;
    if (type == OpCode.createContainer) {
        request.setTxn(new CreateContainerTxn(path, data, listACL, newCversion));
    } else if (type == OpCode.createTTL) {
        request.setTxn(new CreateTTLTxn(path, data, listACL, newCversion, ttl));
    } else {
        request.setTxn(new CreateTxn(path, data, listACL, createMode.isEphemeral(),
                newCversion));
    }
    StatPersisted s = new StatPersisted();
    if (createMode.isEphemeral()) {
        s.setEphemeralOwner(request.sessionId);
    }
    parentRecord = parentRecord.duplicate(request.getHdr().getZxid());
    parentRecord.childCount++;
    parentRecord.stat.setCversion(newCversion);
  	// 这里
    addChangeRecord(parentRecord);
    addChangeRecord(new ChangeRecord(request.getHdr().getZxid(), path, s, 0, listACL));
}
```

调用方法，处理变化

```java
private void addChangeRecord(ChangeRecord c) {
    synchronized (zks.outstandingChanges) {
        zks.outstandingChanges.add(c);
        zks.outstandingChangesForPath.put(c.path, c);
    }
}
```

其中：outstandingChanges 是一组 ChangeRecord，outstandingChangesForPath 是 map 的 ChangeRecord，如下定义：

```java
final List<ChangeRecord> outstandingChanges = new ArrayList<ChangeRecord>();
// this data structure must be accessed under the outstandingChanges lock
final HashMap<String, ChangeRecord> outstandingChangesForPath =
new HashMap<String, ChangeRecord>();
```

ChangeRecord 是一个数据结构，方便 PrepRP 和 FinalRp 共享信息

```java
/**
 * This structure is used to facilitate information sharing between PrepRP
 * and FinalRP.
 */
static class ChangeRecord {
    ChangeRecord(long zxid, String path, StatPersisted stat, int childCount,
            List<ACL> acl) {
        this.zxid = zxid;
        this.path = path;
        this.stat = stat;
        this.childCount = childCount;
        this.acl = acl;
    }

    long zxid;

    String path;

    StatPersisted stat; /* Make sure to create a new object when changing */

    int childCount;

    List<ACL> acl; /* Make sure to create a new object when changing */

    ChangeRecord duplicate(long zxid) {
        StatPersisted stat = new StatPersisted();
        if (this.stat != null) {
            DataTree.copyStatPersisted(this.stat, stat);
        }
        return new ChangeRecord(zxid, path, stat, childCount,
                acl == null ? new ArrayList<ACL>() : new ArrayList<ACL>(acl));
    }
}
```

#  ProposalRequestProcessor

This RequestProcessor simply forwards requests to an AckRequestProcessor and SyncRequestProcessor. 该处理器只是简单的将请求转发给 SyncRequestProcessor 和 AckRequestProcessor


```java
public void processRequest(Request request) throws RequestProcessorException {
    // LOG.warn("Ack>>> cxid = " + request.cxid + " type = " +
    // request.type + " id = " + request.sessionId);
    // request.addRQRec(">prop");


    /* In the following IF-THEN-ELSE block, we process syncs on the leader.
     * If the sync is coming from a follower, then the follower
     * handler adds it to syncHandler. Otherwise, if it is a client of
     * the leader that issued the sync command, then syncHandler won't
     * contain the handler. In this case, we add it to syncHandler, and
     * call processRequest on the next processor.
     */

    if (request instanceof LearnerSyncRequest){
        zks.getLeader().processSync((LearnerSyncRequest)request);
    } else {
        nextProcessor.processRequest(request);
        if (request.getHdr() != null) {
            // We need to sync and get consensus on any transactions
            try {
                zks.getLeader().propose(request);
            } catch (XidRolloverException e) {
                throw new RequestProcessorException(e.getMessage(), e);
            }
            syncProcessor.processRequest(request);
        }
    }
}
```

## SyncRequestProcessor

```java
@Override
public void run() {
    try {
        int logCount = 0;

        // we do this in an attempt to ensure that not all of the servers
        // in the ensemble take a snapshot at the same time
        int randRoll = r.nextInt(snapCount/2);
        while (true) {
            Request si = null;
            if (toFlush.isEmpty()) {
                si = queuedRequests.take();
            } else {
                si = queuedRequests.poll();
                if (si == null) {
                    flush(toFlush);
                    continue;
                }
            }
            if (si == requestOfDeath) {
                break;
            }
            if (si != null) {
                // track the number of records written to the log
                if (zks.getZKDatabase().append(si)) {
                    logCount++;
                    if (logCount > (snapCount / 2 + randRoll)) {
                        randRoll = r.nextInt(snapCount/2);
                        // roll the log
                        zks.getZKDatabase().rollLog();
                        // take a snapshot
                        if (snapInProcess != null && snapInProcess.isAlive()) {
                            LOG.warn("Too busy to snap, skipping");
                        } else {
                          	// 异步处理日志和快照，启动ZooKeeperThread线程来生成快照。
                            snapInProcess = new ZooKeeperThread("Snapshot Thread") {
                                    public void run() {
                                        try {
                                            zks.takeSnapshot();
                                        } catch(Exception e) {
                                            LOG.warn("Unexpected exception", e);
                                        }
                                    }
                                };
                            snapInProcess.start();
                        }
                        logCount = 0;
                    }
                } else if (toFlush.isEmpty()) {
                    // optimization for read heavy workloads
                    // iff this is a read, and there are no pending
                    // flushes (writes), then just pass this to the next
                    // processor
                    if (nextProcessor != null) {
                        nextProcessor.processRequest(si);
                        if (nextProcessor instanceof Flushable) {
                            ((Flushable)nextProcessor).flush();
                        }
                    }
                    continue;
                }
                toFlush.add(si);
                if (toFlush.size() > 1000) {
                    flush(toFlush);
                }
            }
        }
    } catch (Throwable t) {
        handleException(this.getName(), t);
    } finally{
        running = false;
    }
    LOG.info("SyncRequestProcessor exited!");
}
```

异步处理日志和快照，启动ZooKeeperThread线程来生成快照。

```java
public void takeSnapshot(){
    try {
        txnLogFactory.save(zkDb.getDataTree(), zkDb.getSessionWithTimeOuts());
    } catch (IOException e) {
        LOG.error("Severe unrecoverable error, exiting", e);
        // This is a severe error that we cannot recover from,
        // so we need to exit
        System.exit(10);
    }
}
```

FileTxnSnapLog是个工具类，帮助处理txtlog和snapshot。

```java
/**
 * save the datatree and the sessions into a snapshot
 * @param dataTree the datatree to be serialized onto disk
 * @param sessionsWithTimeouts the sesssion timeouts to be
 * serialized onto disk
 * @throws IOException
 */
public void save(DataTree dataTree,
        ConcurrentHashMap<Long, Integer> sessionsWithTimeouts)
    throws IOException {
    long lastZxid = dataTree.lastProcessedZxid;
    File snapshotFile = new File(snapDir, Util.makeSnapshotName(lastZxid));
    LOG.info("Snapshotting: 0x{} to {}", Long.toHexString(lastZxid),
            snapshotFile);
    snapLog.serialize(dataTree, sessionsWithTimeouts, snapshotFile);

}
```

持久化为文件

```java
/**
 * serialize the datatree and session into the file snapshot
 * @param dt the datatree to be serialized
 * @param sessions the sessions to be serialized
 * @param snapShot the file to store snapshot into
 */
public synchronized void serialize(DataTree dt, Map<Long, Integer> sessions, File snapShot)
        throws IOException {
    if (!close) {
        OutputStream sessOS = new BufferedOutputStream(new FileOutputStream(snapShot));
        CheckedOutputStream crcOut = new CheckedOutputStream(sessOS, new Adler32());
        //CheckedOutputStream cout = new CheckedOutputStream()
        OutputArchive oa = BinaryOutputArchive.getArchive(crcOut);
        FileHeader header = new FileHeader(SNAP_MAGIC, VERSION, dbId);
        serialize(dt,sessions,oa, header);
        long val = crcOut.getChecksum().getValue();
        oa.writeLong(val, "val");
        oa.writeString("/", "path");
        sessOS.flush();
        crcOut.close();
        sessOS.close();
    }
}
```

## AckRequestProcessor

AckRequestProcessor 是 syncRequestProcessor 的下一个 Processor 仅仅将发送过来的请求作为ACk转发给leader。代码见明细：

```java
public void processRequest(Request request) {
    QuorumPeer self = leader.self;
    if(self != null)
        leader.processAck(self.getId(), request.zxid, null);
    else
        LOG.error("Null QuorumPeer");
}
```

leader处理请求如下所示

```java
/**
 * Keep a count of acks that are received by the leader for a particular
 * proposal
 *
 * @param zxid, the zxid of the proposal sent out
 * @param sid, the id of the server that sent the ack
 * @param followerAddr
 */
synchronized public void processAck(long sid, long zxid, SocketAddress followerAddr) {        
    if (!allowedToCommit) return; // last op committed was a leader change - from now on 
                                 // the new leader should commit        
    if (LOG.isTraceEnabled()) {
        LOG.trace("Ack zxid: 0x{}", Long.toHexString(zxid));
        for (Proposal p : outstandingProposals.values()) {
            long packetZxid = p.packet.getZxid();
            LOG.trace("outstanding proposal: 0x{}",
                    Long.toHexString(packetZxid));
        }
        LOG.trace("outstanding proposals all");
    }
    
    if ((zxid & 0xffffffffL) == 0) {
        /*
         * We no longer process NEWLEADER ack with this method. However,
         * the learner sends an ack back to the leader after it gets
         * UPTODATE, so we just ignore the message.
         */
        return;
    }
        
        
    if (outstandingProposals.size() == 0) {
        if (LOG.isDebugEnabled()) {
            LOG.debug("outstanding is 0");
        }
        return;
    }
    if (lastCommitted >= zxid) {
        if (LOG.isDebugEnabled()) {
            LOG.debug("proposal has already been committed, pzxid: 0x{} zxid: 0x{}",
                    Long.toHexString(lastCommitted), Long.toHexString(zxid));
        }
        // The proposal has already been committed
        return;
    }
    Proposal p = outstandingProposals.get(zxid);
    if (p == null) {
        LOG.warn("Trying to commit future proposal: zxid 0x{} from {}",
                Long.toHexString(zxid), followerAddr);
        return;
    }
    
    p.addAck(sid);        
    /*if (LOG.isDebugEnabled()) {
        LOG.debug("Count for zxid: 0x{} is {}",
                Long.toHexString(zxid), p.ackSet.size());
    }*/
    
  	// 尝试提交
    boolean hasCommitted = tryToCommit(p, zxid, followerAddr);

    // If p is a reconfiguration, multiple other operations may be ready to be committed,
    // since operations wait for different sets of acks.
   // Currently we only permit one outstanding reconfiguration at a time
   // such that the reconfiguration and subsequent outstanding ops proposed while the reconfig is
   // pending all wait for a quorum of old and new config, so its not possible to get enough acks
   // for an operation without getting enough acks for preceding ops. But in the future if multiple
   // concurrent reconfigs are allowed, this can happen and then we need to check whether some pending
    // ops may already have enough acks and can be committed, which is what this code does.

    if (hasCommitted && p.request!=null && p.request.getHdr().getType() == OpCode.reconfig){
           long curZxid = zxid;
       while (allowedToCommit && hasCommitted && p!=null){
           curZxid++;
           p = outstandingProposals.get(curZxid);
           if (p !=null) hasCommitted = tryToCommit(p, curZxid, null);             
       }
    }
}
```

调用实现，最终由CommitProcessor 接着处理请求：

```java
/**
 * @return True if committed, otherwise false.
 * @param a proposal p
 **/
synchronized public boolean tryToCommit(Proposal p, long zxid, SocketAddress followerAddr) {       
   // make sure that ops are committed in order. With reconfigurations it is now possible
   // that different operations wait for different sets of acks, and we still want to enforce
   // that they are committed in order. Currently we only permit one outstanding reconfiguration
   // such that the reconfiguration and subsequent outstanding ops proposed while the reconfig is
   // pending all wait for a quorum of old and new config, so it's not possible to get enough acks
   // for an operation without getting enough acks for preceding ops. But in the future if multiple
   // concurrent reconfigs are allowed, this can happen.
   if (outstandingProposals.containsKey(zxid - 1)) return false;
   
   // getting a quorum from all necessary configurations
    if (!p.hasAllQuorums()) {
       return false;                 
    }
    
    // commit proposals in order
    if (zxid != lastCommitted+1) {    
       LOG.warn("Commiting zxid 0x" + Long.toHexString(zxid)
                + " from " + followerAddr + " not first!");
        LOG.warn("First is "
                + (lastCommitted+1));
    }     
    
    // in order to be committed, a proposal must be accepted by a quorum              
    
    outstandingProposals.remove(zxid);
    
    if (p.request != null) {
         toBeApplied.add(p);
    }

    if (p.request == null) {
        LOG.warn("Going to commmit null: " + p);
    } else if (p.request.getHdr().getType() == OpCode.reconfig) {                                   
        LOG.debug("Committing a reconfiguration! " + outstandingProposals.size()); 
             
        //if this server is voter in new config with the same quorum address, 
        //then it will remain the leader
        //otherwise an up-to-date follower will be designated as leader. This saves
        //leader election time, unless the designated leader fails                             
        Long designatedLeader = getDesignatedLeader(p, zxid);
        //LOG.warn("designated leader is: " + designatedLeader);

        QuorumVerifier newQV = p.qvAcksetPairs.get(p.qvAcksetPairs.size()-1).getQuorumVerifier();
   
        self.processReconfig(newQV, designatedLeader, zk.getZxid(), true);

        if (designatedLeader != self.getId()) {
            allowedToCommit = false;
        }
               
        // we're sending the designated leader, and if the leader is changing the followers are 
        // responsible for closing the connection - this way we are sure that at least a majority of them 
        // receive the commit message.
        commitAndActivate(zxid, designatedLeader);
        informAndActivate(p, designatedLeader);
        //turnOffFollowers();
    } else {
        commit(zxid);   <<<<<<
        inform(p);			<<<<<<
    }
    zk.commitProcessor.commit(p.request);  <<<<<<
    if(pendingSyncs.containsKey(zxid)){
        for(LearnerSyncRequest r: pendingSyncs.remove(zxid)) {
            sendSync(r);
        }               
    } 
    
    return  true;   
}
```

该程序第一步是发送一个请求到Quorum的所有成员

```java
/**
 * Create a commit packet and send it to all the members of the quorum
 *
 * @param zxid
 */
public void commit(long zxid) {
    synchronized(this){
        lastCommitted = zxid;
    }
    QuorumPacket qp = new QuorumPacket(Leader.COMMIT, zxid, null, null);
    sendPacket(qp);
}
```

```java
/**
 * send a packet to all the followers ready to follow
 *
 * @param qp
 *                the packet to be sent
 */
void sendPacket(QuorumPacket qp) {
    synchronized (forwardingFollowers) {
        for (LearnerHandler f : forwardingFollowers) {
            f.queuePacket(qp);
        }
    }
}
```

第二步是通知Observer

```java
/**
 * Create an inform packet and send it to all observers.
 * @param zxid
 * @param proposal
 */
public void inform(Proposal proposal) {
    QuorumPacket qp = new QuorumPacket(Leader.INFORM, proposal.request.zxid,
                                        proposal.packet.getData(), null);
    sendObserverPacket(qp);
}
```

第三步到

```
 zk.commitProcessor.commit(p.request);
```

# CommitProcessor

CommitProcessor是多线程的，线程间通信通过queue，atomic和wait/notifyAll同步。CommitProcessor扮演一个网关角色，允许请求到剩下的处理管道。在同一瞬间，它支持多个读请求而仅支持一个写请求，这是为了保证写请求在事务中的顺序。
1个commit处理主线程，**它监控请求队列，并将请求分发到工作线程，分发过程基于sessionId，这样特定session的读写请求通常分发到同一个线程，因而可以保证运行的顺序**。
0~N个工作进程，他们在请求上运行剩下的请求处理管道。如果配置为0个工作线程，主commit线程将会直接运行管道。
经典(默认)线程数是：在32核的机器上，一个commit处理线程和32个工作线程。
多线程的限制：
	**每个session的请求处理必须是顺序的。**
	**写请求处理必须按照zxid顺序。**
	**必须保证一个session内不会出现写条件竞争，条件竞争可能导致另外一个session的读请求触发监控。**
当前实现解决第三个限制，仅仅通过不允许在写请求时允许读进程的处理。

```java
 @Override
 public void run() {
     try {
         /*
          * In each iteration of the following loop we process at most
          * requestsToProcess requests of queuedRequests. We have to limit
          * the number of request we poll from queuedRequests, since it is
          * possible to endlessly poll read requests from queuedRequests, and
          * that will lead to a starvation of non-local committed requests.
          */
         int requestsToProcess = 0;
         boolean commitIsWaiting = false;
				 do {
             /*
              * Since requests are placed in the queue before being sent to
              * the leader, if commitIsWaiting = true, the commit belongs to
              * the first update operation in the queuedRequests or to a
              * request from a client on another server (i.e., the order of
              * the following two lines is important!).
              */
             commitIsWaiting = !committedRequests.isEmpty();
             requestsToProcess =  queuedRequests.size();
             // Avoid sync if we have something to do
             if (requestsToProcess == 0 && !commitIsWaiting){
                 // Waiting for requests to process
                 synchronized (this) {
                     while (!stopped && requestsToProcess == 0
                             && !commitIsWaiting) {
                         wait();
                         commitIsWaiting = !committedRequests.isEmpty();
                         requestsToProcess = queuedRequests.size();
                     }
                 }
             }
             /*
              * Processing up to requestsToProcess requests from the incoming
              * queue (queuedRequests), possibly less if a committed request
              * is present along with a pending local write. After the loop,
              * we process one committed request if commitIsWaiting.
              */
             Request request = null;
             while (!stopped && requestsToProcess > 0
                     && (request = queuedRequests.poll()) != null) {
                 requestsToProcess--;
                 if (needCommit(request)
                         || pendingRequests.containsKey(request.sessionId)) {
                     // Add request to pending
                     LinkedList<Request> requests = pendingRequests
                             .get(request.sessionId);
                     if (requests == null) {
                         requests = new LinkedList<Request>();
                         pendingRequests.put(request.sessionId, requests);
                     }
                     requests.addLast(request);
                 }
                 else {
                     sendToNextProcessor(request);
                 }
                 /*
                  * Stop feeding the pool if there is a local pending update
                  * and a committed request that is ready. Once we have a
                  * pending request with a waiting committed request, we know
                  * we can process the committed one. This is because commits
                  * for local requests arrive in the order they appeared in
                  * the queue, so if we have a pending request and a
                  * committed request, the committed request must be for that
                  * pending write or for a write originating at a different
                  * server.
                  */
                 if (!pendingRequests.isEmpty() && !committedRequests.isEmpty()){
                     /*
                      * We set commitIsWaiting so that we won't check
                      * committedRequests again.
                      */
                     commitIsWaiting = true;
                     break;
                 }
             }

             // Handle a single committed request
             if (commitIsWaiting && !stopped){
                 waitForEmptyPool();

                 if (stopped){
                     return;
                 }

                 // Process committed head
                 if ((request = committedRequests.poll()) == null) {
                     throw new IOException("Error: committed head is null");
                 }

                 /*
                  * Check if request is pending, if so, update it with the
                  * committed info
                  */
                 LinkedList<Request> sessionQueue = pendingRequests
                         .get(request.sessionId);
                 if (sessionQueue != null) {
                     // If session queue != null, then it is also not empty.
                     Request topPending = sessionQueue.poll();
                     if (request.cxid != topPending.cxid) {
                         LOG.error(
                                 "Got cxid 0x"
                                         + Long.toHexString(request.cxid)
                                         + " expected 0x" + Long.toHexString(
                                                 topPending.cxid)
                                 + " for client session id "
                                 + Long.toHexString(request.sessionId));
                         throw new IOException("Error: unexpected cxid for"
                                 + "client session");
                     }
                     /*
                      * We want to send our version of the request. the
                      * pointer to the connection in the request
                      */
                     topPending.setHdr(request.getHdr());
                     topPending.setTxn(request.getTxn());
                     topPending.zxid = request.zxid;
                     request = topPending;
                 }

                 sendToNextProcessor(request);

                 waitForEmptyPool();sendToNextProcessor

                 /*
                  * Process following reads if any, remove session queue if
                  * empty.
                  */
                 if (sessionQueue != null) {
                     while (!stopped && !sessionQueue.isEmpty()
                             && !needCommit(sessionQueue.peek())) {
                         sendToNextProcessor(sessionQueue.poll());
                     }
                     // Remove empty queues
                     if (sessionQueue.isEmpty()) {
                         pendingRequests.remove(request.sessionId);
                     }
                 }
             }
         } while (!stoppedMainLoop);
     } catch (Throwable e) {
         handleException(this.getName(), e);
     }
     LOG.info("CommitProcessor exited loop!");
 }
```

```java
/**
 * Schedule final request processing; if a worker thread pool is not being
 * used, processing is done directly by this thread.
 */
private void sendToNextProcessor(Request request) {
    numRequestsProcessing.incrementAndGet();
    workerPool.schedule(new CommitWorkRequest(request), request.sessionId);
}
```

```java
/**
 * Schedule work to be done by the thread assigned to this id. Thread
 * assignment is a single mod operation on the number of threads.  If a
 * worker thread pool is not being used, work is done directly by
 * this thread.
 */
public void schedule(WorkRequest workRequest, long id) {
    if (stopped) {
        workRequest.cleanup();
        return;
    }

    ScheduledWorkRequest scheduledWorkRequest =
        new ScheduledWorkRequest(workRequest);

    // If we have a worker thread pool, use that; otherwise, do the work
    // directly.
    int size = workers.size();
    if (size > 0) {
        try {
            // make sure to map negative ids as well to [0, size-1] 这里根据 sessionid 分配到对应的线程去处理
            int workerNum = ((int) (id % size) + size) % size;
            ExecutorService worker = workers.get(workerNum);
            worker.execute(scheduledWorkRequest);
        } catch (RejectedExecutionException e) {
            LOG.warn("ExecutorService rejected execution", e);
            workRequest.cleanup();
        }
    } else {
      	// 如果没有配置 workSize 即表示不采用多线程，此时单线程执行
        // When there is no worker thread pool, do the work directly
        // and wait for its completion
        scheduledWorkRequest.start();
        try {
            scheduledWorkRequest.join();
        } catch (InterruptedException e) {
            LOG.warn("Unexpected exception", e);
            Thread.currentThread().interrupt();
        }
    }
}
```

子线程的处理逻辑

```java
private class ScheduledWorkRequest extends ZooKeeperThread {
    private final WorkRequest workRequest;

    ScheduledWorkRequest(WorkRequest workRequest) {
        super("ScheduledWorkRequest");
        this.workRequest = workRequest;
    }

    @Override
    public void run() {
        try {
            // Check if stopped while request was on queue
            if (stopped) {
                workRequest.cleanup();
                return;
            }
            workRequest.doWork();
        } catch (Exception e) {
            LOG.warn("Unexpected exception", e);
            workRequest.cleanup();
        }
    }
}
```

调用commitProcessor的doWork方法

```java
public void doWork() throws RequestProcessorException {
    try {
        nextProcessor.processRequest(request); // 调用下一个处理器
    } finally {
        if (numRequestsProcessing.decrementAndGet() == 0){
            wakeupOnEmpty();
        }
    }
}
```

将请求传递给下一个RP：Leader.ToBeAppliedRequestProcessor

# Leader.ToBeAppliedRequestProcessor

Leader.ToBeAppliedRequestProcessor仅仅维护一个toBeApplied列表。

```java
static class ToBeAppliedRequestProcessor implements RequestProcessor {
    private final RequestProcessor next;

    private final Leader leader;

    /**
     * This request processor simply maintains the toBeApplied list. For
     * this to work next must be a FinalRequestProcessor and
     * FinalRequestProcessor.processRequest MUST process the request
     * synchronously!
     *
     * @param next
     *                a reference to the FinalRequestProcessor
     */
    ToBeAppliedRequestProcessor(RequestProcessor next, Leader leader) {
        if (!(next instanceof FinalRequestProcessor)) {
            throw new RuntimeException(ToBeAppliedRequestProcessor.class
                    .getName()
                    + " must be connected to "
                    + FinalRequestProcessor.class.getName()
                    + " not "
                    + next.getClass().getName());
        }
        this.leader = leader;
        this.next = next;
    }

    /*
     * (non-Javadoc)
     *
     * @see org.apache.zookeeper.server.RequestProcessor#processRequest(org.apache.zookeeper.server.Request)
     */
    public void processRequest(Request request) throws RequestProcessorException {
        next.processRequest(request);

        // The only requests that should be on toBeApplied are write
        // requests, for which we will have a hdr. We can't simply use
        // request.zxid here because that is set on read requests to equal
        // the zxid of the last write op.
        if (request.getHdr() != null) {
            long zxid = request.getHdr().getZxid();
            Iterator<Proposal> iter = leader.toBeApplied.iterator();
            if (iter.hasNext()) {
                Proposal p = iter.next();
                if (p.request != null && p.request.zxid == zxid) {
                    iter.remove();
                    return;
                }
            }
            LOG.error("Committed request not found on toBeApplied: "
                      + request);
        }
    }

    /*
     * (non-Javadoc)
     *
     * @see org.apache.zookeeper.server.RequestProcessor#shutdown()
     */
    public void shutdown() {
        LOG.info("Shutting down");
        next.shutdown();
    }
}
```

# FinalRequestProcessor

 FinalRequestProcessor，这个请求处理器实际上应用到一个请求的所有事务，针对任何查询提供服务。它通常处于请求处理的最后（不会有下一个消息处理器），故此得名。 它是如何处理请求呢？

```java
public void processRequest(Request request) {
    if (LOG.isDebugEnabled()) {
        LOG.debug("Processing request:: " + request);
    }
    // request.addRQRec(">final");
    long traceMask = ZooTrace.CLIENT_REQUEST_TRACE_MASK;
    if (request.type == OpCode.ping) {
        traceMask = ZooTrace.SERVER_PING_TRACE_MASK;
    }
    if (LOG.isTraceEnabled()) {
        ZooTrace.logRequest(LOG, traceMask, 'E', request, "");
    }
    ProcessTxnResult rc = null;
    synchronized (zks.outstandingChanges) {
        // Need to process local session requests
      	// 根据共享的outstandingChanges，先处理事务后处理session
        rc = zks.processTxn(request);
		...
```

根据共享的 outstandingChanges，先处理事务后处理 session

```java
private ProcessTxnResult processTxn(Request request, TxnHeader hdr,
                                    Record txn) {
    ProcessTxnResult rc;
    int opCode = request != null ? request.type : hdr.getType();
    long sessionId = request != null ? request.sessionId : hdr.getClientId();
    if (hdr != null) {
        rc = getZKDatabase().processTxn(hdr, txn);
    } else {
        rc = new ProcessTxnResult();
    }
    if (opCode == OpCode.createSession) {
        if (hdr != null && txn instanceof CreateSessionTxn) {
            CreateSessionTxn cst = (CreateSessionTxn) txn;
            sessionTracker.addGlobalSession(sessionId, cst.getTimeOut());
        } else if (request != null && request.isLocalSession()) {
            request.request.rewind();
            int timeout = request.request.getInt();
            request.request.rewind();
            sessionTracker.addSession(request.sessionId, timeout);
        } else {
            LOG.warn("*****>>>>> Got "
                    + txn.getClass() + " "
                    + txn.toString());
        }
    } else if (opCode == OpCode.closeSession) {
        sessionTracker.removeSession(sessionId);
    }
    return rc;
}
```

处理事务，本地和数据库的不同分支， DataTree创建节点

```java
public ProcessTxnResult processTxn(TxnHeader header, Record txn)
{
    ProcessTxnResult rc = new ProcessTxnResult();

    try {
        rc.clientId = header.getClientId();
        rc.cxid = header.getCxid();
        rc.zxid = header.getZxid();
        rc.type = header.getType();
        rc.err = 0;
        rc.multiResult = null;
        switch (header.getType()) {
            case OpCode.create:
                CreateTxn createTxn = (CreateTxn) txn;
                rc.path = createTxn.getPath();
                createNode(
                        createTxn.getPath(),
                        createTxn.getData(),
                        createTxn.getAcl(),
                        createTxn.getEphemeral() ? header.getClientId() : 0,
                        createTxn.getParentCVersion(),
                        header.getZxid(), header.getTime(), null);
                break;
...
```

新增一个节点的逻辑是：

```java
/**
 * Add a new node to the DataTree.
 * @param path
 *             Path for the new node.
 * @param data
 *            Data to store in the node.
 * @param acl
 *            Node acls
 * @param ephemeralOwner
 *            the session id that owns this node. -1 indicates this is not
 *            an ephemeral node.
 * @param zxid
 *            Transaction ID
 * @param time
 * @param outputStat
 *             A Stat object to store Stat output results into.
 * @throws NodeExistsException 
 * @throws NoNodeException 
 * @throws KeeperException
 */
public void createNode(final String path, byte data[], List<ACL> acl,
        long ephemeralOwner, int parentCVersion, long zxid, long time, Stat outputStat)
        throws KeeperException.NoNodeException,
        KeeperException.NodeExistsException {
    int lastSlash = path.lastIndexOf('/');
    String parentName = path.substring(0, lastSlash);
    String childName = path.substring(lastSlash + 1);
    StatPersisted stat = new StatPersisted();
    stat.setCtime(time);
    stat.setMtime(time);
    stat.setCzxid(zxid);
    stat.setMzxid(zxid);
    stat.setPzxid(zxid);
    stat.setVersion(0);
    stat.setAversion(0);
    stat.setEphemeralOwner(ephemeralOwner);
    DataNode parent = nodes.get(parentName);
    if (parent == null) {
        throw new KeeperException.NoNodeException();
    }
    synchronized (parent) {
        Set<String> children = parent.getChildren();
        if (children != null && children.contains(childName)) {
            throw new KeeperException.NodeExistsException();
        }

        if (parentCVersion == -1) {
            parentCVersion = parent.stat.getCversion();
            parentCVersion++;
        }
        parent.stat.setCversion(parentCVersion);
        parent.stat.setPzxid(zxid);
        Long longval = aclCache.convertAcls(acl);
      	// 添加子节点
        DataNode child = new DataNode(data, longval, stat);
        parent.addChild(childName);
        nodes.put(path, child);
        EphemeralType ephemeralType = EphemeralType.get(ephemeralOwner);
        if (ephemeralType == EphemeralType.CONTAINER) {
            containers.add(path);
        } else if (ephemeralType == EphemeralType.TTL) {
            ttls.add(path);
        } else if (ephemeralOwner != 0) {
            HashSet<String> list = ephemerals.get(ephemeralOwner);
            if (list == null) {
                list = new HashSet<String>();
                ephemerals.put(ephemeralOwner, list);
            }
            synchronized (list) {
                list.add(path);
            }
        }
        if (outputStat != null) {
           child.copyStat(outputStat);
        }
    }
    // now check if its one of the zookeeper node child
    if (parentName.startsWith(quotaZookeeper)) {
        // now check if its the limit node
        if (Quotas.limitNode.equals(childName)) {
            // this is the limit node
            // get the parent and add it to the trie
            pTrie.addPath(parentName.substring(quotaZookeeper.length()));
        }
        if (Quotas.statNode.equals(childName)) {
            updateQuotaForPath(parentName
                    .substring(quotaZookeeper.length()));
        }
    }
    // also check to update the quotas for this node
    String lastPrefix = getMaxPrefixWithQuota(path);
    if(lastPrefix != null) {
        // ok we have some match and need to update
        updateCount(lastPrefix, 1);
        updateBytes(lastPrefix, data == null ? 0 : data.length);
    }
    // 最后的逻辑是触发创建节点和子节点改变事件
    dataWatches.triggerWatch(path, Event.EventType.NodeCreated);
    childWatches.triggerWatch(parentName.equals("") ? "/" : parentName,
            Event.EventType.NodeChildrenChanged);
}
```

最后的逻辑是触发创建节点和子节点改变事件

```java
Set<Watcher> triggerWatch(String path, EventType type, Set<Watcher> supress) {
    WatchedEvent e = new WatchedEvent(type,
            KeeperState.SyncConnected, path);
    HashSet<Watcher> watchers;
    synchronized (this) {
        watchers = watchTable.remove(path);
        if (watchers == null || watchers.isEmpty()) {
            if (LOG.isTraceEnabled()) {
                ZooTrace.logTraceMessage(LOG,
                        ZooTrace.EVENT_DELIVERY_TRACE_MASK,
                        "No watchers for " + path);
            }
            return null;
        }
        for (Watcher w : watchers) {
            HashSet<String> paths = watch2Paths.get(w);
            if (paths != null) {
                paths.remove(path);
            }
        }
    }
    for (Watcher w : watchers) {
        if (supress != null && supress.contains(w)) {
            continue;
        }
      	// 这里
        w.process(e);
    }
    return watchers;
}
```

WatcherManager 调用定义的 watcher 进行事件处理。

# 小结

从上面的分析可以知道，leader处理请求的顺序分别是：PrepRequestProcessor -> ProposalRequestProcessor ->CommitProcessor -> Leader.ToBeAppliedRequestProcessor -> FinalRequestProcessor

PrepRequestProcessor接收请求，并进行包装，然后请求类型的不同，设置共享数据；主要负责通知所有follower和observer；CommitProcessor 启动多线程处理请求；
Leader.ToBeAppliedRequestProcessor仅仅维护一个toBeApplied列表；
FinalRequestProcessor来作为消息处理器的终结者，发送响应消息，并触发watcher的处理程序。