# 创建日志服务（Log Service）源表 {#concept_62521_zh .concept}

本文为您介绍如何为实时计算创建日志服务（Log Service）源表以及创建过程涉及到的属性字段、WITH参数和类型映射。

## 什么是日志服务 {#section_qr1_2wy_bgb .section}

日志服务（Log Service）是针对日志类数据的一站式服务，在阿里巴巴集团经历大量大数据场景锤炼而成。 日志服务本身是流数据存储，实时计算能将其作为流式数据输入。对于日志服务而言，数据格式类似JSON，示例如下。

```language-json
{
	"a": 1000,
	"b": 1234,
	"c": "li"
}

```

对于实时计算而言，我们需要定义的DDL如下（代码中的sls代表Log Service）。

```language-sql
create table sls_stream(
  a int,
  b int,
  c VARCHAR
) with (
  type ='sls',  
  endPoint ='******',
  accessId ='******',
  accessKey ='******',
  startTime = '******',
  project ='******',
  logStore ='******',
  consumerGroup ='******'
);

```

## 属性字段 {#section_xhx_xxy_bgb .section}

目前Flink SQL默认支持3个Log Servic属性字段的获取，也支持其它自定义字段的写入。

|字段名|注释说明|
|---|----|
| `__source__` |消息源|
| `__topic__` |消息主题|
| `__timestamp__` |日志时间|

属性字段使用说明

为了获取这些属性字段，除了正常逻辑声明外，还需要在类型声明后加上`HEADER`关键字。示例如下。

-   测试数据

    ```
         __topic__:  ens_altar_flow  
            result:  {"MsgID":"ems0a","Version":"0.0.1"}
    
    ```

-   测试语句

    ```language-sql
    CREATE TABLE sls_log (
      __topic__  VARCHAR HEADER,
      result     VARCHAR  
    )
    WITH
    (
      type ='sls'
    );
    
    CREATE TABLE sls_out (
      name     VARCHAR,
      MsgID    VARCHAR,
      Version  VARCHAR 
    )
    WITH
    (
      type ='RDS'
    );
    
    INSERT INTO sls_out
    SELECT 
    __topic__,
    JSON_VALUE(result,'$.MsgID'),
    JSON_VALUE(result,'$.Version')
    FROM
    sls_log
    
    ```

-   测试结果

    |name\(VARCAHR\)|MsgID\(VARCAHR\)|Version\(VARCAHR\)|
    |---------------|----------------|------------------|
    |ens\_altar\_flow|ems0a|0.0.1|


## WITH参数 {#section_uz1_zxy_bgb .section}

|参数|注释说明|备注|
|--|----|--|
|endPoint|消费端点信息| [服务入口](../../../../../cn.zh-CN/API 参考/服务入口.md#)|
|accessId|sls读取的accessKey|无|
|accessKey|sls读取的密钥|无|
|project|读取的sls项目|无|
|logStore|project下的具体的LogStore名称|无|
|consumerGroup|消费组名|用户可以自定义消费组名（没有固定格式）|
|startTime|消费日志开始的时间点|无|
|heartBeatIntervalMills|可选，消费客户端心跳间隔时间|默认为10s|
|maxRetryTimes|读取最大尝试次数|可选，默认为5。|
|batchGetSize|单次读取logGroup条数|可选，默认为10。|
|lengthCheck|单行字段条数检查策略|可选，默认为SKIP,其它可选值为EXCEPTION、PAD。-   SKIP：字段数目不符合时跳过 。
-   EXCEPTION:字段数目不符合时抛出异常。
-   PAD:按顺序填充，不存在的置为null。

|
|columnErrorDebug|是否打开调试开关，如果打开，会把解析异常的log打印出来|可选，默认为false。|

**说明：** 

-   sls暂不支持MAP类型的数据。
-   字段顺序支持无序（建议字段顺序和表中定义一致）。
-   输入数据源为JSON形式时，注意定义分隔符，并且需要采用内置函数[JSON\_VALUE](cn.zh-CN/使用指南/Flink SQL/内置函数/字符串函数/JSON_VALUE.md#)分析，否则就会解析失败。报错如下:

    ```
    2017-12-25 15:24:43,467 WARN [Topology-0 (1/1)] com.alibaba.blink.streaming.connectors.common.source.parse.DefaultSourceCollector - Field missing error, table column number: 3, data column number: 3, data filed number: 1, data: [{"lg_order_code":"LP00000005","activity_code":"TEST_CODE1","occur_time":"2017-12-10 00:00:01"}]
    
    ```

-   batchGetSize设置不能超过1000，否则会报错。
-   batchGetSize指明的是logGroup获取的数量，如果单条logItem的大小和batchGetSize都很大，有可能会导致频繁的垃圾回收（Garbage Collection\)，这种情况下该参数应调小。

## 类型映射 {#section_zgx_zxy_bgb .section}

日志服务和实时计算字段类型对应关系如下。建议您使用该对应关系进行DDL声明:

|日志服务字段类型|实时计算字段类型|
|--------|--------|
|STRING|VARCHAR|

