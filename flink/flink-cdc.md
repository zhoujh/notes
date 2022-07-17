# FLINK-CDC 实时采集MySQL数据

Flink集成了 Debezium实现MySQL binlog的实时采集，可以获得MySQL的数据变化。

应用开发的方式有Flink-SQL 和 StreamAPI两种方式。

测试环境 flink 1.14.4 ，依赖的mysql-cdc包

```
<dependency>
    <groupId>com.ververica</groupId>
    <artifactId>flink-sql-connector-mysql-cdc</artifactId>
    <version>2.2.0</version>
</dependency>
```


# * 1.使用Flink-SQL进行采集

## 1) 创建数据源

```
CREATE TABLE orders (
     order_id INT,
     order_date TIMESTAMP(0),
     customer_name STRING,
     price DECIMAL(10, 5),
     product_id INT,
     order_status BOOLEAN,
     PRIMARY KEY(order_id) NOT ENFORCED
     ) WITH (
     'connector' = 'mysql-cdc',
     'hostname' = 'localhost',
     'port' = '3306',
     'username' = 'root',
     'password' = '123456',
     'database-name' = 'mydb',
     'table-name' = 'orders');
```

## 2）目标数据源输出kafka

```
CREATE TABLE kafka_orders (
     order_id INT,
     order_date TIMESTAMP(0),
     customer_name STRING,
     price DECIMAL(10, 5),
     product_id INT,
     order_status BOOLEAN,
     PRIMARY KEY(order_id) NOT ENFORCED
     ) WITH ( 
 'connector' ='kafka',
 'format' ='debezium-json',
 'topic' ='kafka_orders',
 'properties.group.id' ='job-1652068107157',
 'scan.startup.mode' = 'latest-offset',
 'properties.bootstrap.servers' = '127.0.0.1:19092'
 ); 
 
```

## 3）执行SQL同步数据

读取mysql-cdc数据源表，写入kafak的数据源表。

insert into kafka_orders
select order_id,order_date,customer_name,product_id,order_status from orders ；

## 4）存在问题

这种方式，会针对每个表创建1个mysql 连接进行binlog采集的。

用StreamTableEnvironmenttableEn.createTable 的方式注册MySQL CDC表，效果也是一样。


# 2.通过mysql-cdc的API进行采集

大致思路是通过mysql-cdc定于的数据源，采集binlog日志，再通过flink的sideoutput,按表推送到不同的kafka队列中。

mysql-cdc的数据源类为com.ververica.cdc.connectors.mysql.source.MySqlSource

## 1）数据源声明

```
MySqlSource `<String>` source = MySqlSource.`<String>`builder()
                .hostname(host)
                .port(portInt)
                .password(jdbcConf.getPassword())
                .username(jdbcConf.getUsername())
                .databaseList(srcDbList.toArray(new String[] {}))
                .tableList(srcTableList.toArray(new String[] {}))
                .deserializer(debe)
                .serverId(dsId))
                .startupOptions(StartupOptions.latest())
                .jdbcProperties(jdbcProperties).build();
```

其中Deserializer debe 是在内置的JsonDebeziumDeserializationSchema基础上进行修改的反序列化类。内置的JsonDebeziumDeserializationSchema和JsonConverterJso在处理时间类型时，使用的日期格式比较特殊，需要拷贝出来继续修改（可能有其他更好的处理方式）。Debezium的同步记录中，日期和时间相关的类型，值用long/int加scehma类型来保存。例如io.debezium.time.Date 类型，值为从1970开始的天数。需要根据类型处理转成需要的时间格式。这里统一转成yyyy-MM-dd HH:mm:ss

拷贝JsonConverter类convertToJson方法中进行修改增加日期的类型的处理（漏了ZonedTimestamp、ZonedTime 可能会有问题，待处理）

```
if (schema.name().equals(io.debezium.time.Timestamp.SCHEMA_NAME) ||
                schema.name().equals(io.debezium.time.Date.SCHEMA_NAME) ||
                schema.name().equals(io.debezium.time.MicroTimestamp.SCHEMA_NAME) ||
                schema.name().equals(io.debezium.time.NanoTimestamp.SCHEMA_NAME)) {
                return JSON_NODE_FACTORY.textNode(DebeTimeUtils.convertToTimestampString(value, schema));
            }
```

其中DebeTimeUtils为：

