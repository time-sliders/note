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
    // 核心方法：开始群首选举
    startLeaderElection();
    // 核心方法：Main Loop
    super.start();
}
```

