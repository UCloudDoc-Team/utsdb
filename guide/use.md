# InfluxDB使用指南

## 命令行

命令行界面（influx）是与HTTP API进行交互的shell。可以使用influx写入数据、查询数据，及用不同的格式查看查询结果。

### 下载CLI

进入到[页面下载](https://portal.influxdata.com/downloads/)

### 启动influx

要使用CLI，首先在终端启动influx，可通过ip成功连接到InfluxDB实例：
```
[root@10-10-XXX-XXX~]# ./influx -host 10.10.XXX.XXX
Connected to http://10.10.5.129:8086 version XXX
InfluxDB shell version: XXX
Enter an InfluxQL query  
```

##  HTTP

本文将介绍使用HTTP接口如何读写数据；本文中以控制台创建的时序数据实例（ip为10.10.5.129）为例。

### 写入数据

有很多可以向InfluxDB写数据的方式，包括命令行、客户端还有一些像Graphite有一样数据格式的插件。

#### 创建数据库

请使用UCloud 控制台创建数据库，具体操作见快速上手中的[实例管理](/quick/instance.md)。

#### 使用HTTP接口写数据

我们往InfluxDB写数据的主要方式是通过HTTP接口POST数据到/write路径；以下举例，写了一条数据到mydb数据库；该数据的组成部分是measurement为'cpu_load_short'，tag的key为host和region，对应tag的value是'server01'和'us-west'，field的key是'value'，对应数值为'0.64'，而时间戳是'1434055562000000000'。

```
curl -i -XPOST 'http://10.10.5.129:8086/write?db=mydb' --data-binary 'cpu,host=server01,region=us-west value=0.64 1434055562000000000'
```

![image](/images/influxdb0002.png)

当写入这条数据点的时候，你必须明确存在一个数据库对应名字是'db'参数的值。如果你没有通过'rp'参数设置retention policy的话，那么这个数据会写到'db'默认的retention policy中。

称POST的请求体为'Line Protocol'，包含了用户希望存储的时间序列数据。其组成部分有measurement，tags，fields和timestamp。其中，measurement是InfluxDB必须的，tags是可选的，但是对于大部分数据都会包含tags用来区分数据的来源，让查询变得容易和高效；tag的key和value都必须是字符串；fields的key也是必须的，且是字符串，默认情况下field的value是float类型。timestamp在这个请求行的最后，是一个纳秒级的Unix time，且是可选的，如果不传，InfluxDB会使用服务器的本地的纳米级的timestamp来作为数据的时间戳，注意无论哪种方式，在InfluxDB中的timestamp只能是UTC时间。

#### 同时写入多个点

要想同时发送多个数据点到多个series(在InfluxDB中measurement加tags组成了一个series)，可以用新的行来分开这些数据点。这种批量发送的方式可以获得更高的性能。

下面的例子就是写了三个数据点到'mydb'数据库中：

第一个点所属series的measurement为'cpu_load_short'，tag是'host=server02'，timestamp是server本地的时间戳；

第二个点同样是measurement为'cpu_load_short'，但是tag为'host=server02,region=us-west',且有明确timestamp为'1422568543702900257'的series；

第三个数据点和第二个的timestamp是一样的，但是series不一样，其measurement为'cpu_load_short'，tag为'direction=in,host=server01,region=us-west'。

```
curl -i -XPOST 'http://10.10.5.129:8086/write?db=mydb' --data-binary 'cpu,host=server02 value=0.67
cpu,host=server02,region=us-west value=0.55 1422568543702900257
cpu,direction=in,host=server01,region=us-west value=2.0 1422568543702900257'
```

![image](/images/influxdb0002.png)

#### 无模式设计

InfluxDB是一个无模式(schemaless)的数据库，你可以在任意时间添加measurement，tags和fields。注意：如果你试图写入一个和之前的类型不一样的数据(例如，filed字段之前接收的是数字类型，现在写了个字符串进去)，那么InfluxDB会拒绝这个数据。

#### HTTP返回值概要

- 2xx：如果你写了数据后收到'HTTP 204 No Content'，说明写入成功。
- 4xx：表示InfluxDB无法处理该请求；无法鉴别请求发送的是什么，可能是查询的语法不正确引起的。
- 5xx：无法处理该请求，可能是系统过载或是应用受损。

### 查询数据

#### 使用HTTP接口查询数据

通过HTTP接口查询数据，是InfluxDB查询数据的主要方式。通过发送一个GET请求到'/query'路径，并设置URL的'db'参数为目标数据库，设置URL参数'q'为查询语句。例如：

```
curl -G 'http://10.10.5.129:8086/query?pretty=true' --data-urlencode "db=influxdbmydb" --data-urlencode "q=SELECT \"value\" FROM \"cpu_load_short\" WHERE \"region\"='us-west';SELECT count(\"value\") FROM \"cpu_load_short\" WHERE \"region\"='us-west'"
```

InfluxDB返回一个json值，你查询的结果在'result'列表中，如果有错误发送，InfluxDB会在'error'这个key里解释错误发生的原因。

```
{
    "results": [
        {
            "statement_id": 0
        },
        {
            "statement_id": 1
        }
    ]
}
```

>备注：其中，添加'pretty=ture'参数在URL里面，是为了让返回的json格式化。这在调试或者是直接用'curl'的时候很有用，但在生产上不建议使用，因为这样会消耗不必要的网络带宽。

#### 多个查询

在一次调用中发送多个InfluxDB的查询语句，可以简单地使用分号分隔每个查询。

#### 查询数据时的其他可选参数

#### 时间戳格式

在InfluxDB中的所有数据都是存的UTC时间，时间戳默认返回RFC3339格式的纳米级的UTC时间，例如'2019-11-04T19:05:14.318570484Z'，如果你想要返回Unix格式的时间，可以在请求参数里设置'epoch'参数，其中epoch可以是'[h,m,s,ms,u,ns]'之一。例如返回一个秒级的epoch：

```
curl -G 'http://10.10.5.129:8086/query' --data-urlencode "db=influxdbmydb" --data-urlencode "epoch=s" --data-urlencode "q=SELECT \"value\" FROM \"cpu_load_short\" WHERE \"region\"='us-west'"
```
更多使用指南可参考[InfluxDB官方操作文档](https://docs.influxdata.com/influxdb/v1.7/)
