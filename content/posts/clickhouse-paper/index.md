---
title: "Paper notes: Clickhouse - Lightning Fast Analytics for Everyone"
date: 2024-09-14T20:00:17+05:30
draft: true
tags: [database,paper-notes]
---

[Link to the VLDB paper](https://www.vldb.org/pvldb/vol17/p3731-schulze.pdf)

{{< toc >}}

## Summary

* Distributed, analytical database optimized for high ingestion rates
* Besides horizontal scalability, all major decisions revolve around
    1. High data ingestion rates
    2. Fast query execution

![High-level architecture (from section-2)](./images/architecture-section-2.png)
From section-2 of the paper

* Columnar database with [LSM tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree) based table storage: called `MergeTree` in the paper.
* Highly optimized for the use case where the data is ingested once and is queried, but
updates and deletes are non-atomic with the concurrent queries.
* It is NOT fully ACID compliant. Hence, **good for use cases where the small risk of losing new data is tolerable**.
* Vectorized query execution engine to maximize parallelism
    * Table shards are processed in parallel by multiple nodes
    * Data chunks within a node are processed by multiple cores
    * Individual data elements are processed by SIMD operators
    * Compile queries with LLVM to make them even faster


## Physical storage layer: MergeTree table engine
* Each table is organized as a collection of **immutable** "parts"
* A part is created whenever a set of rows is added and bulk inserts are recommended to avoid
  creating lots of small parts.
* A part is self-contained and has all the metadata required to interpret the contents without
  looking up the central metadata store.

![Part directory with column files](./images/table-file-organization.png)

* Typical in LSM trees: A background job merges multiple parts into a single large part until
  the size reaches a configurable threshold. The columns are sorted in the order of the primary
  key column and hence, an efficient k-way merge algorithm is used.

### Parts, Blocks, Granules
* Part corresponds to a filesystem directory
* Usually contains **one file per column**
* Rows within a part are further divided into groups of 8192 records, called **Granules**
* **Blocks** are basic read/write unit and consist of multiple granules
    * Several neighboring granules are combined to form a block which is around 1MB by default
    * The number of blocks in the Granule is variable and depends on the column's data type and
      the distribution of values. For instance, it is possible to have fewer granules in a block
      if the column consists of lots of long strings because each value of the column takes up
      more space.
    * Smallest unit which is compressed. Various compression algorithms like LZ4, Gorilla, FPC
      (for floating-point data) are used and multiple algorithms can be chained.
    * Scan and index lookup operators process granule level. But the actual read/write of the
      table column files happen at the block level.
* Enabling random access to granules despite compression
    * Maintain granule ID to offset of the block in the compressed table file
    * Maintain granule offset within the uncompressed block

![Column file organization](./images/column-file-organization.svg)


### Special encoding of columns
* Enabling dictionary encoding with a special wrapper type `LowCardinality(T)` for the columns
  where the number of values is low. Dictionary encoding replaces the values with integers to
  reduce the size of the column and to enable better compression in the block compression step

TODO: Show dictionary encoding in picture

* The `Nullable(T)` type adds an internal bitmap to column T to compactly represent NULL values.
  This is a common trick in the database engines for space-efficient NULL value storage.

TODO: Show nullable bitmap in picture

### Table partitioning
Distributing across the nodes is required to **horizontally scale** the amount of data
a table can hold and also to improve the query execution speed by distributing processing
across them. Clickhouse supports various common partitioning strategies to distribute the 
table data over multiple nodes: range, hash, round-robin and custom partition expression.

It also supports creating advanced column statistics for providing cardinality estimate
of each partition (and these are important for **data pruning** which greatly improves 
query performance by eliminating the table partitions that need not be looked up)
* Approximate set cardinality with HyperLogLog
* Approximate distribution with T-digest

### Additional structures for data pruning
One of the two design goals of Clickhouse is fast query processing. To get to the result as
soon as possible, it is important to narrow down the set of rows/parts/granules to be checked very early on in the query execution efficiently.

#### Sparse index of primary key to granule
Map keyed by the value of the first row of the granule of primary key column to the granule ID. This index is sparse and can easily fit into memory. One entry addresses 8192 rows and hence 1000 entries are enough to address ~8.1 million rows.

This is loaded into memory and is used to decide what granules can be skipped. Ultimately, skipping granules or whole blocks saves on the expensive disk I/O and decompression costs.

#### Table projections
* Alternative version of the table where the rows are sorted by a different primary key
* As projections are sorted by the different column, the queries filtering on that column are sped up
* This is similar to creating indexes except that the data is also stored alongside making it a copy of the table with different data organization

#### Skipping indexes
* Lightweight alternative to table projections
* Store small amounts of metadata at the level of multiple granules (configurable)
* Available skipping index types

| Index type | Description |
|------------|-------------|
| Min-max indexes | storing minimum and maximum values for the range covered by the index |
| Set indexes | storing the set of values in the index block. Good when the cardinality is small and bounded |
| Bloom filter | can be created by a configurable false positive rate. As bloom filters are compact, they can be loaded into memory and can be checked to determine if the queried key definitely does NOT exist |

### Merging different parts
For efficient query processing, the number of parts have to be kept small to avoid jumping around. Like many LSM tree implementations, there is a background worker responsible for merging the parts into bigger ones until it hits some maximum size. Clickhouse supports different methods

| Merging method | Description |
|----------------|-------------|
| Replacing | Keep the most recent "version" of the row. By default it is the creation timestamp, but custom "version" column can be computed and specified to customize the behavior |
| Aggregating | More general version of replacing merge. Partial aggregation states like `sum()` and `avg()` are combined |
| TTL | For use cases like moving historical data to archival, another (possibly slower) volume, re-compress with a different, much more storage-efficient codec or even delete the part |

## Data Replication

## Query processing