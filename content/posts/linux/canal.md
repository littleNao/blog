---
date: '2025-06-05T16:16:06+08:00'
draft: false
title: 'Canal部署记录'

tags: ['服务器', '数据库', 'canal']
categories: ['服务器']
---
### 搭建时MySql相关版本配置与准备工作
华为云数据库MySQL版本：5.7.38
- MySQL canal账号创建：
 1. 可以在控制台通过网页进行创建
 2. 也可以通过命令创建：
```SQL
-- 连接华为云数据库（MySQL只有内网IP的情况）
mysql -h内网地址 -P端口号 -uroot -p
-- 新增canal用户 用户名：canal 密码：Canal@123456
CREATE USER 'canal'@'%' IDENTIFIED BY 'Canal@123456';
-- 授权 *.*表示所有库
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%' IDENTIFIED BY 'Canal@123456';
-- 授权后，需要刷新权限
FLUSH PRIVILEGES;
```
MySQL配置文件my.cnf设置：
> tip: 在华为云服务器上应该是默认开启的，没有找到设置项
```
[mysqld]
# 打开binlog
log-bin=mysql-bin
# 选择ROW(行)模式
binlog-format=ROW
# 配置MySQL replaction需要定义，不要和canal的slaveId重复
server_id=1
```
如果修改了配置文件需要重启MySQL，重启后使用命令查看是否开启binlog模式：
```SQL
show variables like 'log_bin';
```
显示为ON则表示开启成功  

