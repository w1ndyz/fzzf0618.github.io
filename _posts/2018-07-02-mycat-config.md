---
layout: post
title:  “Mycat数据库读写分离”
date:   2018-07-02 11:38:59
author: Chris Wang
categories: ror
tags: mysql mycat 读写分离
---

### 下载地址

1. 去`http://dl.mycat.io/`下载对应的平台安装包
2. 下载版本为`1.6-RELEASE/`
3. 把mysql文件夹移动到 `/usr/local/` 下
4. 需要安装Java

    * 首先下载jdk, 版本`>=1.8`
    * 解压 `tar -zxvf jdk-8u91-linux-x64.tar.gz`
    * 设置JAVA_HOME：修改配置文件`vi /etc/profile`, 最后一行添加以下代码
    ```
      export JAVA_HOME=/usr/java/jdk1.8.0_91   
      export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar  
      export PATH=$PATH:$JAVA_HOME/bin
    ```
    * 命令：`source /etc/profile` 使文件立即生效
    * 命令：`java -version` 检测是否安装成功

### 配置
--bin 启动目录

--conf 配置文件存放配置文件:
```
--server.xml：是Mycat服务器参数调整和用户授权的配置文件。

--schema.xml：是逻辑库定义和表以及分片定义的配置文件。

--rule.xml：  是分片规则的配置文件，分片规则的具体一些参数信息单独存放为文件，也在这个目录下，配置文件修改需要重启MyCAT。

--log4j.xml： 日志存放在logs/log中，每天一个文件，日志的配置是在conf/log4j.xml中，根据自己的需要可以调整输出级别为debug                           debug级别下，会输出更多的信息，方便排查问题。

--autopartition-long.txt,partition-hash-int.txt,sequence_conf.properties， sequence_db_conf.properties 分片相关的id分片规则配置文件

--lib	    MyCAT自身的jar包或依赖的jar包的存放目录。

--logs        MyCAT日志的存放目录。日志存放在logs/log中，每天一个文件
```

逻辑库配置

配置`server.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://io.mycat/">
	<system>
          <property name="useSqlStat">0</property>  <!-- 1为开启实时统计、0为关闭 -->
          <property name="useGlobleTableCheck">0</property>  <!-- 1为开启全加班一致性检测、0为关闭 -->
          <property name="sequnceHandlerType">2</property>
          <property name="processorBufferPoolType">0</property>
          <!--分布式事务开关，0为不过滤分布式事务，1为过滤分布式事务（如果分布式事务内只涉及全局表，则不过滤），2为不过滤分布式事务,但是记录分布式事务日志-->
          <property name="handleDistributedTransactions">0</property>
          <property name="useOffHeapForMerge">1</property>
          <property name="memoryPageSize">1m</property>
          <property name="spillsFileBufferSize">1k</property>
          <property name="useStreamOutput">0</property>
          <property name="systemReserveMemorySize">384m</property>
        </system>
        <!--给schema分配权限-->
        <user name="root">
          <property name="password">123456</property>
          <!-- 这里可以配置多个schema -->
          <property name="schemas">GCCDB</property>
        </user>

        <user name="user">
          <property name="password">user</property>
          <property name="schemas">GCCDB</property>
          <property name="readOnly">true</property>
        </user>
  </mycat:server>
```

配置`schema.xml`

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
  <schema name="GCCDB" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1">    
	</schema>
  <dataNode name="dn1" dataHost="localhost1" database="gcc" />  
  <!-- balance="1" 读写分离 -->
	<dataHost name="localhost1" maxCon="1000" minCon="10" balance="1" writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
        <heartbeat>select user()</heartbeat>
        <writeHost host="hostM1" url="localhost:3306" user="root" password="">
              <readHost host="hostS2" url="192.168.125.117:3306" user="root" password="" />  
        </writeHost>  
  </dataHost>  
