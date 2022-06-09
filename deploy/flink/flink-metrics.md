# flink metrics部署

## 1.下载metrics包

``` text
wget https://repo1.maven.org/maven2/org/apache/flink/flink-metrics-prometheus/1.14.4/flink-metrics-prometheus-1.14.4.jar
```

## 2.配置flink-conf.yaml

``` text
metrics.reporters: prom

metrics.reporter.prom.class: org.apache.flink.metrics.prometheus.PrometheusReporter

metrics.reporter.prom.port: 9999
```

## 3.重启flink服务
