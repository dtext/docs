---
title: About compression
excerpt: How to compress hypertables
products: [self_hosted]
keywords: [compression, hypertables]
---

import CompressionIntro from 'versionContent/_partials/_compression-intro.mdx';

# About compression

<CompressionIntro />

<Highlight type="note">
Most indexes set on the hypertable are removed or ignored
when reading from compressed chunks. Timescale creates and uses custom indexes
to incorporate the `segmentby` and `orderby` parameters during compression.
</Highlight>

This section explains how to enable native compression, and then goes into
detail on the most important settings for compression, to help you get the
best possible compression ratio.

## Key aspects of compression

Compression always starts with the hypertable. Every table has a different schema but they do share some commonalities that we need to think about.

Let's take the following schema as an example table named `metrics`:

|Column|Type|Collation|Nullable|Default|
|-|-|-|-|-|
 time|timestamp with time zone|| not null|
 device_id| integer|| not null|
 device_type| integer|| not null|
 cpu| double precision|||
 disk_io| double precision|||

Our hypertable needs to have a primary dimension. In the example, we are showing the classic time-series use case with the `time` column as the primary dimension that is used for partitioning. Besides that we have two columns `cpu` and `disk_io` which are essentially value columns that we are capturing over time. There is also another column `device_id` which is used as a designator or a lookup key which designates which device these captured values belong to at a certain point in time.
Columns can be used in a few different ways:
You can use values in a column as a lookup key, in the example above the device_id is a typical example of such a column.
You can use a column for partitioning a table. This is typically a time column, but it is possible to partition the table using other columns as well.
You can use a column as a filter to narrow down on what data you select. The column device_type is an example of such a column where you can decide to only look at, for example, solid state drives (SSDs)
The remaining columns typically are the values or metrics you are querying for. They are typically aggregated or presented in other ways. The columns `cpu` and `disk_io` are typical examples of such columns.
An example query using value columns and filter on time and device type could look like this:

<CodeBlock canCopy={false} showLineNumbers={false} children={`
SELECT avg(cpu), sum(disk_io)
FROM metrics
WHERE device_type = ‘SSD’
AND time >= now() - ‘1 day’::interval;
`} />

When chunks are compressed in a hypertable, data stored in them is reorganized and stored in column-order rather than row-order. As a result, it is not possible to use the same uncompressed schema version of the chunk and a different schema must be created. This is automatically handled by TimescaleDB, but it has a few implications:
The compression ratio and query performance is very dependent on the order and structure of the compressed data, so some considerations are needed when setting up compression.
Indexes on the hypertable cannot always be used in the same manner for the compressed data.


Based on the previous schema, filtering of data should happen over a certain time period and analytics are done on device granularity. This pattern of data access lends itself to organizing the data layout suitable for compression.

### Segmenting and ordering.

Segmenting the compressed data should be based on the way you access the data. Basically, you want to segment your data in such a way that you can make it easier for your queries to fetch the right data at the right time. That is to say, your queries should dictate how you segment the data so they can be optimized and yield even better query performance.

For example, If you want to access a single device using a specific `device_id` value (either all records or maybe for a specific time range), you would need to filter all those records one by one during row access time. To get around this, you can use device_id column for segmenting. This would allow you to run analytical queries on compressed data much faster if you are looking for specific device IDs.

<CodeBlock canCopy={false} showLineNumbers={false} children={`
postgres=# \\timing
Timing is on.
postgres=# SELECT device_id, AVG(cpu) AS avg_cpu, AVG(disk_io) AS avg_disk_io 
FROM metrics 
WHERE device_id = 5 
GROUP BY device_id;
 device_id |      avg_cpu       |     avg_disk_io     
-----------+--------------------+---------------------
         5 | 0.4972598866221261 | 0.49820356730280524
(1 row)
Time: 177,399 ms
postgres=# ALTER TABLE metrics SET (timescaledb.compress, timescaledb.compress_segmentby = 'device_id', timescaledb.compress_orderby='time');
ALTER TABLE
Time: 6,607 ms
postgres=# SELECT compress_chunk(c) FROM show_chunks('metrics') c;
             compress_chunk             
----------------------------------------
 _timescaledb_internal._hyper_2_4_chunk
 _timescaledb_internal._hyper_2_5_chunk
 _timescaledb_internal._hyper_2_6_chunk
(3 rows)
Time: 3070,626 ms (00:03,071)
postgres=# SELECT device_id, AVG(cpu) AS avg_cpu, AVG(disk_io) AS avg_disk_io 
FROM metrics 
WHERE device_id = 5 
GROUP BY device_id;
 device_id |      avg_cpu      |     avg_disk_io     
-----------+-------------------+---------------------
         5 | 0.497259886622126 | 0.49820356730280535
(1 row)
Time: 42,139 ms
`} />

Ordering the data will have a great impact on the compression ratio since we want to have rows that change over a dimension (most likely time) close to each other. Most of the time data changes in a predictable fashion, following a certain trend. We can exploit this fact to encode the data so it takes less space to store. For example, if you order the records over time, they will get compressed in that order and subsequently also accessed in the same order.


This makes the time column a perfect candidate for ordering your data since the measurements evolve as time goes on. If you were to use that as your only compression setting, you would most likely get a good enough compression ratio to save a lot of storage. However, accessing the data effectively depends on your use case and your queries. With this setup, you would always have to access the data by using the time dimension and subsequently filter all the rows based on any other criteria.


<CodeBlock canCopy={false} showLineNumbers={false} children={`
postgres=# select avg(cpu) from metrics where time >= '2024-03-01 00:00:00+01' and time < '2024-03-02 00:00:00+01';
        avg         
--------------------
 0.4996848437842719
(1 row)
Time: 87,218 ms
postgres=# ALTER TABLE metrics SET (timescaledb.compress, timescaledb.compress_segmentby = 'device_id', timescaledb.compress_orderby='time');
ALTER TABLE
Time: 6,607 ms
postgres=# SELECT compress_chunk(c) FROM show_chunks('metrics') c;
             compress_chunk             
----------------------------------------
 _timescaledb_internal._hyper_2_4_chunk
 _timescaledb_internal._hyper_2_5_chunk
 _timescaledb_internal._hyper_2_6_chunk
(3 rows)
Time: 3070,626 ms (00:03,071)
postgres=# select avg(cpu) from metrics where time >= '2024-03-01 00:00:00+01' and time < '2024-03-02 00:00:00+01';
       avg        
------------------
 0.49968484378427
(1 row)
Time: 45,384 ms
`} />