</mycat:schema>
```
#### balance 属性 

负载均衡类型，目前的取值有 3 种：

* balance=”0”, 不开启读写分离机制，所有读操作都发送到当前可用的 writeHost 上。
* balance=”1”，全部的 readHost 与 stand by writeHost 参与 select 语句的负载均衡，简单的说，当双主双从模式(M1->S1，M2->S2，并且 M1 与 M2 互为主备)，正常情况下，M2,S1,S2 都参与 select 语句的负载均衡。
* balance=”2”，所有读操作都随机的在 writeHost、readhost 上分发。
* balance=”3”，所有读请求随机的分发到 wiriterHost 对应的 readhost 执行，writerHost 不负担读压 
力，注意 balance=3 只在 1.4 及其以后版本有，1.3 没有。

#### writeType 属性 
write切换, 目前的取值有 2 种：

* writeType=”0”, 所有写操作发送到配置的第一个 writeHost，第一个挂了切到还生存的第二个writeHost，重新启动后以切换后的为准，切换记录在配置文件中:dnindex.properties .
* writeType=”1”，所有写操作都随机的发送到配置的 writeHost，1.5 以后废弃不推荐。

#### switchType 属性

* -1 表示不自动切换
* 1 默认值，自动切换
* 2 基于 MySQL 主从同步的状态决定是否切换 

心跳语句为 `show slave status` 或 `select user()`
#### slaveThreshold 属性

    此时意味着开启 MySQL 主从复制状态绑定的读写分离与切换机制，Mycat 心跳机
    制通过检测 show slave status 中的 "Seconds_Behind_Master", "Slave_IO_Running",
    "Slave_SQL_Running" 三个字段来确定当前主从同步的状态以及 Seconds_Behind_Master 主从复制时延，
    当 Seconds_Behind_Master>slaveThreshold 时，读写分离筛选器会过滤掉此 Slave 机器，防止读到很久之
    前的旧数据，而当主节点宕机后，切换逻辑会检查 Slave 上的 Seconds_Behind_Master 是否为 0，为 0 时则
    表示主从同步，可以安全切换，否则不会切换。

### Mysql配置
备注: 
1. 主从数据库版本最好保持一致
2. 主库负责写入操作
3. 从库负责读

主库master修改:

1. 找到my.cnf文件,在`[mysqld]`部分插入下面几行:
      ```
      log-bin=mysql-bin #开启二进制日志
      server-id=1 #设置server-id
      ```
2. 重启mysql，创建用于同步的用户账号
      ```
        #进入mysql
        mysql -u root -p
      ```
      ```mysql
        mysql> CREATE USER 'repl'@'%' IDENTIFIED BY '123456';#创建用户
        mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';#分配权限
        mysql> flush privileges;   #刷新权限
      ```
3. 查看master状态，记录二进制文件名(mysql-bin.000001)和位置(1224)：
    ```
      mysql> SHOW MASTER STATUS;
      +------------------+----------+--------------+------------------+-------------------+
      | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
      +------------------+----------+--------------+------------------+-------------------+
      | mysql-bin.000001 |     1224 |              |                  |                   |
      +------------------+----------+--------------+------------------+-------------------+
    ```

从库slave修改:
1. 修改mysql配置:

    同样找到my.cnf配置文件，添加server-id
    ```
      server-id=2 #设置server-id
    ```
2. 重启mysql，打开mysql会话，执行同步SQL语句(需要主服务器主机名，登陆凭据，二进制文件的名称和位置)：
    ```mysql
      mysql> CHANGE MASTER TO MASTER_HOST='192.168.125.97', #主库IP
      ->  MASTER_USER='repl',
      ->  MASTER_PASSWORD='123456',
      ->  MASTER_LOG_FILE='mysql-bin.000001', #与主库的File值对应
      ->  MASTER_LOG_POS=1224; #与主库的Position值对应
    ```
3. 启动slave同步进程：
    ```
      mysql> start slave;
    ```
4. 查看slave状态：
    ```
      mysql> show slave status\G;
          *************************** 1. row ***************************
                  Slave_IO_State: Waiting for master to send event
                      Master_Host: 192.168.125.97
                      Master_User: repl
                      Master_Port: 3306
                    Connect_Retry: 60
                  Master_Log_File: mysql-bin.000001
              Read_Master_Log_Pos: 1224
                  Relay_Log_File: iMac-4-relay-bin.000001
                    Relay_Log_Pos: 320
            Relay_Master_Log_File: mysql-bin.000001
                Slave_IO_Running: Yes
                Slave_SQL_Running: Yes
                ...............................  
        1 row in set (0.02 sec)

        ERROR:
        No query specified
    ```

    看到最后

    ```
      Slave_IO_Running: Yes
      Slave_SQL_Running: Yes
    ```

    这两个值都为`Yes`, 则配置成功


### linux：
```
./mycat start 启动

./mycat stop 停止

./mycat console 前台运行

./mycat install 添加到系统自动启动（暂未实现）

./mycat remove 取消随系统自动启动（暂未实现）

./mycat restart 重启服务

./mycat pause 暂停

./mycat status 查看启动状态
```

### 测试主从读写分离是否成功
查看是否启动
```
mysql -h192.168.0.1 -P8806 -urepl -p123456 #-h localhost
```
进入mysql,通过写查询或者插入语句，观察数据是否同步