# 创建消息队列（Kafka）源表 {#concept_86824_zh .concept}

本文为您介绍如何创建实时计算消息队列（Kafka）源表以及Kafka版本对应关系和Kafka消息解析示例。

**说明：** 本文档只适用于独享模式。

## 什么是Kafka源表 {#section_mqr_zmz_bgb .section}

Kafka源表的实现迁移自社区的kafka版本实现。Kafka源表数据解析流程：Kafka Source Table -\> UDTF -\>Realtime Compute -\> SINK。从Kakfa读入的数据，都是VARBINARY（二进制）格式，对读入的每条数据，都需要用UDTF将其解析成格式化数据。

## DDL定义 {#section_zyh_dnz_bgb .section}

Kafka源表定义DDL必须与以下SQL一摸一样，表中的五个字段顺序务必保持一致：

```language-sql
-- 定义源表，注意：kafka源表DDL字段必须与以下例子一模一样。WITH中参数可改。
create table kafka_stream(
  messageKey VARBINARY,
  `message`    VARBINARY,
  topic      VARCHAR,
  `partition`  INT,
  `offset`     BIGINT
) with (
  type ='kafka010',
  topic = 'xxx',
  `group.id` = 'xxxxxx',
  ...
);

```

## WITH参数 {#section_ivk_14z_bgb .section}

-   通用配置

    |参数|注释说明|备注|
    |--|----|--|
    |type|kafka对应版本|必选，必须是 Kafka08、Kafka09、Kafka010或Kafka011，版本对应关系见表格下方。|
    |topic|读取的单个topic|无|
    |topicPattern|读取一批topic的表达式|无|
    |startupMode|启动位点|     -   EARLISET：从kafka最早分区开始读取。
    -   Group\_OFFSETS: 根据Group读取。
    -   LATEST: 从kafka最新位点开始读取。
    -   TIMESTAMP:从指定的时间点读取。\(Kafka010、Kafka011支持。\)
 |
    |partitionDiscoveryIntervalMS|定时检查是否有新分区产生|默认值为60000,即1分钟。|
    |extraConfig|额外的kafkaConsumer配置项目|可选，未在可选配置项中，但是额外期望的配置。|

-   kafka08必选配置

    |参数|注释说明|备注|
    |--|----|--|
    |group.id|组名|消费组id|
    |zookeeper.connect|zk链接地址|zk连接id|

