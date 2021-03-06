---
title: Metrics Schema
summary: 了解 TiDB METRICS SCHEMA 系统数据库。
category: reference
aliases: ['/docs-cn/dev/reference/system-databases/metrics-schema/']
---

# Metrics Schema

为了能够动态地观察并对比不同时间段的集群情况，TiDB 4.0 诊断系统添加了集群监控系统表。所有表都在 metrics schema 中，可以通过 SQL 的方式查询监控。实际上，SQL 诊断，以及 `metrics_summary`，`metrics_summary_by_label`，`inspection_result` 这三个监控相关的汇总表数据都是通过查询 metrics schema 库中的各种监控表来获取信息的。目前添加的系统表数量较多，用户可以通过 [`information_schema.metrics_tables`](/system-tables/system-table-metrics-tables.md) 查询这些表的相关信息。

## 概览

下面以 `tidb_query_duration` 表来作为示例介绍监控表相关的使用和原理，其他的监控表原理都是类似的。

先查询 `information_schema.metrics_tables` 中关于 `tidb_query_duration` 表相关的信息：

{{< copyable "sql" >}}

```sql
select * from information_schema.metrics_tables where table_name='tidb_query_duration';
```

```
+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------+----------+----------------------------------------------+
| TABLE_NAME          | PROMQL                                                                                                                                                   | LABELS            | QUANTILE | COMMENT                                      |
+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------+----------+----------------------------------------------+
| tidb_query_duration | histogram_quantile($QUANTILE, sum(rate(tidb_server_handle_query_duration_seconds_bucket{$LABEL_CONDITIONS}[$RANGE_DURATION])) by (le,sql_type,instance)) | instance,sql_type | 0.9      | The quantile of TiDB query durations(second) |
+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------+----------+----------------------------------------------+
```

* `TABLE_NAME`：对应于 metrics schema 中的表名，这里表名是 `tidb_query_duration`。
* `PROMQL`：因为监控表的原理是将 SQL 映射成 `PromQL`，并将 Prometheus 结果转换成 SQL 查询结果。这个字段是 `PromQL` 的表达式模板，获取监控表数据时使用查询条件改写模板中的变量，生成最终的查询表达式。
* `LABELS`：监控定义的 label，`tidb_query_duration` 有两个 label，分别是 `instance` 和 `sql_type`。
* `QUANTILE`：百分位。直方图类型的监控数据会指定一个默认百分位。如果值为 `0`，表示该监控表对应的监控不是直方图。`tidb_query_duration` 默认查询 0.9 ，也就是 P90 的监控值。
* `COMMENT`：对这个监控表的解释。可以看出 `tidb_query_duration` 表是用来查询 TiDB query 执行的百分位时间，如 P999/P99/P90 的查询耗时，单位是秒。

再来看 `tidb_query_duration` 的表结构：

{{< copyable "sql" >}}

```sql
show create table metrics_schema.tidb_query_duration;
```

```
+---------------------+--------------------------------------------------------------------------------------------------------------------+
| Table               | Create Table                                                                                                       |
+---------------------+--------------------------------------------------------------------------------------------------------------------+
| tidb_query_duration | CREATE TABLE `tidb_query_duration` (                                                                               |
|                     |   `time` datetime unsigned DEFAULT CURRENT_TIMESTAMP,                                                              |
|                     |   `instance` varchar(512) DEFAULT NULL,                                                                            |
|                     |   `sql_type` varchar(512) DEFAULT NULL,                                                                            |
|                     |   `quantile` double unsigned DEFAULT '0.9',                                                                        |
|                     |   `value` double unsigned DEFAULT NULL                                                                             |
|                     | ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='The quantile of TiDB query durations(second)' |
+---------------------+--------------------------------------------------------------------------------------------------------------------+
```

* `time`：监控项的时间。
* `instance` 和 `sql_type`：是 `tidb_query_duration` 这个监控项的 label。`instance` 表示监控的地址，`sql_type` 表示执行 SQL 的类似。
* `quantile`，百分位，直方图类型的监控都会有该列，表示查询的百分位时间，如 `quantile=0.9` 就是查询 P90 的时间。
* `value`：监控项的值。

