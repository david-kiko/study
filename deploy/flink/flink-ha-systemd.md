# flink ha systemd部署

## 1.下载解压flink包

``` text
wget https://linktime-external-project.oss-cn-hangzhou.aliyuncs.com/realtime/flink-1.13.2-bin-scala_2.11.tgz

tar -xzvf flink-1.13.2-bin-scala_2.11.tgz

mv flink-1.13.2 /etc/flink
```

## 2.下载connector及hadoop相关包

``` text
wget -O lib/flink-sql-connector-elasticsearch7_2.11-1.13.2.jar https://linktime-external-project.oss-cn-hangzhou.aliyuncs.com/realtime/flink-sql-connector-elasticsearch7_2.11-1.13.2.jar
wget -O lib/flink-sql-connector-mongodb-cdc-2.1.0.jar https://linktime-external-project.oss-cn-hangzhou.aliyuncs.com/realtime/flink-sql-connector-mongodb-cdc-2.1.0.jar
wget -O lib/flink-sql-connector-mysql-cdc-2.1.0.jar https://linktime-external-project.oss-cn-hangzhou.aliyuncs.com/realtime/flink-sql-connector-mysql-cdc-2.1.0.jar
wget -O lib/flink-sql-connector-oracle-cdc-2.1.1.jar https://linktime-external-project.oss-cn-hangzhou.aliyuncs.com/realtime/flink-sql-connector-oracle-cdc-2.1.1.jar
wget -O lib/flink-sql-connector-postgres-cdc-2.1.0.jar https://linktime-external-project.oss-cn-hangzhou.aliyuncs.com/realtime/flink-sql-connector-postgres-cdc-2.1.0.jar
wget -O lib/flink-connector-jdbc_2.11-1.13.2.jar https://linktime-external-project.oss-cn-hangzhou.aliyuncs.com/realtime/flink-connector-jdbc_2.11-1.13.2.jar
wget -O lib/flink-shaded-hadoop-2-uber-2.7.5-10.0.jar https://linktime-external-project.oss-cn-hangzhou.aliyuncs.com/realtime/flink-shaded-hadoop-2-uber-2.7.5-10.0.jar
```

## 3.配置flink conf

``` text
security.kerberos.login.use-ticket-cache: false
security.kerberos.login.keytab: /home/dcos/dcos.keytab
security.kerberos.login.principal: dcos@LINKTIME.CLOUD
env.java.opts: -Djava.security.krb5.conf=/etc/krb5.conf
state.backend: filesystem
state.checkpoints.dir: hdfs://default/flink/flink-checkpoints
state.savepoints.dir: hdfs://default/flink/flink-checkpoints
state.backend.incremental: false
high-availability: zookeeper
high-availability.zookeeper.quorum: zk-1.zk:2181,zk-2.zk:2181,zk-3.zk:2181,zk-4.zk:2181,zk-5.zk:2181
high-availability.storageDir: hdfs://default/flink/ha
high-availability.zookeeper.path.root: /flink
high-availability.cluster-id: /flink_cluster
jobmanager.archive.fs.dir: hdfs://default/flink/completed-jobs/
historyserver.archive.fs.dir: hdfs://default/flink/completed-jobs/
classloader.resolve-order: parent-first
restart-strategy: fixed-delay
restart-strategy.fixed-delay.delay: 60 s
cluster.evenly-spread-out-slots: true
```

## 4.配置hadoop路径

config.sh文件中添加export语句

``` text
export HADOOP_CONF_DIR=/etc/hadoop/etc/hadoop
```

## 5.设置systemd服务

jobmanager master

``` text
cat > /usr/lib/systemd/system/flink-jobmanager.service << EOF
[Unit]
Description=Flink standalone HA mode jobmanager

[Service]
Type=forking
User=dcos
Group=dcos
Environment=HADOOP_HDFS_HOME=/etc/hadoop
Environment=HADOOP_COMMON_HOME=/etc/hadoop
Environment=JAVA_HOME=/opt/mesosphere/active/java/usr/java
Environment=FLINK_HOME=/etc/flink
ExecStart=/etc/flink/bin/jobmanager.sh start agent2 18081
ExecStop=/etc/flink/bin/jobmanager.sh stop
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
EOF
```

taskmanager agent

``` text
cat > /usr/lib/systemd/system/flink-taskmanager.service << EOF
[Unit]
Description=Flink standalone HA mode taskmanager

[Service]
Type=forking
User=dcos
Group=dcos
Environment=HADOOP_HDFS_HOME=/etc/hadoop
Environment=HADOOP_COMMON_HOME=/etc/hadoop
Environment=JAVA_HOME=/opt/mesosphere/active/java/usr/java
Environment=FLINK_HOME=/etc/flink
ExecStart=/etc/flink/bin/taskmanager.sh start
ExecStop=/etc/flink/bin/taskmanager.sh stop
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
EOF
```

## 6.加载并设置开机启动

``` text
systemctl daemon-reload
systemctl start flink-taskmanager.service
systemctl enable flink-taskmanager.service
```
