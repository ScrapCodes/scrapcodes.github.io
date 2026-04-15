---
layout: post
title:  "Simple performance patterns but massive gains - Iceberg on Prestodb."
date:   2026-03-30 05:54:31 +0530
categories: Iceberg Prestodb
---
Recently, a issue came to me with the user complaining their code takes 43 minutes to update 1000 rows. My immediate impression was, may be they have a really complex update query. A typical update query syntax is : 

{% highlight sql %}
UPDATE table_name SET [ column = expression [, ... ] ] [ WHERE condition ]
{% endhighlight %}
More on: [prestodb update docs][presto-update]

This is designed to update several rows based on certain WHERE clause condition being met. It turned out user was using a update query per each row in the target table, of size 1000 rows. That means they had about 1000 update queries. Next since the table was backed by Iceberg format stored on HDFS storage - this had a completely different meaning. Neither of the Iceberg or HDFS are designed for such usage patterns. Often users coming from RDBMS background, tend to have similar expectations from system designed for handling data storage sizes beyond peta bytes. No wonder sales guys takes the adventurous customer through the new and fancy product and customer buys it whether or not they are going to use those features. Their hope is, if it's faster for very large data then it will be faster for my small amount of data as well, and since it is growing it will be wise to migrate to such a system sooner than later.

Anyways, for each executed update Iceberg was generating a snapshot and all the associated metadata. This resulted in huge metadata bloat and if we go on updating each rows as a single update sql, we would result a huge amount of Iceberg metadata. Iceberg is not designed for this kind of usage pattern, though there is compaction that can compact the metadata and prune the unnecessary snapshots by calling stored procedures [Iceberg stored procedures][iceberg-spark-sp]. Calling these stored procedure and achieving exactly what we want requires a strategy of it's own (topic for another time).

Their code for updating those rows looked like the following.
{% highlight java %}
  String updateSQL = String.format("UPDATE \"%s\".\"%s\".\"%s\" SET \"C2\" = ?, \"C3\" = ?, \"C4\" = ? WHERE \"C1\" = ?", "iceberg", "perf_test", "perf_test_tab");
  preparedStatement = connection.prepareStatement(updateSQL);
  // update batch
  for (int i = 1; i <= TOTAL_RECORDS; ++i) {
      preparedStatement.setString(1, "UpdatedValue_" + i);
      preparedStatement.setLong(2, 1000L + (long) i);
      preparedStatement.setBigDecimal(3, (new BigDecimal("123.45")).add(new BigDecimal(i)));
      preparedStatement.setInt(4, i);
      preparedStatement.addBatch();
      if (i % 20 == 0) {
          System.out.println("  Added " + i + " statements to batch...");
      }
  }

  startExecTime = System.currentTimeMillis();
  results = preparedStatement.executeBatch();
  execTime = System.currentTimeMillis() - startExecTime;
{% endhighlight %}

This was consuming a whooping 43 minutes of time for the reasons already explained. 

If we want to achieve update of all the rows (or large number of rows) in a table. Just insert the data to be updated into a new table first and then use MERGE table as follows.

{% highlight sql %}
presto> select * from iceberg.perf_test.perf_test_tab ORDER BY c1 LIMIT 10;
 c1 |        c2        |  c3  |   c4   
----+------------------+------+--------
  1 | InsertedValue_10 | 1001 | 124.45 
  2 | InsertedValue_11 | 1002 | 125.45 
  3 | InsertedValue_12 | 1003 | 126.45 
  4 | InsertedValue_13 | 1004 | 127.45 
  5 | InsertedValue_14 | 1005 | 128.45 
  6 | InsertedValue_15 | 1006 | 129.45 
  7 | InsertedValue_16 | 1007 | 130.45 
  8 | InsertedValue_17 | 1008 | 131.45 
  9 | InsertedValue_18 | 1009 | 132.45 
 10 | InsertedValue_19 | 1010 | 133.45 
(10 rows)