```
public class DebeTimeUtils {
    static FastDateFormat fdf = FastDateFormat.getInstance("yyyy-MM-dd HH:mm:ss");

    public static String convertToTimestampString(Object dbzObj, Schema schema) {
        TimestampData timestampData = convertToTimestamp(dbzObj, schema);
        java.sql.Timestamp timestamp = timestampData.toTimestamp();
        return fdf.format(timestamp);
    }

    private static TimestampData convertToTimestamp(Object dbzObj, Schema schema) {
        long value = 0;
        if (dbzObj instanceof Long) {
            value = (long)dbzObj;
        } else if (dbzObj instanceof Integer) {
            value = (int)dbzObj;
        } else {
            LocalDateTime localDateTime =
                TemporalConversions.toLocalDateTime(dbzObj, TimeZone.getTimeZone("GMT+8").toZoneId());
            return TimestampData.fromLocalDateTime(localDateTime);
        }

    switch (schema.name()) {
            case Timestamp.SCHEMA_NAME:
                return TimestampData.fromEpochMillis(value);
            case MicroTimestamp.SCHEMA_NAME:
                return TimestampData.fromEpochMillis(value / 1000, (int)(value % 1000 * 1000));
            case NanoTimestamp.SCHEMA_NAME:
                return TimestampData.fromEpochMillis(value / 1000_000, (int)(value % 1000_000));
            case Date.SCHEMA_NAME:
                return TimestampData.fromEpochMillis(value * 86400000, 0);
        }
        throw new RuntimeException("Invalid type " + schema.name());
    }
}
```

## 2）按表分发数据

按表名将数据发送到不同的sideOutput，再关联到Kafka Sink 。

```
        DataStreamSource<String> mySqlCdcSource =
            streamEnv.fromSource(source, WatermarkStrategy.noWatermarks(), "mySqlCdcSource");

        ArrayList<String> tableList = new ArrayList();
        tableList.add("test.t_user");
        Map<String, OutputTag<String>> outputTags = new HashMap<String, OutputTag<String>>();

        for (int i = 0; i < tableList.size(); i++) {
            String fullTableName = tableList.get(i);
            OutputTag<String> outi = new OutputTag<String>(fullTableName) {};
            outputTags.put(fullTableName, outi);
        }

        SingleOutputStreamOperator<Object> mainStream =
            mySqlCdcSource.process(new ProcessFunction<String, Object>() {
                @Override
                public void processElement(String value, ProcessFunction<String, Object>.Context ctx,
                    Collector<Object> out)
                    throws Exception {
                    JSONObject jsonObject = JSON.parseObject(value);
                    JSONObject source1 = jsonObject.getJSONObject("source");
                    String tableName = source1.getString("table");
                    String db = source1.getString("db");
                    OutputTag<String> outputTag = outputTags.get(db + "." + tableName);
                    ctx.output(outputTag, value);
                }
            });

        for (int i = 0; i < tableList.size(); i++) {
            DataStream<String> sideOutput = mainStream.getSideOutput(outputTags.get(tableList.get(i)));
            DataStreamSink dataStreamSink = sideOutput.addSink(buildKafkaSink(tableList.get(i)));
            dataStreamSink.name(sideOutput.toString());
            LOG.info("CDC_JOB init SideOutput {} ", tableList.get(i));
        }
        streamEnv.execute("test-job");
    }
    private static RichSinkFunction buildKafkaSink(String topicName) {//测试输出
        return new PrintSinkFunction<String>(topicName,false);
    }

```

## **3）输出结果**

测试输出的消息格式为：

```
test.t_user> {"before":{"id":2,"name":"tom","birthday":"2010-01-01 00:00:00","create_time":null,"gender":4,"dept_id":2},"after":{"id":2,"name":"tom","birthday":"2010-01-01 00:00:00","create_time":null,"gender":1,"dept_id":2},"source":{"version":"1.5.4.Final","connector":"mysql","name":"mysql_binlog_source","ts_ms":1658030708000,"snapshot":"false","db":"test","sequence":null,"table":"t_user","server_id":1,"gtid":null,"file":"binlog.000026","pos":5019,"row":0,"thread":null,"query":null},"op":"u","ts_ms":1658030708587,"transaction":null}{"before":{"id":2,"name":"tom","birthday":"2010-01-01 00:00:00","gender":1,"dept_id":2},"after":{"id":2,"name":"tom","birthday":"2010-01-01 00:00:00","gender":2,"dept_id":2},"source":{"version":"1.5.4.Final","connector":"mysql","name":"mysql_binlog_source","ts_ms":1658027718000,"snapshot":"false","db":"test","sequence":null,"table":"t_user","server_id":1,"gtid":null,"file":"binlog.000026","pos":1037,"row":0,"thread":null,"query":null},"op":"u","ts_ms":1658027719021,"transaction":null
```


这种方式只产生一个MySQL的slave， 但是每个表不能根据需要定制cdc的参数，例如设置offset，不过应该可能满足很多场景了。
