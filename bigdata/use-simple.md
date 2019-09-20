# 基本数据平台第一期

## 基础架构设计

Apache Kafka + Apache Flink + Apache Druid + Apache Superset + Hbase

## 业务消息构造
原始消息（日志）需要满足以下属性：
![image](https://note.youdao.com/yws/res/17834/0243CFA8E7AE4EFA8221349CE2AA1286)

实时计算业务展现：

商户以及客户
1. 近1小时、6小时、24小时实时热点商品TopN（浏览、点击、购买、加购物车、分享）；
2. 近1小时、6小时、24小时实时访问用户画像分析；
3. 基于近三天的浏览、点击、收藏、分享的商品，推送客户感兴趣的商品。

平台：
1. 近10分钟、30分钟、60分钟的热门小程序排行；
2. 近5分钟、10分钟、30分钟、60分钟的流量分析；
3. 通过Nginx日志实时分析请求情况。

## Kafka
安装：略

### 基础使用
新建Topic：applet-test
```
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic applet-test
```
查询topic列表：

```
bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```
客户端启动producer测试：

```
root@node5:/mnt/kafka/kafka_2.11-2.3.0# bin/kafka-console-producer.sh --broker-list localhost:9092 --topic applet-test
>hello world
>heihei


```
客户端启动consumer测试：

```
root@node5:/mnt/kafka/kafka_2.11-2.3.0# bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic applet-test --from-beginning
hello world
heihei

```

## Apache Flink

## 运行任务

```
bin/flink run -c com.weijuju.flink.applet.job.AppletBaseStreamingJob /mnt/flink/trainData/applet-flink-0.0.1-SNAPSHOT.jar 127.0.0.1 9000
```


## OLAP架构：Apache Druid
tutorial:https://druid.apache.org/docs/latest/operations/getting-started.html

### 配置参考
https://druid.apache.org/docs/latest/configuration/index.html#jvm-configuration-best-practices
### 运行注意点
#### zookeeper
1. 不需要自带的zookeeper，需要连接到现有zookeeper集群
修改apache-druid-0.15.0-incubating/conf/druid/single-server/micro-quickstart/_common/common.runtime.properties
```
#
# Zookeeper
#

druid.zk.service.host=localhost:3148,localhost:3248,localhost:3348
druid.zk.paths.base=/druid
```
然后注释启动脚本：
/conf/supervise/single-server/micro-quickstart.conf

```
#!p10 zk bin/run-zk conf

```
如果跟flink部署在同一台机器，需要修改端口号：
apache-druid-0.15.0-incubating/conf/druid/single-server/micro-quickstart/coordinator-overlord，将8081换为18081

同时将apache-druid-0.15.0-incubating/bin/verify-default-ports端口验证脚本的端口变更


Spec:http://druid.apache.org/docs/latest/design/

### Druid遇到的问题

#### UTC时区问题导致导入数据的__time不对

#### query只能查询5000条数据
其实问题很简单，主要在于集群工作时区与导入数据时区不一致。由于Druid是时间序列数据库，所以对时间非常敏感。Druid底层采用绝对毫秒数存储时间，如果不指定时区，默认输出为零时区时间，即ISO8601中yyyy-MM-ddThh:mm:ss.SSSZ。我们生产环境中采用东八区，也就是Asia/Hong Kong时区，所以需要将集群所有UTC时间调整为UTC+08:00；同时导入的数据的timestamp列格式必须为：yyyy-MM-ddThh:mm:ss.SSS+08:00


数据摄取规范：
https://druid.apache.org/docs/latest/ingestion/ingestion-spec.html
```
{
  "type": "kafka",
  "dataSchema": {
    "dataSource": "applet-item-click-test",
    "parser": {
      "type": "string",
      "parseSpec": {
        "format": "json",
        "timestampSpec": {
          "column": "windowEnd",
          "format": "yyyy-MM-ddThh:mm:ss.SSS+08:00"
        },
        "dimensionsSpec": {
          "dimensions": [
            "uid",
            "objId",
            { "name": "platform", "type": "int" },
            { "name": "sourceType", "type": "int" },
            { "name": "objType", "type": "int" },
            { "name": "clickCount", "type": "long" },
            { "name": "windowStart", "type": "long" },
            { "name": "windowEnd", "type": "long" }
          ]
        }
      }
    },
    "metricsSpec" : [],
    "granularitySpec": {
      "type": "uniform",
      "segmentGranularity": "DAY",
      "queryGranularity": "NONE",
      "rollup": false
    }
  },
  "tuningConfig": {
    "type": "kafka",
    "reportParseExceptions": false
  },
  "ioConfig": {
    "topic": "applet-item-click-test",
    "replicas": 2,
    "taskDuration": "PT10M",
    "completionTimeout": "PT20M",
    "consumerProperties": {
      "bootstrap.servers": "192.168.100.248:9092"
    }
  }
}
```


## Druid数据展示：Apache Superset
tutorial：http://superset.apache.org/tutorial.html

spec:http://superset.apache.org

### 安装

官方安装文档：https://superset.incubator.apache.org/installation.html

友好安装博客：https://www.cnblogs.com/tonglin0325/p/11189979.html

当前版本需保证有python3.6以上的环境

1. 对于Debian和Ubuntu，以下命令将确保安装所需的依赖项：
```
sudo apt-get install build-essential libssl-dev libffi-dev python-dev python-pip libsasl2-dev libldap2-dev
```

2. 建议在virtualenv中安装Superset。 Python 3已经发布了virtualenv。 但是，如果由于某种原因没有在您的环境中安装它，您可以通过您的操作系统的软件包安装它，否则您可以从pip安装：

```
pip install virtualenv -i http://mirrors.aliyun.com/pypi/simple --trusted-host mirrors.aliyun.com
```

3. 您可以通过以下方式创建和激活virtualenv：

先通过which python3.6查询python3.6的安装位置
```
//创建virtualenv环境，指定环境名为venv3.6
virtualenv -p /usr/bin/python3.6 venv3.6
//激活创建的环境
. venv3.6/bin/activate
```

```

```

4. 通过获取最新的pip和setuptools库，将所有机会放在一边：

```
pip install --upgrade setuptools pip
```
5. 按照以下几个简单步骤安装Superset：

- 安装superset，使用阿里云的源速度更快
```
pip install superset**0.29.0rc7 -i http://mirrors.aliyun.com/pypi/simple --trusted-host mirrors.aliyun.com
```
如果遇到错误：Building wheel for python-geohash (setup.py) ... error，执行

```
sudo apt-get install python3.6-dev libsasl2-dev gcc
```
- 初始化数据库
```
superset db upgrade
```
如果初始化数据库出现错误：ImportError: cannot import name '_maybe_box_datetimelike'，是pandas版本过高导致的,进行降级

```
pip list | grep pandas

pip install pandas==0.23.4 -i http://mirrors.aliyun.com/pypi/simple --trusted-host mirrors.aliyun.com
```
如果继续出现错误类似：
```
Traceback (most recent call last):
  File "/root/venv3.6/bin/superset", line 15, in <module>
    cli()
  File "/root/venv3.6/lib/python3.6/site-packages/click/core.py",
```
安装sqlalchemy（版本不对卸载重新安装：pip uninstall SQLAlchemy）：
```
pip install sqlalchemy==1.2.18 -i http://mirrors.aliyun.com/pypi/simple --trusted-host mirrors.aliyun.com
```
- 创建管理员用户（admin/admin,使用邮箱liujie@365huaer.com）

```
fabmanager create-admin --app superset
```

- 加载样例数据

```
superset load_examples
```


- 初始化默认角色和权限

```
superset init
```


- 使用gunicorn运行

```
//前台运行
gunicorn -w 2 --timeout 60 -b  0.0.0.0:8022 --limit-request-line 0 --limit-request-field_size 0 superset:app
//后台运行
gunicorn -D -w 2 --timeout 60 -b  0.0.0.0:8022 --limit-request-line 0 --limit-request-field_size 0 superset:app

//关闭进程
pstree -ap|grep gunicorn
killall -9 gunicorn
```
访问地址：http://192.168.100.247:8022/

输入用户名密码（admin/admin）登录

![image](https://note.youdao.com/yws/res/17835/4B6D0D8A45FA4F79A876BB09BD3BC94E)

### 连接Druid数据源

![image](https://note.youdao.com/yws/res/17830/27404C29DE2A4B20BE94CACAD904468D)

![image](https://note.youdao.com/yws/res/17831/1E01EF82F7A34642BBA7AB8C73858741)

![image](https://note.youdao.com/yws/res/17833/A21C5AE3633D49F7906BB5A79E864FC2)

![image](https://note.youdao.com/yws/res/17829/26832C76D036401582E95D1FBAC7A883)

点击Charts菜单出现：

```
Sorry, something went wrong
500 - Internal Server Error
Stacktrace
         Traceback (most recent call last):
  File "/root/venv3.6/lib/python3.6/site-packages/flask/app.py", line 2446, in wsgi_app
    response = self.full_dispatch_request()
  File "/root/venv3.6/lib/python3.6/site-packages/flask/app.py", line 1951, in full_dispatch_request
    rv = self.handle_user_exception(e)
  File "/root/venv3.6/lib/python3.6/site-packages/flask/app.py", lin
  。。。
    File "/root/venv3.6/lib/python3.6/site-packages/superset/connectors/druid/models.py", line 533, in link
    return Markup('<a href="{self.url}">{name}</a>').format(**locals())
TypeError: format() got multiple values for argument 'self'
```
对此，引用githup上的问题修复：https://github.com/apache/incubator-superset/issues/6347

![image](https://note.youdao.com/yws/res/17836/9FC01FAA75184CF8BA8235F6881BCB32)

重新启动superset后修复：

![image](https://note.youdao.com/yws/res/17832/B57C14AC996F4EEA98D8881BAF94C2DC)