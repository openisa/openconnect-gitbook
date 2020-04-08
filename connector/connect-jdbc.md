# Connect-Jdbc



## Prerequisites要求
This tutorial runs on Ubuntu server 16.04 LTS

- follow the file-connector tutorial
- install docker


### 项目下载

之前应该已经下载过rocketmq-externals, rocketmq-connect-jdbc也在rocketmq-externals中，

```bash
$ cd ~/rocketmq-externals/rocketmq-connect-jdbc
```

### 构建jar包

```
mvn clean install -Dmaven.test.skip=true
```
### 将jar包放入runtime的plugin目录下
runtime的plugin目录应该在connect.conf里设置过，推荐设置是`/usr/local/connector-plugins/`

```
sudo cp target/rocketmq-connect-jdbc-0.0.1-SNAPSHOT-jar-with-dependencies.jar /usr/local/connector-plugins/
```
请注意一定要用这个后缀为`with-dependencies.jar`的，否则会在运行时报`NoClassDefFoundError`.


### 启动source connector的mysql实例和sink connector的mysql实例
我们建议使用docker安装mysql 5.7来作为jdbc连接的对象

```
sudo apt install docker-io #
cd ~
mkdir source-mysql # 使用host的disk作为mysql容器的持久化目录
mkdir sink-mysql

# 指定将host上3305端口map到source-mysql容器3306端口
sudo docker run -d --name source-mysql -v ~/source-mysql:/var/lib/mysql -p 3305:3306 -e MYSQL_ROOT_PASSWORD=root
# 指定将host上3307端口map到sink-mysql容器3306端口
sudo docker run -d --name sink-mysql -v ~/sink-mysql:/var/lib/mysql -p 3307:3306 -e MYSQL_ROOT_PASSWORD=root

```
我们source-mysql和sink-mysql的root password都设置成了root，这肯定是不安全的，您可以自行修改。
如果您在重新启动机器后，可以使用如下命令重启两个mysql容器

```
sudo docker start source-mysql
sudo docker start sink-mysql
```

### 在source-mysql和sink-mysql中创建对应的表项
注意，rocketmq-connect-jdbc目前要求database，table都已经在mysql实例中被创建过了，如果sink-connector连接到的mysql实例中没有对应的数据库表，那么将会报错。

登入source-mysql实例
```
mysql -uroot -proot -h127.0.0.1 -P3305
```

在mysql命令行界面中
```
create database jdbc;

USE jdbc;

CREATE TABLE `person` (
  `age` INT NOT NULL,
  `salary` INT NOT NULL,
PRIMARY KEY(`age`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO person VALUES(10, 1000);
```
很显然age不应该是这个表的primary key，但是为了简单起见，我们暂时委屈一下。

然后在sink-mysql中创建表，但不插入数据

登入sink-mysql实例
```
mysql -uroot -proot -h127.0.0.1 -P3307
```

在mysql命令行界面中
```
create database jdbc;

USE jdbc;

CREATE TABLE `person` (
  `age` INT NOT NULL,
  `salary` INT NOT NULL,
PRIMARY KEY(`age`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

```
### 在rocketmq中创建被jdbc connector使用的topic

查找broker cluster的地址
```
sh ~/rocketmq/distribution/target/rocketmq-4.7.0/rocketmq-4.7.0/bin/mqadmin clusterList -n 127.0.0.1:9876
```
output如下
```
#Cluster Name     #Broker Name            #BID  #Addr                  #Version                #InTPS(LOAD)       #OutTPS(LOAD) #PCWait(ms) #Hour #SPACE
DefaultCluster    affe2                   0     172.17.0.1:10911       V4_7_0                   1.20(0,0ms)         0.80(0,0ms)          0 440637.81 -1.0000
```
这个#Addr ：172.17.0.1:10911 以及 `DefaultCluster` 将会在之后的步骤中用到

和file connector一样
```
sh ~/rocketmq/distribution/target/rocketmq-4.7.0/rocketmq-4.7.0/bin/mqadmin updateTopic -c DefaultCluster -t jdbcTopic -n 127.0.0.1:9876 mqadmin updateTopic -b 172.17.0.1:10911 -t fileTopic
```
注意这里`-b` 选项后就是上一步rocektmq broker cluster的地址。

### 编写JdbcSourceConnector的config file

因为目前想要在rocketmq runtime里启动connector，最好的办法是通过RESTful API来发送配置信息，为了方便我们将配置信息放在～/config/jdbc_source_connector.json中。

示例文件如下
```
{
    "connector-class":"org.apache.rocketmq.connect.jdbc.connector.JdbcSourceConnector",
    "rocketmqTopic":"jdbcTopic", 
    "dbUrl":"127.0.0.1",    
    "dbPort":"3305",        
    "dbUsername":"root",    
    "dbPassword":"root",
    "whiteDataBase": {
        "jdbc":{
           "person": {"NO-FILTER": "10"}
         }
      },
    "mode": "bulk",
    "task-parallelism": 1,
    "source-cluster": "172.17.0.1:10911",
    "source-rocketmq": "127.0.0.1:9876",
    "source-record-converter":"org.apache.rocketmq.connect.runtime.converter.JsonConverter"
 }
```
详细的参数设置可以参见[rocketmq-connect-jdbc](https://github.com/apache/rocketmq-externals/tree/master/rocketmq-connect-jdbc)的
readme page，这里对一些可能有问题的参数进行解释。
- rocketmqTopic 设置为我们刚创建的 `jdbcTopic`
- whiteDataBase 指定了需要被同步的数据库与表，'NO-FILTER' 代表将`person`表中所有记录同步到sink-mysql中
- source-cluster: 填rocketmq broker cluster的地址
- source-rocketmq： 填rocektmq nameserver的地址



### 编写JdbcSinkConnector的config file
示例文件如下,存放在～/config/jdbc_sink_connector.json
```
{
    "connector-class":"org.apache.rocketmq.connect.jdbc.connector.JdbcSinkConnector",
    "topicNames":"person",
    "rocketmqTopic":"jdbcTopic",
    "dbUrl":"127.0.0.1",
    "dbPort":"3307",
    "dbUsername":"root",
    "dbPassword":"root",
    "source-rocketmq": "127.0.0.1:9876",
    "source-cluster": "172.17.0.1:10911",
    "mode": "bulk",
    "source-record-converter":"org.apache.rocketmq.connect.runtime.converter.JsonConverter"
 }

```
唯一要注意的点是，这里的的`topicNames`必须要和数据库里table的name一致，这里我们需要同步`person` 表，topicNames就应该填`person`




### 提交connector config到runtime

得到urlencode之后的config
```
jdbc_source_config=$(urlencode `cat ~/config/jdbc_source_connector.json`)
jdbc_sink_config=$(urlencode `cat ~/config/jdbc_sink_connector.json`)
```
使用curl命令启动JdbcSourceConnector 以及 JdbcSinkConnector
```
curl -i -H "Accept: application/json" "http://localhost:8081/connectors/jdbc-source-connector?config=$jdbc_source_config"
echo "\n"
curl -i -H "Accept: application/json" "http://localhost:8081/connectors/jdbc-sink-connector?config=$jdbc_sink_config"
echo "\n"

```

### 验证
登入sink-mysql实例
```
mysql -uroot -proot -h127.0.0.1 -P3307
```

验证
```
USE jdbc;
SELECT * FROM person;
```
应该有如下输出
```
+-----+--------+
| age | salary |
+-----+--------+
|  10 |   1000 |
+-----+--------+
1 row in set (0.00 sec)
```