Query 20260410_160854_00140_zb4hc, FINISHED, 1 node
http://127.0.0.1:8080/ui/query.html?20260410_160854_00140_zb4hc
Splits: 27 total, 27 done (100.00%)
[Latency: client-side: 192ms, server-side: 183ms] [1.01K rows, 13KB] [5.52K rows/s, 71.3KB/s]

presto> select * from iceberg.perf_test.perf_test_tab2 ORDER BY c1 LIMIT 10;
 c1 |       c2        |  c3  |   c4   
----+-----------------+------+--------
  1 | UpdatedValue_10 | 1001 | 124.45 
  2 | UpdatedValue_11 | 1002 | 125.45 
  3 | UpdatedValue_12 | 1003 | 126.45 
  4 | UpdatedValue_13 | 1004 | 127.45 
  5 | UpdatedValue_14 | 1005 | 128.45 
  6 | UpdatedValue_15 | 1006 | 129.45 
  7 | UpdatedValue_16 | 1007 | 130.45 
  8 | UpdatedValue_17 | 1008 | 131.45 
  9 | UpdatedValue_18 | 1009 | 132.45 
 10 | UpdatedValue_19 | 1010 | 133.45 
(10 rows)

Query 20260410_160912_00142_zb4hc, FINISHED, 1 node
http://127.0.0.1:8080/ui/query.html?20260410_160912_00142_zb4hc
Splits: 27 total, 27 done (100.00%)
[Latency: client-side: 197ms, server-side: 187ms] [1.01K rows, 13KB] [5.4K rows/s, 69.7KB/s]

presto> MERGE INTO iceberg.perf_test.perf_test_tab as t1
     -> USING iceberg.perf_test.perf_test_tab2 as t2
     -> ON t1.c1 = t2.c1
     -> WHEN MATCHED THEN
     -> UPDATE SET
     ->  c2 = t2.c2
     -> , c3 = t2.c3
     -> , c4 = t2.c4
     -> WHEN NOT MATCHED THEN
     ->     INSERT (c1, c2, c3, c4)
     ->     VALUES (t2.c1, t2.c2, t2.c3, t2.c4);
MERGE: 1000 rows

Query 20260410_160929_00143_zb4hc, FINISHED, 1 node
http://127.0.0.1:8080/ui/query.html?20260410_160929_00143_zb4hc
Splits: 102 total, 102 done (100.00%)
[Latency: client-side: 0:01, server-side: 0:01] [2.02K rows, 26.1KB] [3.08K rows/s, 39.8KB/s]

presto> select * from iceberg.perf_test.perf_test_tab ORDER BY c1 LIMIT 10;
 c1 |       c2        |  c3  |   c4   
----+-----------------+------+--------
  1 | UpdatedValue_10 | 1001 | 124.45 
  2 | UpdatedValue_11 | 1002 | 125.45 
  3 | UpdatedValue_12 | 1003 | 126.45 
  4 | UpdatedValue_13 | 1004 | 127.45 
  5 | UpdatedValue_14 | 1005 | 128.45 
  6 | UpdatedValue_15 | 1006 | 129.45 
  7 | UpdatedValue_16 | 1007 | 130.45 
  8 | UpdatedValue_17 | 1008 | 131.45 
  9 | UpdatedValue_18 | 1009 | 132.45 
 10 | UpdatedValue_19 | 1010 | 133.45 
(10 rows)

Query 20260410_160940_00144_zb4hc, FINISHED, 1 node
http://127.0.0.1:8080/ui/query.html?20260410_160940_00144_zb4hc
Splits: 28 total, 28 done (100.00%)
[Latency: client-side: 236ms, server-side: 228ms] [2.01K rows, 21.8KB] [8.81K rows/s, 95.5KB/s]

presto> 
{% endhighlight %}

It took just 1 second to do what was taking 43 minutes previously, and the metadata is clean too - no compaction to worry about. 