下面是查询时间 [`2020-03-25 23:40:00`, `2020-03-25 23:42:00`] 范围内的 P99 的 TiDB Query 耗时：

{{< copyable "sql" >}}

```sql
select * from metrics_schema.tidb_query_duration where value is not null and time>='2020-03-25 23:40:00' and time <= '2020-03-25 23:42:00' and quantile=0.99;
```

```
+---------------------+-------------------+----------+----------+----------------+
| time                | instance          | sql_type | quantile | value          |
+---------------------+-------------------+----------+----------+----------------+
| 2020-03-25 23:40:00 | 172.16.5.40:10089 | Insert   | 0.99     | 0.509929485256 |
| 2020-03-25 23:41:00 | 172.16.5.40:10089 | Insert   | 0.99     | 0.494690793986 |
| 2020-03-25 23:42:00 | 172.16.5.40:10089 | Insert   | 0.99     | 0.493460506934 |
| 2020-03-25 23:40:00 | 172.16.5.40:10089 | Select   | 0.99     | 0.152058493415 |
| 2020-03-25 23:41:00 | 172.16.5.40:10089 | Select   | 0.99     | 0.152193879678 |
| 2020-03-25 23:42:00 | 172.16.5.40:10089 | Select   | 0.99     | 0.140498483232 |
| 2020-03-25 23:40:00 | 172.16.5.40:10089 | internal | 0.99     | 0.47104        |
| 2020-03-25 23:41:00 | 172.16.5.40:10089 | internal | 0.99     | 0.11776        |
| 2020-03-25 23:42:00 | 172.16.5.40:10089 | internal | 0.99     | 0.11776        |
+---------------------+-------------------+----------+----------+----------------+
```

以上查询结果的第一行意思是，在 `2020-03-25 23:40:00` 时，在 TiDB 实例 `172.16.5.40:10089` 上，`Insert` 类型的语句的 P99 执行时间是 0.509929485256 秒。其他各行的含义类似，`sql_type` 列的其他值含义如下：

* `Select`：表示执行的 `select` 类型的语句。
* `internal`：表示 TiDB 的内部 SQL 语句，一般是统计信息更新，获取全局变量相关的内部语句。

进一步再查看上面语句的执行计划如下：

{{< copyable "sql" >}}

```sql
desc select * from metrics_schema.tidb_query_duration where value is not null and time>='2020-03-25 23:40:00' and time <= '2020-03-25 23:42:00' and quantile=0.99;
```

