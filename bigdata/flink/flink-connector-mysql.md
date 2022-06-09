# flink-connector 对接mysql binlog

## 1.配置mysql source table

``` text
CREATE TABLE `ybsj_xxb`  (
  `UID_SS` string,
  `CI_LASTUPDATE` TIMESTAMP(3) COMMENT '最后修改日期',
  `CI_LASTUPTIME` bigint COMMENT '最后修改时间',
  `CI_AUDITTAG` bigint COMMENT '审核标记',
  `DAY` string,
  `DIM_TBDW` string  COMMENT '填报单位',
  `YBBH` string COMMENT '唯一键',
  `CYRQ` TIMESTAMP(3) COMMENT '采样日期',
  `XM` string COMMENT '姓名',
  `NL` bigint COMMENT '年龄',
  `XB` string COMMENT '性别',
  `SFZH` string COMMENT '身份证号',
  `JCORG` string COMMENT '检测机构',
  `BSRQ` TIMESTAMP(3) COMMENT '系统填报日期',
  `YBLY` string COMMENT '样本来源',
  `ZJLX` string COMMENT '证件类型',
  `SYBM` string COMMENT '送样编号',
  `BLLXJJCCS` string COMMENT '病例类型及检测次数',
  `HZID` string COMMENT '患者ID',
  `LXDH` string COMMENT '联系电话',
  `YBLX` string COMMENT '样本类型',
  `SFJYLH` string COMMENT '人员类型原：境外返（来）汉人员字段沿用下来',
  `qycj_rwdm` string COMMENT '全员检测任务代码',
  `qycj_rwmc` string COMMENT '全员检测任务名称',
  `CYDD` string COMMENT '采样地点',
  `HYFS` string COMMENT '混样方式',
  `sjly` string,
  `create_time` TIMESTAMP(0),
  `sz_uptime` TIMESTAMP(0),
  `ssrq` string,
  `cyfzr` string,
  `lxfs` string,
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
CREATE TABLE `ybsj_xxb_s`  (
  `UID_SS` string,
  `CI_LASTUPDATE` TIMESTAMP(3) COMMENT '最后修改日期',
  `CI_LASTUPTIME` bigint COMMENT '最后修改时间',
  `CI_AUDITTAG` bigint COMMENT '审核标记',
  `DAY` string,
  `DIM_TBDW` string COMMENT '填报单位',
  `YBBH` string COMMENT '唯一键',
  `CYRQ` TIMESTAMP(3) COMMENT '采样日期',
  `XM` string COMMENT '姓名',
  `NL` bigint COMMENT '年龄',
  `XB` string COMMENT '性别',
  `SFZH` string COMMENT '身份证号',
  `JCORG` string COMMENT '检测机构',
  `BSRQ` TIMESTAMP COMMENT '系统填报日期',
  `YBLY` string COMMENT '样本来源',
  `ZJLX` string COMMENT '证件类型',
  `SYBM` string COMMENT '送样编号',
  `BLLXJJCCS` string COMMENT '病例类型及检测次数',
  `HZID` string COMMENT '患者ID',
  `LXDH` string COMMENT '联系电话',
  `YBLX` string COMMENT '样本类型',
  `SFJYLH` string COMMENT '人员类型原：境外返（来）汉人员字段沿用下来',
  `qycj_rwdm` string COMMENT '全员检测任务代码',
  `qycj_rwmc` string COMMENT '全员检测任务名称',
  `CYDD` string COMMENT '采样地点',
  `HYFS` string COMMENT '混样方式',
  `sjly` string,
  `create_time` TIMESTAMP,
  `sz_uptime` TIMESTAMP,
  `ssrq` string,
  `cyfzr` string,
  `lxfs` string,
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
  insert into ybsj_xxb_s(`UID_SS`,`CI_LASTUPDATE`,`CI_LASTUPTIME`,`CI_AUDITTAG`,`DAY`,`DIM_TBDW`,`YBBH`,`CYRQ`,`XM`,`NL`,`XB`,`SFZH`,
                       `JCORG`,`BSRQ`,`YBLY`,`ZJLX`,`SYBM` ,`BLLXJJCCS`,`HZID`,`LXDH`,`YBLX`,`SFJYLH`,`qycj_rwdm`,`qycj_rwmc`,`CYDD`,
                        `HYFS`,`sjly`,`create_time`,`sz_uptime`,`ssrq`,`cyfzr`,`lxfs`,`data_up_uuid`,`data_up_time`) 
  select `UID_SS`,`CI_LASTUPDATE`,`CI_LASTUPTIME`,`CI_AUDITTAG`,`DAY`,`DIM_TBDW`,`YBBH`,`CYRQ`,`XM`,`NL`,`XB`,`SFZH`,
                       `JCORG`,`BSRQ`,`YBLY`,`ZJLX`,`SYBM` ,`BLLXJJCCS`,`HZID`,`LXDH`,`YBLX`,`SFJYLH`,`qycj_rwdm`,`qycj_rwmc`,`CYDD`,
                        `HYFS`,`sjly`,`create_time`,`sz_uptime`,`ssrq`,`cyfzr`,`lxfs`,`data_up_uuid`,`data_up_time`
  from ybsj_xxb;
```
