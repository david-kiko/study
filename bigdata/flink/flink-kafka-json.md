# flink对接kafka，处理json数据

## 1.flink定义kafka metadata table

``` text
CREATE TABLE `appupload`(
source string,
msg row<appProfile row<appPackageName string,
                       appVersionCode string,
                       appVersionName string,
                       appstoreId string,
                       installationTime timestamp,
                       isCracked boolean,
                       partnerId string,
                       purchaseTime timestamp,
                       sdkVersion string,
                       startTime timestamp,
                       mStatus tinyint,
                       `start` timestamp>,
        deviceProfile row<advertisingId string,
                          carrier string,
                          cellId string,
                          channel string,
                          connectType string,
                          country string,
                          cpuabi string,
                          deviceName string,
                          gis row<lat float,lng float>,
                          hostname string,
                          isJailbroken boolean,
                          is_new_inst boolean,
                          kernBootTime timestamp,
                          lac string,
                          `language` string,
                          mobileModel string,
                          mobileType string,
                          networkOperator string,
                          osVersion string,
                          ossdkVersion tinyint,
                          pixelmetric string,
                          pre_app_version string,
                          simOperator string,
                          timezone tinyint,
                          wifiBssid string>,
        developerAppkey string,
        deviceId string,
        ip string,
        message array<row<msgtype tinyint,
                          session row<activity array<row<name string,
                                                         sessionId string,
                                                         refer string,
                                                         parameters string,
                                                         `start` timestamp,
                                                         duration int>>,
                                       duration int,
                                       event array<row<`count` int,
                                                       label string,
                                                       parameters row<screen_name string,
                                                                      element_id string,
                                                                      label string,
                                                                      element_type string,
                                                                      title string,
                                                                      element_content string>,
                                                       sessionId string,
                                                       id string,
                                                       startTime timestamp>>,
                                       id string,
                                       mStatus tinyint,
                                       `start` bigint
                                       >>>
        >,
ts as msg['message'][1]['session']['start'],
ts_ltz AS TO_TIMESTAMP_LTZ(msg['message'][1]['session']['start'], 3),
WATERMARK FOR ts_ltz AS ts_ltz - INTERVAL '10' SECOND
)
WITH(
'connector' = 'kafka',
'topic' = 'appupload',
'properties.bootstrap.servers' = 'testbee.cguardian.com:10012',
'properties.group.id' = 'testGroup',
'format' = 'json',
'scan.startup.mode' = 'earliest-offset',
'json.fail-on-missing-field' = 'false',
'json.ignore-parse-errors' = 'true'
);
```

## 2.解析json数据

``` text
select day_str, source, count(1) FROM (SELECT distinct deviceId,source,FROM_UNIXTIME(msg['message'][1]['session']['start'] / 1000, 'yyyy-MM-dd') as day_str FROM ods_appupload) t group by day_str, source;
```

## 3.滑动窗口计算

``` text
SELECT window_start, window_end, source, count(1)
  FROM TABLE(
    HOP(TABLE appupload, DESCRIPTOR(ts_ltz), INTERVAL '5' MINUTES, INTERVAL '60' MINUTES))
  GROUP BY window_start, window_end, source
;
```

FlinkSQL窗口 <https://www.modb.pro/db/78793>
