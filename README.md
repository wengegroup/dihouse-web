# dihouse-web
DiHouse，面向海量异构数据存储、多模态数据处理、跨数据库实时查询与分析场景，采用数据联邦、分布式计算等技术手段,实现关系数据库、向量数据库、图数据库、搜索引擎等多种类数据联邦查询与融合应用。我们使用标准SQL语法支持用户业务建设，提供多模型数据分析、实时数据处理、存储与计算模块解耦、异构服务器混合部署能力。


### 1. dihouse-web部署

```sh
tar -zxvf dihouse-web.tar.gz
```
目录结构
```
drwxr-xr-x 4 isi isi   145 6月  29 09:58 bin
drwxr-xr-x 2 isi isi    57 6月  29 10:11 config
drwxr-xr-x 2 isi isi    22 6月  29 09:47 data
drwxr-xr-x 2 isi isi 16384 6月  29 09:52 lib
drwxrwxr-x 5 isi isi    43 6月  29 09:52 log
drwxrwxr-x 2 isi isi    25 6月  29 09:52 logs
drwxr-xr-x 2 isi isi     6 6月  29 10:08 result
drwxrwxr-x 8 isi isi   156 6月  29 09:52 web
```
#### 1.1 前置基础组件依赖

- jdk >= 11
- mysql >= 8.0.28

#### 1.2 导入sql脚本

创建数据库dihouse
```sql
CREATE DATABASE dihouse CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```
导入数据

```sql
CREATE TABLE IF NOT EXISTS query (
  datasource varchar(256),
  engine varchar(256),
  query_id varchar(256),
  fetch_result_time_string varchar(256),
  query_string mediumtext,
  user varchar(256),
  status varchar(256),
  elapsed_time_millis integer,
  result_file_size bigint,
  linenumber integer,
  primary key(datasource, engine, query_id))
;

CREATE TABLE IF NOT EXISTS publish (
  publish_id varchar(256),
  datasource varchar(256),
  engine varchar(256),
  query_id varchar(256),
  user varchar(256),
  viewers text,
  primary key(publish_id))
;

CREATE TABLE IF NOT EXISTS bookmark (
  bookmark_id integer primary key auto_increment,
  datasource varchar(256),
  engine varchar(256),
  query text,
  title varchar(256),
  user varchar(256),
  snippet varchar(256))
;

CREATE TABLE IF NOT EXISTS comment (
  datasource varchar(256),
  engine varchar(256),
  query_id varchar(256),
  content text,
  update_time_string varchar(256),
  user varchar(256),
  like_count integer,
  primary key(datasource, engine, query_id));

CREATE TABLE IF NOT EXISTS starred_schema (
  starred_schema_id integer primary key auto_increment,
  datasource varchar(256) not null,
  engine varchar(256) not null,
  catalog varchar(256) not null,
  `schema` varchar(256) not null,
  user varchar(256));

CREATE TABLE IF NOT EXISTS session_property (
  session_property_id integer primary key auto_increment,
  datasource varchar(256) not null,
  engine varchar(256) not null,
  query_id varchar(256) not null,
  session_key varchar(256) not null,
  session_value varchar(256) not null);

```
#### 1.3 application.yaml 配置文件

```application.yaml
server:
  port: 8091
  jetty:
    max-http-form-post-size: 2GB
  servlet:
    encoding:
      enabled: true
      charset: utf-8
      force: true
spring:
  servlet:
    multipart.max-request-size: 1GB
    multipart.max-file-size: 1GB
  application:
    name: dihouse-server
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://xxx.xxx.xxx.xxx:3306/dihouse?allowPublicKeyRetrieval=true&useSSL=false
    username: xxx
    password: xxx
    initialization-mode: always
  resources:
    static-locations: file:web
  mvc:
    view:
      prefix: share/
# trinoUrl
dsconfig:
  INSERT_OR_UPDATE_URL: http://xxxxx:8080/v1/catalog/addOrUpdateCatalog
  DELETE_URL: http://xxxx:8080/v1/catalog/removeCatalogs/%s
# Metrics
management:
  metrics:
    export.prometheus.enabled: true
    distribution:
      percentiles:
        http.server.requests: 0.5, 0.75, 0.95, 0.99
  endpoint:
    metrics.enabled: true
    prometheus.enabled: true
    heapdump.enabled: false
    health:
      show-details: always
    env:
      keys-to-sanitize: .*password.*
  endpoints:
    web.exposure.include: '*'

#word等文件预览 这个暂时不用修改
kkFileViewURL: http://172.16.10.19:31755/onlinePreview?url=

# Datasources
sql.query.engines: dihouse
check.datasource: false
select.limit: 500
audit.http.header.name: some.auth.header
use.audit.http.header.name: false
to.values.query.limit: 500
cors.enabled: true

# diHouse
dihouse.datasources: diHouse
dihouse.query.max-run-time-seconds: 1800
dihouse.max-result-file-byte-size: 1073741824
dihouse.coordinator.server.diHouse: http://xxxxx:8080
dihouse.redirect.server.diHouse: http://xxxx:8080/ui
auth.diHouse: false
catalog.diHouse: tpch
schema.diHouse: sf1

mybatis-plus:
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl

dihouse:
  jdbc:
    url: jdbc:trino://xxxx:8080
    drivier: io.trino.jdbc.TrinoDriver
    username: root
    password:

```
参数说明：
- spring.datasouce.url: 数据库url 如`jdbc:mysql://xxxx:3306/dihouse?allowPublicKeyRetrieval=true&useSSL=false`
- spring.datasouce.username: 数据库用户名
- spring.datasouce.password: 数据库密码
- dsconfig.INSERT_OR_UPDATE_URL:  trino server 的URl+`/v1/catalog/addOrUpdateCatalog` 如： http://xxxx:8080/v1/catalog/addOrUpdateCatalog
- dsconfig.DELETE_URL: trino server 的URl+`/v1/catalog/removeCatalogs/%s` 如：http://xxxx:8080/v1/catalog/removeCatalogs/%s
- dihouse.coordinator.server.diHouse: Dihouse-Server的url  http://xxxxx:8080
- dihouse.redirect.server.diHouse: Dihouse-Server的web ui的url http://xxxxx:8080/ui
- spring.resources.static-locations: file:web 
- dihouse.jdbc.url: jdbc:trino://xxxxx:8080
- dihouse.jdbc.drivier: io.trino.jdbc.TrinoDriver
- dihouse.jdbc.username: root
- dihouse.jdbc.password: 

其他配置默认即可

#### 1.4 启动dihouse-web服务

在dihouse-web/bin目录下修改start.sh脚本中JAVA_HOME路径

```shell
#!/bin/bash
export JAVA_HOME=/xxxxx/jdk-11.0.16
export PATH=$JAVA_HOME/bin/:$PATH
nohup sh bin/dihouse-start.sh > logs/dihouse.log 2>&1 &
```
执行start.sh
```shell
sh +x bin/start.sh
```
#### 1.5 dihouse-web服务访问

http://xxxx:8091/