-   可选配置key `"consumer.id","socket.timeout.ms","fetch.message.max.bytes","num.consumer.fetchers","auto.commit.enable","auto.commit.interval.ms","queued.max.message.chunks", "rebalance.max.retries","fetch.min.bytes","fetch.wait.max.ms","rebalance.backoff.ms","refresh.leader.backoff.ms","auto.offset.reset","consumer.timeout.ms","exclude.internal.topics","partition.assignment.strategy","client.id","zookeeper.session.timeout.ms","zookeeper.connection.timeout.ms","zookeeper.sync.time.ms","offsets.storage","offsets.channel.backoff.ms","offsets.channel.socket.timeout.ms","offsets.commit.max.retries","dual.commit.enabled","partition.assignment.strategy","socket.receive.buffer.bytes","fetch.min.bytes"` 
-   kafka09/kafka010/kafka011必选配置

    |参数|注释说明|备注|
    |--|----|--|
    |group.id|组名|消费组id|
    |bootstrap.servers|Kafka集群地址|Kafka集群地址|

    其它可选配置项参考Kafka官方文档进行配置

    -    [Kafka09](https://kafka.apache.org/0110/documentation.html#consumerconfigs) 
    -   [Kafka010](https://kafka.apache.org/090/documentation.html#newconsumerconfigs) 
    -   [Kafka011](https://kafka.apache.org/0102/documentation.html#newconsumerconfigs) 
    当需要配置某选项时，在DDL中的with部分增加对应的参数即可，例如配置SASL登录增加如下三个参数:

    ```
    create table kafka_stream(
      messageKey varbinary,
      `message` varbinary,
      topic varchar,
      `partition`int,
      `offset`bigint
    ) with (
      type ='kafka010',
      topic = 'xxx',
      `group.id` = 'xxxxxx',
      ...,
      `security.protocol`=SASL_PLAINTEXT,
      `sasl.mechanism`=PLAIN,
      `sasl.jaas.config`='org.apache.kafka.common.security.plain.PlainLoginModule required username="USERNAME" password="PASSWORD";'--其中USERNAME和PASSWORD填写真实的用户名和密码
    );
    ```


## Kafka版本对应关系 {#section_o4c_b4z_bgb .section}

|type|Kafka版本|
|----|-------|
|Kafka08|0.8.22|
|Kafka09|0.9.0.1|
|Kafka010|0.10.2.1|
|Kafka011|0.11.0.2|

## Kafka消息解析示例 {#section_ycd_g4z_bgb .section}

-   示例一
    -   场景

        要将Kafka中的数据进行计算，并将计算结果输出到RDS。 Kafka中保存了JSON格式数据，需要用Realtime Compute进行计算，消息格式为：

        ```language-json
        {
          "name":"Alice",
          "age":13,
          "grade":"A"
        }
        
        ```

        整个计算流程为：Kafka SOURCE-\>UDTF-\>Realtime Compute-\>RDS SINK

    -   示例代码
        -   SQL

            ```language-sql
            -- 定义解析Kakfa message的UDTF
            CREATE FUNCTION kafkapaser AS 'com.alibaba.kafkaUDTF';
            
            -- 定义源表，注意：kafka源表DDL字段必须与以下例子一模一样。WITH中参数可改。
            create table kafka_src (
                messageKey  VARBINARY,
                `message`   VARBINARY,
                topic       VARCHAR,
                `partition` INT,
                `offset`    BIGINT
            ) WITH (
                type = 'kafka010',    -- Kafka Source类型，与Kafka版本强相关，目前支持的Kafka版本请参考本文档
                topic = 'test_kafka_topic',
                `group.id` = 'test_kafka_consumer_group',
                bootstrap.servers = 'ip1:port1,ip2:port2,ip3:port3'
            );
            create table rds_sink (
              name       VARCHAR,
              age        INT,
              grade      VARCHAR,
              updateTime TIMESTAMP
            ) WITH(
             type='rds',
             url='jdbc:mysql://localhost:3306/test',
             tableName='test4',
             userName='test',
             password='XXXXXX'
            );
            
            -- 使用UDTF，将二进制数据解析成格式化数据
            CREATE VIEW input_view (
                name,
                age,
                grade,
                updateTime
            ) AS
            SELECT
                T.name,
                T.age,
                T.grade,
                T.updateTime
            from
                kafka_src as S,
                LATERAL TABLE (kafkapaser (`message`)) as T (
                    name,
                    age,
                    grade,
                    updateTime
                );
            
            -- 使用解析出的格式化数据进行计算，并将结果输出到RDS中
            insert into rds_sink
              SELECT 
                  name,
                  age,
                  grade,
                  updateTime
              from input_view;
            
            ```

        -   UDTF

            **说明：** 以下Maven依赖中的版本，要根据作业运行的版本来定。例如使用2.2.4的Realtime Compute运行作业，maven依赖中各flink模块版本即为blink-2.2.4-SNAPSHOT。下载地址请参看[环境搭建](https://help.aliyun.com/document_detail/69463.html?spm=a2c4g.11174283.6.663.56f51e49xcZv8U#h2-u73AFu5883u642Du5EFA2)。 Maven依赖：

            ```language-java
                <dependencies>
                    <dependency>
                        <groupId>org.apache.flink</groupId>
                        <artifactId>flink-core</artifactId>
                        <version>blink-2.2.4-SNAPSHOT</version>
                        <scope>provided</scope>
                    </dependency>
                    <dependency>
                        <groupId>org.apache.flink</groupId>
                        <artifactId>flink-streaming-java_2.11</artifactId>
                        <version>blink-2.2.4-SNAPSHOT</version>
                        <scope>provided</scope>
                    </dependency>
                    <dependency>
                        <groupId>org.apache.flink</groupId>
                        <artifactId>flink-table_2.11</artifactId>
                        <version>blink-2.2.4-SNAPSHOT</version>
                        <scope>provided</scope>
                    </dependency>
                    <dependency>
                        <groupId>com.alibaba</groupId>
                        <artifactId>fastjson</artifactId>
                        <version>1.2.9</version>
                    </dependency>
                </dependencies>
            
            ```

            ```language-java
            package com.alibaba;
             
            import com.alibaba.fastjson.JSONObject;
            import org.apache.flink.table.functions.TableFunction;
            import org.apache.flink.table.types.DataType;
            import org.apache.flink.table.types.DataTypes;
            import org.apache.flink.types.Row;
             
            import java.io.UnsupportedEncodingException;
            import java.sql.Timestamp;
            
            public class kafkaUDTF extends TableFunction<Row> {
                public void eval(byte[] message) {
                    try {
                        /* input message :
                            {
                              "name":"Alice",
                              "age":13,
                              "grade":"A",
                              "updateTime":1544173862
                            }
                        */
                        String msg = new String(message, "UTF-8");
                        try {
                            JSONObject data = JSON.parseObject(msg);
                            if (data != null) {
                                String name = data.getString("name") == null ? "null" : data.getString("name");
                                Integer age = data.getInteger("age") == null ? 0 : data.getInteger("age");
                                String grade = data.getString("grade") == null ? "null" : data.getString("grade");
                                Timestamp updateTime = data.getTimestamp("updateTime");
             
                                Row row = new Row(4);
                                row.setField(0, name);
                                row.setField(1, age);
                                row.setField(2, grade);
                                row.setField(3,updateTime );
             
                                System.out.println("Kafka message str ==>" + row.toString());
                                collect(row);
                            }
                        } catch (ClassCastException e) {
                            System.out.println("Input data format error. Input data " + msg + "is not json string");
                        }
                    } catch (UnsupportedEncodingException e) {
                        e.printStackTrace();
                    }
                }
                @Override
                // 如果返回值是Row，就必须重载实现这个方法，显式地告诉系统返回的字段类型
                public DataType getResultType(Object[] arguments, Class[] argTypes) {
                    return DataTypes.createRowType(DataTypes.STRING, DataTypes.INT, DataTypes.STRING, DataTypes.TIMESTAMP);
                }
            }
            
            ```

-   示例二
    -   场景

        从kafka读出的数据，需要进行窗口计算。 按照实时计算目前的设计，滚窗/滑窗等窗口操作，需要（且必须）在源表DDL上定义Watermark。Kafka源表比较特殊。如果要以kafka中message字段中的的Event Time进行窗口操作，需要先从message字段，使用UDX解析出event time，才能定义watermark。 在kafka源表场景中，需要使用计算列。 假设，kafka中写入的数据如下：

         `2018-11-11 00:00:00|1|Anna|female` 整个计算流程为：Kafka SOURCE-\>UDTF-\>Realtime Compute-\>RDS SINK。

    -   示例代码
        -   SQL

            ```language-sql
            -- 定义解析Kakfa message的UDTF
            CREATE FUNCTION kafkapaser AS 'com.alibaba.kafkaUDTF';
            CREATE FUNCTION kafkaUDF AS 'com.alibaba.kafkaUDF';
            
            -- 定义源表，注意：afka源表DDL字段必须与以下例子一模一样。WITH中参数可改。
            create table kafka_src (
                messageKey  VARBINARY,
                `message`   VARBINARY,
                topic       VARCHAR,
                `partition` INT,
                `offset`    BIGINT,
                ctime AS TO_TIMESTAMP(kafkaUDF(`message`)), -- 定义计算列，计算列可理解为占位符，源表中并没有这一列，其中的数据可经过下游计算得出。注意计算列的类型必须为timestamp才能在做watermark。
                watermark for `ctime` as withoffset(`ctime`,0) -- 在计算列上定义watermark
            ) WITH (
                type = 'kafka010',    -- Kafka Source类型，与Kafka版本强相关，目前支持的Kafka版本请参考本文档
                topic = 'test_kafka_topic',
                `group.id` = 'test_kafka_consumer_group',
                bootstrap.servers = 'ip1:port1,ip2:port2,ip3:port3'
            );
            
            create table rds_sink (
              name       VARCHAR,
              age        INT,
              grade      VARCHAR,
              updateTime TIMESTAMP
            ) WITH(
             type='rds',
             url='jdbc:mysql://localhost:3306/test',
             tableName='test4',
             userName='test',
             password='XXXXXX'
            );
            
            -- 使用UDTF，将二进制数据解析成格式化数据
            CREATE VIEW input_view (
                name,
                age,
                grade,
                updateTime
            ) AS
            SELECT
                COUNT(*) as cnt,
                T.ctime,
                T.order,
                T.name,
                T.sex
            from
                kafka_src as S,
                LATERAL TABLE (kafkapaser (`message`)) as T (
                    ctime,
                    order,
                    name,
                    sex
                )
            Group BY T.sex,
                    TUMBLE(ctime, INTERVAL '1' MINUTE);
            
            -- 对input_view中输出的数据做计算
            CREATE VIEW view2 (
                cnt,
                sex
            ) AS
            SELECT
                COUNT(*) as cnt,
                T.sex
            from
                input_view
            Group BY sex, TUMBLE(ctime, INTERVAL '1' MINUTE);
            
            
            
            -- 使用解析出的格式化数据进行计算，并将结果输出到RDS中
            insert into rds_sink
              SELECT 
                  cnt,sex
              from view2;
            
            ```

        -   UDF&UDTF

            **说明：** 以下Maven依赖中的版本，要根据作业运行的版本来定。例如使用2.2.4的Realtime Compute运行作业，Maven依赖中各flink模块版本即为blink-2.2.4-SNAPSHOT。下载地址请参看[环境搭建](https://help.aliyun.com/document_detail/69463.html?spm=a2c4g.11174283.6.663.56f51e49xcZv8U#h2-u73AFu5883u642Du5EFA2)。 Maven依赖：

            ```language-java
              <dependencies>
                    <dependency>
                        <groupId>org.apache.flink</groupId>
                        <artifactId>flink-core</artifactId>
                        <version>blink-2.2.4-SNAPSHOT</version>
                        <scope>provided</scope>
                    </dependency>
                    <dependency>
                        <groupId>org.apache.flink</groupId>
                        <artifactId>flink-streaming-java_2.11</artifactId>
                        <version>blink-2.2.4-SNAPSHOT</version>
                        <scope>provided</scope>
                    </dependency>
                    <dependency>
                        <groupId>org.apache.flink</groupId>
                        <artifactId>flink-table_2.11</artifactId>
                        <version>blink-2.2.4-SNAPSHOT</version>
                        <scope>provided</scope>
                    </dependency>
                    <dependency>
                        <groupId>com.alibaba</groupId>
                        <artifactId>fastjson</artifactId>
                        <version>1.2.9</version>
                    </dependency>
                </dependencies>
            
            ```

            -   UDTF

                ```
                package com.alibaba;
                
                import com.alibaba.fastjson.JSONObject;
                import org.apache.flink.table.functions.TableFunction;
                import org.apache.flink.table.types.DataType;
                import org.apache.flink.table.types.DataTypes;
                import org.apache.flink.types.Row;
                
                import java.io.UnsupportedEncodingException;
                
                /**
                  以下例子解析输入Kafka中的JSON字符串，并将其格式化输出
                **/
                public class kafkaUDTF extends TableFunction<Row> {
                
                    public void eval(byte[] message) {
                        try {
                          // 读入一个二进制数据，并将其转换为String格式
                            String msg = new String(message, "UTF-8");
                
                                // 提取JSON Object中各字段
                                    String ctime = Timestamp.valueOf(data.split('\\|')[0]);
                                    String order = data.split('\\|')[1];
                                    String name = data.split('\\|')[2];
                                    String sex = data.split('\\|')[3];
                
                                    // 将解析出的字段放到要输出的Row()对象
                                    Row row = new Row(4);
                                    row.setField(0, ctime);
                                    row.setField(1, age);
                                    row.setField(2, grade);
                                    row.setField(3, updateTime);
                
                
                                    System.out.println("Kafka message str ==>" + row.toString());
                
                                    // 输出一行
                                    collect(row);
                
                
                            } catch (ClassCastException e) {
                                System.out.println("Input data format error. Input data " + msg + "is not json string");
                            }
                
                
                        } catch (UnsupportedEncodingException e) {
                            e.printStackTrace();
                        }
                
                    }
                
                    @Override
                    // 如果返回值是Row，就必须重载实现这个方法，显式地告诉系统返回的字段类型
                    // 定义输出Row()对象的字段类型
                    public DataType getResultType(Object[] arguments, Class[] argTypes) {
                        return DataTypes.createRowType(DataTypes.TIMESTAMP,DataTypes.STRING, DataTypes.Integer, DataTypes.STRING,DataTypes.STRING);
                    }
                
                }
                
                ```

            -   UDF

                ```language-java
                package com.alibaba;
                package com.hjc.test.blink.sql.udx;
                import org.apache.flink.table.functions.FunctionContext;
                import org.apache.flink.table.functions.ScalarFunction;
                
                
                public class KafkaUDF extends ScalarFunction {
                    // 可选，open方法可以不写
                    // 需要import org.apache.flink.table.functions.FunctionContext;
                
                    public String eval(byte[] message) {
                
                         // 读入一个二进制数据，并将其转换为String格式
                        String msg = new String(message, "UTF-8");
                        return msg.split('\\|')[0];
                    }
                    public long eval(String b, String c) {
                        return eval(b) + eval(c);
                    }
                    //可选，close方法可以不写
                    @Override
                    public void close() {
                        }
                }
                
                ```


自建Kafka

-   示例

    ```language-sql
    create table kafka_stream(
      messageKey VARBINARY,
      `message` VARBINARY, 
      topic varchar,
      `partition` int,
      `offset` bigint
    ) with (
      type ='kafka011',
      topic = 'kafka_01',
      `group.id` = 'CID_blink',
      bootstrap.servers = '192.168.0.251:9092'
    );
    
    ```

-   WITH参数

    关于自建Kafka的WITH参数，请参考本文档Kafka创建时DDL的WITH参数说明。需要注意的是`bootstrap.servers`参数需要填写自建的地址和端口号。

    **说明：** 实时计算仅在2.2.6及以上版本中，支持阿里云Kafka或自建Kafka的TPS、RPS等指标信息的显示。