```
+------------------+----------+------+---------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| id               | estRows  | task | access object             | operator info                                                                                                                                                                                          |
+------------------+----------+------+---------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Selection_5      | 8000.00  | root |                           | not(isnull(Column#5))                                                                                                                                                                                  |
| └─MemTableScan_6 | 10000.00 | root | table:tidb_query_duration | PromQL:histogram_quantile(0.99, sum(rate(tidb_server_handle_query_duration_seconds_bucket{}[60s])) by (le,sql_type,instance)), start_time:2020-03-25 23:40:00, end_time:2020-03-25 23:42:00, step:1m0s |
+------------------+----------+------+---------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

可以发现执行计划中有一个 `PromQL`, 以及查询监控的 `start_time` 和 `end_time`，还有 `step` 值，在实际执行时，TiDB 会调用 Prometheus 的 `query_range` HTTP API 接口来查询监控数据。

从以上结果可知，在 [`2020-03-25 23:40:00`, `2020-03-25 23:42:00`] 时间范围内，每个 label 只有三个时间的值，执行计划中的 `step` 值为一分钟，这实际上是由下面两个 session 变量决定的：

* `tidb_metric_query_step`：查询的分辨率步长。从 Prometheus 的 `query_range` 接口查询数据时需要指定 `start_time`，`end_time` 和 `step`，其中 `step` 会使用该变量的值。
* `tidb_metric_query_range_duration`：查询监控时，会将 `PROMQL` 中的 `$RANGE_DURATION` 替换成该变量的值，默认值是 60 秒。

如果想要查看不同时间粒度的监控项的值，用户可以修改上面两个 session 变量后查询监控表，示例如下：

首先修改两个 session 变量的值，将时间粒度设置为 30 秒。

> **注意：**
>
> Prometheus 支持查询的最小粒度为 30 秒。

{{< copyable "sql" >}}

```sql
set @@tidb_metric_query_step=30;
set @@tidb_metric_query_range_duration=30;
```

再查询 `tidb_query_duration` 监控如下，可以发现在三分钟时间范围内，每个 label 有六个时间的值，每个值时间间隔是 30 秒。

{{< copyable "sql" >}}

```sql
select * from metrics_schema.tidb_query_duration where value is not null and time>='2020-03-25 23:40:00' and time <= '2020-03-25 23:42:00' and quantile=0.99;
```

```
+---------------------+-------------------+----------+----------+-----------------+
| time                | instance          | sql_type | quantile | value           |
+---------------------+-------------------+----------+----------+-----------------+
| 2020-03-25 23:40:00 | 172.16.5.40:10089 | Insert   | 0.99     | 0.483285651924  |
| 2020-03-25 23:40:30 | 172.16.5.40:10089 | Insert   | 0.99     | 0.484151462113  |
| 2020-03-25 23:41:00 | 172.16.5.40:10089 | Insert   | 0.99     | 0.504576        |
| 2020-03-25 23:41:30 | 172.16.5.40:10089 | Insert   | 0.99     | 0.493577384561  |
| 2020-03-25 23:42:00 | 172.16.5.40:10089 | Insert   | 0.99     | 0.49482474311   |
| 2020-03-25 23:40:00 | 172.16.5.40:10089 | Select   | 0.99     | 0.189253402185  |
| 2020-03-25 23:40:30 | 172.16.5.40:10089 | Select   | 0.99     | 0.184224951851  |
| 2020-03-25 23:41:00 | 172.16.5.40:10089 | Select   | 0.99     | 0.151673410553  |
| 2020-03-25 23:41:30 | 172.16.5.40:10089 | Select   | 0.99     | 0.127953838989  |
| 2020-03-25 23:42:00 | 172.16.5.40:10089 | Select   | 0.99     | 0.127455434547  |
| 2020-03-25 23:40:00 | 172.16.5.40:10089 | internal | 0.99     | 0.0624          |
| 2020-03-25 23:40:30 | 172.16.5.40:10089 | internal | 0.99     | 0.12416         |
| 2020-03-25 23:41:00 | 172.16.5.40:10089 | internal | 0.99     | 0.0304          |
| 2020-03-25 23:41:30 | 172.16.5.40:10089 | internal | 0.99     | 0.06272         |
| 2020-03-25 23:42:00 | 172.16.5.40:10089 | internal | 0.99     | 0.0629333333333 |
+---------------------+-------------------+----------+----------+-----------------+
```

最后查看执行计划，也会发现执行计划中的 `PromQL` 以及 `step` 的值都已经变成了 30 秒。

{{< copyable "sql" >}}

```sql
desc select * from metrics_schema.tidb_query_duration where value is not null and time>='2020-03-25 23:40:00' and time <= '2020-03-25 23:42:00' and quantile=0.99;
```

```
+------------------+----------+------+---------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| id               | estRows  | task | access object             | operator info                                                                                                                                                                                         |
+------------------+----------+------+---------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Selection_5      | 8000.00  | root |                           | not(isnull(Column#5))                                                                                                                                                                                 |
| └─MemTableScan_6 | 10000.00 | root | table:tidb_query_duration | PromQL:histogram_quantile(0.99, sum(rate(tidb_server_handle_query_duration_seconds_bucket{}[30s])) by (le,sql_type,instance)), start_time:2020-03-25 23:40:00, end_time:2020-03-25 23:42:00, step:30s |
+------------------+----------+------+---------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```
