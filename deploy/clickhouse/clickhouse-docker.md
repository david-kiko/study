# clickhouse docker安装

## 一、搭建clcikhouse

### 1.docker-compose.yml

``` text
version: "3.6"

services:

  clickhouse-halobug:
    container_name: clickhouse-halobug.cn
    image: yandex/clickhouse-server:latest
    restart: always
    ports:
      - "8123:8123"
      - "9000:9000"
      - "9009:9009"
    networks:
      - traefik
    volumes: 
      # 默认配置 写入config.d/users.d 目录防止更新后文件丢失
      - ./config.xml:/etc/clickhouse-server/config.d/config.xml:rw
      - ./users.xml:/etc/clickhouse-server/users.xml:rw
      # 运行日志
      - ./logs:/var/log/clickhouse-server
      # 数据持久
      - ./data:/var/lib/clickhouse:rw
networks:
  traefik:
    external: true
```

### 2.配置文件内容生成

* 启动时先不挂载config.xml以及users.xml
* docker cp 容器名:/etc/clickhouse-server/config.xml .
* docker cp 容器名:/etc/clickhouse-server/uses.xml .
  
### 3.创建用户登录

修改users.xml文件，自定义用户

``` text
<halobug>
    <password_sha256_hex>4754fc7e290a9c280d9497b2d76dd854e77f7e1c92476577fdb52ed22afc13e7</password_sha256_hex>
    <networks incl="networks" replace="replace">
        <ip>::/0</ip>
    </networks>
    <profile>default</profile>
    <quota>default</quota>
    <allow_databases>
        // 自定义对应数据库 tutorial 为测试库
       <database>tutorial</database>
    </allow_databases>
</halobug>
```

生成密码（进入容器运行）

``` text
PASSWORD=$(base64 < /dev/urandom | head -c8); echo "$PASSWORD"; echo -n "$PASSWORD" | sha256sum | tr -d '-'
```

### 4.验证用户密码，重新启动服务

``` text
# 重启
docker-compose down && docker-compose up -d
# 进入容器
docker exec -it clickhouse-halobug.cn /bin/bash
# 验证用户名密码
clickhouse-client -u halobug -h 127.0.0.1 --password qjLlan5A
```

## 二、搭建tabix可视化操作

### 1.tabix docker-compose.yml

``` text
# docker-compose.yml
version: "3.6"

services:

  clickhouse-halobug:
    container_name: tabix-halobug.cn
    image: spoonest/clickhouse-tabix-web-client
    restart: always
    expose:
      - 80
    networks:
      - traefik
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=traefik"
      - "traefik.http.routers.tabix-test.entrypoints=https"
      - "traefik.http.routers.tabix-test.rule=Host(`tabix.halobug.cn`)"
      - "traefik.http.routers.tabix-test.tls=true"
      - "traefik.http.services.tabix-test-backend.loadbalancer.server.scheme=http"
      - "traefik.http.services.tabix-test-backend.loadbalancer.server.port=80"
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
networks:
  traefik:
    external: true
```

### 2.访问<https://127.0.0.1>

使用上面创建的用户及密码登录

## 三、测试命令

``` text
# 进入容器
docker exec -it clickhouse-halobug.cn /bin/bash

# 创建数据库
clickhouse-client --query "CREATE DATABASE IF NOT EXISTS test"

#使用用户名进入
clickhouse-client -u halobug -h 127.0.0.1 --password qjLlan5A

# 创建测试表
CREATE TABLE test.test_son
(
  `id` Int64,
  `name` String, 
  `title` String
) ENGINE = MergeTree ORDER BY id SETTINGS index_granularity = 8192

# 插入数据

INSERT INTO test.test_son VALUES ('1','Hello, world','标题测试'), ('2','Hello, world2','标题测试3')
```

## 四、参考文档

<https://zhuanlan.zhihu.com/p/383817560>
