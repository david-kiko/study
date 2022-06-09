# flink-connector 对接mysql binlog

## 1.配置mysql source table

``` text
CREATE TABLE `source_table`  (
  `source_columns`,
  `data_up_uuid` int,
  `data_up_time` TIMESTAMP(0),
  `data_up_status` string,
  PRIMARY KEY (data_up_uuid) NOT ENFORCED
) WITH (
    'connector' = 'mysql-cdc',
    'hostname' = 'xxx',
    'port' = '3306',
    'username' = 'user',
    'password' = 'pass',
    'database-name' = 'dbname',
    'table-name' = 'tablename',
    'server-time-zone' = 'Asia/Shanghai',
    'scan.startup.mode' = 'latest-offset',
    'scan.snapshot.fetch.size' = '2048',
    'debezium.skipped.operations'='u,d'
  );
```

## 2.创建sink table

``` text
CREATE TABLE `sink_table`  (
  `sink_columns`,
  `data_up_uuid` int,
  `data_up_time` TIMESTAMP(0),
  PRIMARY KEY (data_up_uuid) NOT ENFORCED
) WITH (
    'connector' = 'jdbc',
    'url' = 'jdbc:mysql://xxx/database?serverTimezone=Asia/Shanghai&useunicode=true&characterEncoding=utf8&yearIsDateType=false&zeroDateTimeBehavior=convertToNull&tinyInt1isBit=false&rewriteBatchedStatements=true',
    'username' = 'user',
    'password' = 'pass',
    'table-name' = 'tablename',
    'sink.buffer-flush.max-rows' = '2048',
    'sink.parallelism' = '12',
    'sink.buffer-flush.interval' = '2s'
  );
```

## 3.从source导入数据到sink

``` text
  insert into sink_table(`sink_columns`,`data_up_uuid`,`data_up_time`) 
  select `source_columns`,`data_up_uuid`,`data_up_time`
  from source_table;
```
