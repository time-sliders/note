    install redis with 10 ports
    #!/bin/bash
    #---------------Environment Variables------------------
    REDISVER="redis-2.4.15"
    REDISDIR=`pwd`/$REDISVER        #Redis directory
    REDISINSNUM=10                  #Redis instance numbers
    #--------------Install redis--------------------------- 
    tar zxvf $REDISVER.tar.gz 
    cd $REDISVER
    make
    #make test
    mkdir bin conf logs data
    cp src/redis-server src/redis-cli src/redis-benchmark bin/
    mv redis.conf conf/
    cp conf/redis.conf conf/redis.conf.bak
    #-------------Configuration----------------------------
    sed -i 's#daemonize no#daemonize yes#' conf/redis.conf
    sed -i "s#pidfile /var/run/redis.pid#pidfile $REDISDIR/bin/redis.pid#" conf/redis.conf
    sed -i "s#logfile stdout#logfile $REDISDIR/logs/stdout.log#" conf/redis.conf
    sed -i "s#dbfilename dump.rdb#dbfilename $REDISDIR/data/dump.rdb#" conf/redis.conf
    bin/redis-server conf/redis.conf 
    #-----------Create multiple instances------------------
    #端口号6380-6389
    mkdir  servers && cd servers
    for ((i=0;i<$REDISINSNUM;i++));do
        mkdir $i
        cd $i
        mkdir bin conf logs data
        cp $REDISDIR/conf/redis.conf.bak conf/redis.conf
        sed -i 's#daemonize no#daemonize yes#' conf/redis.conf
        sed -i "s#port 6379#port 638$i#" conf/redis.conf
        sed -i "s#pidfile /var/run/redis.pid#pidfile $REDISDIR/servers/$i/bin/redis.pid#" conf/redis.conf
        sed -i "s#logfile stdout#logfile $REDISDIR/servers/$i/logs/stdout.log#" conf/redis.conf
        sed -i "s#dbfilename dump.rdb#dbfilename $REDISDIR/servers/$i/data/dump.rdb#" conf/redis.conf
    #   $REDISDIR/bin/redis-server $REDISDIR/servers/$i/conf/redis.conf 
        cd ..
    done