Here we used the following merge command:
{% highlight sql %}
MERGE INTO iceberg.perf_test.perf_test_tab as t1
USING iceberg.perf_test.perf_test_tab2 as t2
ON t1.c1 = t2.c1
WHEN MATCHED THEN
UPDATE SET
 c2 = t2.c2
, c3 = t2.c3
, c4 = t2.c4
WHEN NOT MATCHED THEN
    INSERT (c1, c2, c3, c4)
    VALUES (t2.c1, t2.c2, t2.c3, t2.c4);
{% endhighlight %}

This says, for every row in table t2, if column c1 is equal to column c1 in table t1, perform update by setting all the columns from table t2 to table t1. If a row exists in table t2 and not in table t1, then just insert it in t1. That's it. More on Prestodb's merge command, [Prestodb merge docs][presto-merge].

A similar phenomenon can be observed if we insert a row in iceberg table using insert statements per row. For example:

{% highlight java %}
String insertSQL = String.format("INSERT INTO \"%s\".\"%s\".\"%s\" VALUES (?, ? , ?, ?)", "iceberg", "perf_test", "perf_test_tab");
preparedStatement = connection.prepareStatement(insertSQL);
// insert batch
for (int i = 1; i <= TOTAL_RECORDS; ++i) {
    preparedStatement.setInt(1, i);
    preparedStatement.setString(2, "InsertedValue_" + i);
    preparedStatement.setLong(3, 1000L + (long) i);
    preparedStatement.setBigDecimal(4, (new BigDecimal("123.45")).add(new BigDecimal(i)));
    preparedStatement.addBatch();
}
long startExecTime = System.currentTimeMillis();
int[] results = preparedStatement.executeBatch();
long execTime = System.currentTimeMillis() - startExecTime;
System.out.println("✓ insert Batch of " + TOTAL_RECORDS + " executed in " + execTime + " ms");
{% endhighlight %}

In this case, we would expect presto-jdbc to do something intelligent because we are sending insert's in batch of preperad statements, however such a optimization is difficult to generalize across different connectors. A rdbms based connector will not need the same optimization that iceberg would and similarly, each insert could be for a different connector type too. 

The above insert batch can be developed as follows:
{% highlight java %}
 String insertSQL = "INSERT INTO \"%s\".\"%s\".\"%s\" VALUES (%d, 'first_row' , 100, 11.20)";
String valuesAddendum = ", (%s, '%s', %s, %.2f)";
// insert batch
Statement statement1 = connection.createStatement();
long startExecTime = System.currentTimeMillis();
for (int i = 1; i <= TOTAL_RECORDS; i = i + 200) {
    long startMicroBatchExecTime = System.currentTimeMillis();
    StringBuilder sb = new StringBuilder(String.format(insertSQL, "iceberg", "perf_test", "perf_test_tab", i));
    for (int j = 1; j < 200; j++) {
        sb.append(String.format(valuesAddendum, i + j, "InsertedValue_" + i + j, 1000L + (long) i + j, (new BigDecimal("123.45")).add(new BigDecimal(i + j))));
    }
    statement1.execute(sb.toString());
    System.out.println("Batch no " + i + " time: " + (System.currentTimeMillis() - startMicroBatchExecTime));
}
{% endhighlight %}

This way we batch 200 insert rows as single SQL insert, and does not create a snapshot per row. This is 100x faster than inserting a single row at a time, completes in less than a second opposed to minutes. Incase you are wondering why do we have a batch of 200, might as well insert all 1000 rows in one go. That can be done in this case, but since there is a limit on size of a single SQL query, it cannot be done for every single case. Infact, 200 is not a magic number, a end user has to carefully choose this number such that they do not exceed SQL query size limit. This can be configured via `query.max-length` in prestodb's `config.properties`.

[iceberg-spark-sp]: https://iceberg.apache.org/docs/latest/spark-procedures/
[presto-update]:   https://prestodb.io/docs/current/sql/update.html
[presto-merge]: https://prestodb.io/docs/current/sql/merge.html
