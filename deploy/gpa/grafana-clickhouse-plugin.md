# grafana安装clickhouse插件

## 1.安装插件

``` text
grafana-cli plugins install vertamedia-clickhouse-datasource
```

## 2.重启grafana

注意需要将数据目录(插件安装目录默认为/var/lib/grafana/plugins)做持久化，否则重启后插件会丢失
