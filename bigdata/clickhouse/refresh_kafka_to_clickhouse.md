# ClickHouse-Kafka引擎，数据重新刷新，重新写回所有数据的操作步骤

## 1.首先关闭Kafka消息使用

``` text
DETACH TABLE kafka_readings_queue
```

## 2.清空Kafka数据存储表

``` text
truncate table kafka_readings
```

## 3.在Kafka主题的订阅使用者组中重置分区偏移量

* --to-earliest：把位移调整到分区当前最小/早位移
* --to-latest：把位移调整到分区当前最新位移
* --to-current：把位移调整到分区当前位移
* --to-offset <offset>： 把位移调整到指定位移处
* --shift-by N： 把位移调整到当前位移 + N处，注意N可以是负数，表示向前移动
* --to-datetime <datetime>：把位移调整到大于给定时间的最早位移处，datetime格式是yyyy-MM-ddTHH:mm:ss.xxx，比如2017-08-04T00:00:00.000

``` text
kafka-consumer-groups.sh --bootstrap-server xxx --group test_new_ck --topic appupload -reset-offsets --to-earliest --dry-run

kafka-consumer-groups.sh --bootstrap-server xxx --group test_new_ck --topic appupload -reset-offsets --to-earliest --execute
```

## 4.重新激活Kafka消息的使用

``` text
ATTACH TABLE kafka_readings_queue
```