查看binlog日志文件列表：
```SQL
show binary logs;
```
查看当前正在写入的binlog文件：
```SQL
show master status;
```
### 安装Canal-Deployer
1. [下载canal-deployer安装包](https://github.com/alibaba/canal/releases/)  
这里安装的是1.1.5版本：canal.deployer-1.1.5.tar.gz  
2. 服务器新建一个deployer目录，将文件上传到目录中并解压  
`tar -zxvf canal.deployer-1.1.5.tar.gz`
3. 打开配置文件 conf/example/instance.properties
```properties
#################################################
## mysql serverId , v1.0.26+ will autoGen
# canal.instance.mysql.slaveId=0

# enable gtid use true/false
canal.instance.gtidon=false

# MySQL服务器的IP地址与端口号
canal.instance.master.address=127.0.0.1:3306
# binlog日志名称
canal.instance.master.journal.name=
# mysql主库链接时起始的binlog偏移量
canal.instance.master.position=
canal.instance.master.timestamp=
canal.instance.master.gtid=

# rds oss binlog
canal.instance.rds.accesskey=
canal.instance.rds.secretkey=
canal.instance.rds.instanceId=

# table meta tsdb info
canal.instance.tsdb.enable=true

# 前面数据库授权的canal账号和密码
canal.instance.dbUsername=canal
canal.instance.dbPassword=Canal@123456
canal.instance.connectionCharset = UTF-8
# enable druid Decrypt database password
canal.instance.enableDruid=false

# table regex
canal.instance.filter.regex=.*\\..*
# table black regex
canal.instance.filter.black.regex=mysql\\.slave_.*
```
> `canal.instance.master.journal.name=`和`canal.instance.master.position=`  
这两个参数可以设置可以不设置，不设置表示从最新的binlog日志开始同步  
如果设置必须查看binlog日志列表是否存在，如果不存在运行会报错！！！  
如果设置必须查看binlog日志列表是否存在，如果不存在运行会报错！！！  
如果设置必须查看binlog日志列表是否存在，如果不存在运行会报错！！！

4. 配置完成后通过`sh bin/startup.sh`启动deployer  
可以查看 logs/example/example.log 日志，启动成功提示：
```log
2025-06-04 20:10:51.211 [main] INFO  c.a.otter.canal.instance.spring.CanalInstanceWithSpring - start CannalInstance for 1-example 
2025-06-04 20:10:51.227 [main] WARN  c.a.o.canal.parse.inbound.mysql.dbsync.LogEventConvert - --> init table filter : ^.*\..*$
2025-06-04 20:10:51.227 [main] WARN  c.a.o.canal.parse.inbound.mysql.dbsync.LogEventConvert - --> init table black filter : ^mysql\.slave_.*$
2025-06-04 20:10:51.240 [main] INFO  c.a.otter.canal.instance.core.AbstractCanalInstance - start successful....
```

到此deployer安装启动完成，但是只启动deployer是无法同步数据的，还需要启动adapter

### 安装Canal-Adapter
1. [下载canal-adapter安装包](https://github.com/alibaba/canal/releases/)  
这里安装的是1.1.5版本：canal.adapter-1.1.5.tar.gz
2. 与deployer同级新增adapter目录，将文件上传到目录中并解压  
`tar -zxvf canal.adapter-1.1.5.tar.gz`
3. 编辑配置文件 conf/application.yml
```yml
server:
  port: 8081
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
    default-property-inclusion: non_null

canal.conf:
  mode: tcp #tcp kafka rocketMQ rabbitMQ
  #flatMessage: true
  #zookeeperHosts:
  syncBatchSize: 1000
  retries: 0
  timeout:
  accessKey:
  secretKey:
  consumerProperties:
    # canal tcp consumer
    canal.tcp.server.host: 127.0.0.1:11111
    #canal.tcp.zookeeper.hosts:
    canal.tcp.batch.size: 500
    canal.tcp.username:
    canal.tcp.password:

  srcDataSources:
    defaultDS:
      # 需要同步的源数据库
      url: jdbc:mysql://127.0.0.1:3306/xxx?useUnicode=true
      # 授权的canal账号和密码
      username: canal
      password: Canal@123456
  canalAdapters:
  # 实例名称 如果deployer没有修改这里保持默认值就可以
  - instance: example # canal instance Name or mq topic name
    groups:
      # 适配器组ID。保持默认即可。
    - groupId: g1
      outerAdapters:
      # 适配器类型，我这里保持默认没有修改
      - name: rdb
        # 适配器标识 可自定义
        key: mysql1
        # 目标库信息
        properties:
          jdbc.driverClassName: com.mysql.jdbc.Driver
          jdbc.url: jdbc:mysql://127.0.0.1:3306/xxx?useUnicode=true
          jdbc.username: canal
          jdbc.password: Canal@123456
```
4. 针对多表同步配置，在conf/rdb/目录下添加每个表的配置文件  
我这里需要同步3张表所已添加了3个配置文件，内容大致相同：
```yml
# 源数据库标识 与application.yml文件中的srcDataSources保持一致。
dataSourceKey: defaultDS
# 实例名称 与application.yml文件中的canalAdapters.instance保持一致。
destination: example
# 适配器组ID 与application.yml文件中的canalAdapters.groups.groupId保持一致。
groupId: g1
# 适配器标识 与application.yml文件中的canalAdapters.groups.groupId.outerAdapters.key保持一致。
outerAdapterKey: mysql1
concurrent: true
dbMapping:
  # 源数据库
  database: sql_base
  # 源数据表
  table: sql_depart
  # 目标数据表
  targetTable: sql_committee
  targetPk:
    # 主键配置
    # 源表主键名称: 目标表主键名称
    id: id
  # 全量修改
  mapAll: true
```

`sh bin/startup.sh`命令启动adapter  
`tail -f logs/adapter.log`查看日志输出结果：
```log
2025-06-04 20:11:24.415 [main] INFO  c.a.o.canal.adapter.launcher.loader.CanalAdapterLoader - Load canal adapter: rdb succeed
2025-06-04 20:11:24.423 [main] INFO  c.alibaba.otter.canal.connector.core.spi.ExtensionLoader - extension classpath dir: /mnt/data/canal/adapter/plugin
2025-06-04 20:11:24.443 [main] INFO  c.a.o.canal.adapter.launcher.loader.CanalAdapterLoader - Start adapter for canal-client mq topic: example-g1 succeed
2025-06-04 20:11:24.443 [main] INFO  c.a.o.canal.adapter.launcher.loader.CanalAdapterService - ## the canal client adapters are running now ......
2025-06-04 20:11:24.451 [main] INFO  org.apache.coyote.http11.Http11NioProtocol - Starting ProtocolHandler ["http-nio-8081"]
2025-06-04 20:11:24.454 [Thread-4] INFO  c.a.otter.canal.adapter.launcher.loader.AdapterProcessor - =============> Start to connect destination: example <=============
2025-06-04 20:11:24.459 [main] INFO  org.apache.tomcat.util.net.NioSelectorPool - Using a shared selector for servlet write/read
2025-06-04 20:11:24.624 [main] INFO  o.s.boot.web.embedded.tomcat.TomcatWebServer - Tomcat started on port(s): 8081 (http) with context path ''
2025-06-04 20:11:24.627 [main] INFO  c.a.otter.canal.adapter.launcher.CanalAdapterApplication - Started CanalAdapterApplication in 3.303 seconds (JVM running for 3.714)
2025-06-04 20:11:24.737 [Thread-4] INFO  c.a.otter.canal.adapter.launcher.loader.AdapterProcessor - =============> Subscribe destination: example succeed <=============
```
到此adapter启动完成，数据同步开始
### 踩坑记录
1. 配置授权后没有使用`FLUSH PRIVILEGES;`命令刷新授权。
2. canal数据库密码设置过于复杂，启动时报错密码验证不通过。
3. 只配置了deployer，没有配置adapter，导致可以看到同步记录却没有执行。
4. 配置了过早的`canal.instance.master.journal.name=`和`canal.instance.master.position=` 导致启动时一直报错找不到同步节点。
> 这里如果不考虑旧的同步数据可以不设置，会从最新节点同步。
5. 如果不设置`canal.instance.master.journal.name=`和`canal.instance.master.position=` 要把deployer的`/conf/example/`目录下`meta.bat`文件删除，否则`meta.bat`文件会存有旧的节点数据。
### 2025.07.17 多库同步配置
1. 修改 deployer/conf/canal.properties 配置文件
```yaml
#################################################
#########               destinations            #############
#################################################
# 配置多个通过,分割
canal.destinations = example,test1,test2
```
2. 修改 adapter/conf/application.yml 配置文件
```yaml
server:
  port: 8081
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
    default-property-inclusion: non_null

canal.conf:
  mode: tcp #tcp kafka rocketMQ rabbitMQ
  #flatMessage: true
  #zookeeperHosts:
  syncBatchSize: 1000
  retries: 0
  timeout:
  accessKey:
  secretKey:
  consumerProperties:
    # canal tcp consumer
    canal.tcp.server.host: 127.0.0.1:11111
    #canal.tcp.zookeeper.hosts:
    canal.tcp.batch.size: 500
    canal.tcp.username:
    canal.tcp.password:

  srcDataSources:
    defaultDS:
      # 需要同步的源数据库
      url: jdbc:mysql://127.0.0.1:3306/db?useUnicode=true
      # 授权的canal账号和密码
      username: canal
      password: Canal@123456
  canalAdapters:
  # 实例名称 如果deployer没有修改这里保持默认值就可以
  - instance: example # canal instance Name or mq topic name
    groups:
      # 适配器组ID。保持默认即可。
    - groupId: g1
      outerAdapters:
      # 适配器类型，我这里保持默认没有修改
      - name: rdb
        # 适配器标识 可自定义
        key: mysql1
        # 目标库信息
        properties:
          jdbc.driverClassName: com.mysql.jdbc.Driver
          jdbc.url: jdbc:mysql://127.0.0.1:3306/test1?useUnicode=true
          jdbc.username: canal
          jdbc.password: Canal@123456
  # 与deployer/conf/canal.properties内多个名称保持一致
  - instance: test1 # canal instance Name or mq topic name
    groups:
      # 适配器组ID。保持默认即可。
    - groupId: g2
      outerAdapters:
      # 适配器类型，我这里保持默认没有修改
      - name: rdb
        # 适配器标识 可自定义
        key: mysql2
        # 目标库信息
        properties:
          jdbc.driverClassName: com.mysql.jdbc.Driver
          jdbc.url: jdbc:mysql://127.0.0.1:3306/test2?useUnicode=true
          jdbc.username: canal
          jdbc.password: Canal@123456
```
canalAdapters为一个列表，多个库需要配置时设置多个instance即可