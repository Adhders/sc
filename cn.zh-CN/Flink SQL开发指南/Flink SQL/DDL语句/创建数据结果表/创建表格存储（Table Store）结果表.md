# 创建表格存储（Table Store）结果表 {#concept_62526_zh .concept}

本文为您介绍如何创建实时计算表格存储（Table Store）结果表以及表格存储和实时计算字段类型之间的映射关系。

## 什么是表格存储（Table Store） {#section_i4b_jrf_cgb .section}

表格存储（Table Store），简称OTS，是构建在阿里云飞天分布式系统之上的分布式NoSQL数据存储服务。表格存储通过数据分片和负载均衡技术，实现数据规模与访问并发上的无缝扩展，提供海量结构化数据的存储和实时访问服务。

## DDL定义 {#section_pwl_qrf_cgb .section}

实时计算支持使用TableStore作为结果输出，示例代码如下。

```language-sql
CREATE TABLE stream_test_hotline_agent (
 name varchar,
 age BIGINT,
 birthday BIGINT,
 PRIMARY KEY (name,age)
) WITH (
 type='ots',
 instanceName='******',
 tableName='******',
 accessId='******',
 accessKey='******',
 endPoint='******',
 valueColumns='******'
);

```

**说明：** 

-   推荐使用数据存储注册功能，参见表格存储[注册表格存储（TableStore）](cn.zh-CN/Flink SQL开发指南/数据存储/注册数据存储/注册表格存储（TableStore）.md#)。
-   valueColumns的值不能是声明的主键，可以是主键之外的任意字段。

## WITH参数 {#section_jsj_rrf_cgb .section}

|参数|说明|备注|
|--|--|--|
|instanceName|实例名|无|
|tableName|表名|无|
|endPoint|实例访问地址|参见 [服务地址](../../../../../cn.zh-CN/产品简介/名词解释/服务地址.md#) 。|
|accessId|访问的id|无|
|accessKey|访问的键|无|
|valueColumns|指定插入的字段列名，多个字段以`,`分割。|插入2个字段的输入为`'ID,NAME'`。|
|bufferSize|去重后的buffer大小|可选，默认值5000，表示输入的数据达到5000条就开始输出。|
|batchWriteTimeoutMs|写入超时时间|可选，单位为毫秒，默认值为5000。表示如果缓存中的数据在等待5秒后，依然没有达到输出条件，系统会自动输出缓存中的所有数据。|
|batchSize|一次批量写入的条数|可选，默认值100。|
|retryIntervalMs|重试间隔时间|可选，单位毫秒，默认值1000。|
|maxRetryTimes|最大重试次数|可选，默认值100。|
|ignoreDelete|是否忽略delete操作|默认为false。|

## 类型映射 {#section_z2d_srf_cgb .section}

|OTS字段类型|实时计算字段类型|
|-------|--------|
|INTEGER|BIGINT|
|STRING|VARCHAR|
|BOOLEAN|BOOLEAN|
|DOUBLE|DOUBLE|

**说明：** TableStore结果表须定义有`primary key`，输出数据以[Update方式](cn.zh-CN/Flink SQL开发指南/Flink SQL/DDL语句/创建数据结果表/数据结果表概述.md#ul_tfk_5m2_cgb)写入到现有TableStore表。

