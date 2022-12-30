---
title: "Postgresql internals - Physical data organization"
date: 2022-10-23T20:00:17+05:30
draft: true
tags: [database,postgres,internals]
---

{{< toc >}}

## Introduction
This post assumes that you are familiar with Postgresql database, SQL, logical organization of data into database, tables, indexes, schema etc. This is NOT for absolute beginners. I won't be covering how to install, setup, write SQL queries etc. Please consult documentation for that. I also assume that you have a decent knowledge of algorithms and data structures: at least lists, balanced search trees etc.

In this post, I'll be exploring the fascinating details of the physical storage on disk in Postgres. That is - answering how the data is actually stored on-disk and fetched from the disk when you run different SQL queries. Please note that I'm using the development version of Postgres (16 at the time of writing this). It is the summary of various posts on the internet (see [References](#references) section) and also my own exploration of the Postgres source code (written in C and although decades old is very readable!).

**Note**: Read ["Summary for the impatient"](#summary-for-the-impatient) section if you don't have enough time.

**Disclaimer**: I am not a Postgresql expert, but just a developer interested in the internals of the software that I use day in and out. This must not be treated as expert advice or anything close to that. I'll strive to keep it accurate, but there could be some inaccuracies as new releases come out. Please create [github issue](https://github.com/su225/su225.github.io/issues/new) with the link to the post in case you find any inaccuracy or incorrect information.

## Summary for the impatient
* Think of database as supercharged "on-disk data structures".

* The directory represented by `PGDATA` environment variable or the `data_directory` in the config supplied at startup.

* Different objects in postgres like relation ID (tables, indexes, sequences), databases, types etc in the catalog are represented by type called `oid` which is a number.

* [Tablespace](https://www.postgresql.org/docs/current/manage-ag-tablespaces.html): The `tablespace` is the directory where data objects are stored. This can be used to control the disk layout of the data by the administrator.

* An **access method** defines how data is accessed. It can be a table or a particular index structure.

  * All access methods are in `pg_am` table of the catalog.
    ```
    =# SELECT * FROM pg_am;
     oid  | amname |      amhandler       | amtype 
    ------+--------+----------------------+--------
        2 | heap   | heap_tableam_handler | t
      403 | btree  | bthandler            | i
      405 | hash   | hashhandler          | i
      783 | gist   | gisthandler          | i
     2742 | gin    | ginhandler           | i
     4000 | spgist | spghandler           | i
     3580 | brin   | brinhandler          | i
    ``` 

  * **heap** - for sequential access of table data. This is where actual data is stored

  * **btree**(index) - [B-Tree](https://en.wikipedia.org/wiki/B-tree) data structure

  * **hash**(index) - Hash table implementation. See [Documentation](https://www.postgresql.org/docs/current/hash-index.html)

  * **GiST**(index) - Generalized search tree to implement arbitrary indexing schemes (B-Tree, R-Tree are specific examples). See [documentation](https://www.postgresql.org/docs/current/gist-intro.html)

  * **Gin**(index) - Generalized Inverted Index. It stores `(key, posting list)` pairs where a posting list is a set of row IDs in which the key occurs. Internally, it is a B-Tree index constructed over keys. See [Documentation](https://www.postgresql.org/docs/current/gin-intro.html)

  * **Brin**(index) - Block Range Index. It is for handling very large tables where the for a range of values for a key, the rows also reside in physically adjacent pages. Something like - "all keys starting with A reside in pages 1-100". This way, when "A" is encountered, we don't have to look for pages beyond 100. Here 1-100 is the "block range" for "A" 

  * **SP-GiST**(index) - Space-partitioned GiST for non-balanced data structures like [Quad-Tree](https://en.wikipedia.org/wiki/Quadtree), [k-d tree](https://en.wikipedia.org/wiki/K-d_tree), [radix trees](https://en.wikipedia.org/wiki/Radix_tree)

* **HOT optimization** - Indexes cause **write amplification** and decrease the write throughput. If the column updated is not indexed, then we might still create new index entry pointing to the correct version. If the updated tuple is also in the same page, then "Heap-Only Tuple" optimization does not create a new index entry, but instead updates the tuple representing the previous version of the tuple to point to the next version forming a chain within the same page. This saves a few writes, but at the cost of complexity in the vacuuming process where the pointer to the dead, but indexed tuple cannot be freed immediately.

* **Fork** - A relation (table or index) is stored in **a set of files** within a tablespace called fork
  * **main** - where actual data is stored.

  * **free-space map** - to locate a page with sufficient free space for a tuple.

  * **visibility map** - to check if all tuples in a page are "frozen" or "visible" (for implementing MVCC)

* **Page** - A fixed block (by default, 8KB) in a file. It is the unit of I/O. Each page consists of header, pointers and tuples. A tuple can represent data in case of "heap" (or actual table data) or a pointer to data in a heap (in case of indexes). A tuple is not the same as the row.

* **Binary storage format** - The data is stored and loaded as is, **without any serialization/deserialization** overhead. This means in-memory and on-disk formats are the same. But this means compatibility concerns - details like endianness (Big/Little), data alignment matters.

* **Tuple** - It can be actual data stored by the user or pointer to user data in an index. It also consists of additional fields like `xmin`, `xmax` which are required for implementing Multi-Version Concurreny Control (MVCC). **A tuple must fit in a page**. Attributes with long tuples could be stored separately and this is called [TOAST](https://www.postgresql.org/docs/current/storage-toast.html).

* **Buffer cache** - In-memory, shared (between different Postgres sub-processes running transactions etc) memory to avoid hitting the disk. Internally, it is an LRU cache implemented as a hash table.

* **MVCC** - This is required for implementing certain isolation levels where each transaction would be working on a "certain snapshot/version of the database". Hence...The tables are [persistent data structures](https://en.wikipedia.org/wiki/Persistent_data_structure) on disk - i.e modifying/deleting data creates "new snapshot/version" of the table with the changes applied and transactions working on old version can proceed in many cases. This also enables querying on a snapshot. But...storing all versions wastes a lot of space and outdated tuples must be **garbage collected**.

* The **garbage collection** where old tuples are removed and the space used by them are reclaimed is called **Vacuuming**. [Autovacuum](https://www.postgresql.org/docs/current/runtime-config-autovacuum.html) is where this is automated. This is analogous to reclaiming unreachable memory in Garbage collected languages like Java, Golang etc. Hence, this also has similar issues like eating up CPU, kind of stop-the-world pauses (called `VACUUM FULL`).

* **Freezing** tuples is necessary to preserve MVCC semantics in the face of transaction ID wraparound. If freezing is not in-time, then the only option is to stop accepting writes and vacuum the table, which means downtime.

* **TOAST** is used to store long attributes that don't fit into a page. Depending on the configuration, Postgres can try to compress or store in uncompressed format. TOAST table is largely invisible to the user while querying and is created transparently. Nevertheless, the toast table associated with a relation can be found in `pg_class` table's `reltoastrelid` attribute. The particular TOAST strategy of a column can be found out by looking up `pg_attribute` table's `attstorage` attribute. Also note that reading a TOASTed attribute might cause **read amplication** as Postgres has to read a different table to serve the request. Different TOAST strategies
  * `p:PLAIN` - no compression or out-of-line storage. No toasting here.
  * `x:EXTENDED`(mostly default) - allows compression and storing in TOAST table. First compress and then move to TOAST table if the data is big.
  * `e:EXTERNAL` - does not allow compression, but allows storing in TOAST table. It is fast if the data is searched for substrings as only the chunks required can be looked up unlike `EXTENDED` strategy where the entire data has to be fetched, decompressed before such queries.
  * `m:MAIN` - compression, but out-of-line storage in TOAST table is only done as last resort.

## Setting up the source code for reading
I am running Linux (Ubuntu LTS to be more precise)

Requirements (not an exhaustive list)
* VSCode or your favorite IDE - for browsing through the source code with C/C++ plugins and interactive debugging sessions.
* GDB - for debugging: setting breakpoints, walking through the code. Make sure to set these options as Postgresql uses **multi-process architecture**.
  * `detach-on-fork off`: Don't detach the child process from the debugger so that we can debug child processes. This is important as connecting to the database server with `psql`, autovacuum and WAL writer as separate processes. Even transactions run as separate processes.
  * `follow-exec-mode new`: Follow the child process in case `exec()` is run.

Run `configure` script with the flags to enable debugging and assertions. I am setting custom prefix so that we can install it safely to the directory specified. It is important in case you already have a released version of Postgresql installed. Add `out` folder to `.git/info/exclude` to exclude it from git without changing `.gitignore`.
```sh
./configure --prefix="$(pwd)/out" \
			--enable-debug
			--enable-cassert
```
The build and install as follows
```sh
make
make install
```
Simple...Isn't it?

## References
1. [Internals book by Egor Rogov (Postgres professional)](https://edu.postgrespro.com/postgresql_internals-14_parts1-4_en.pdf) - Highly recommend this.
1. [interdb.jp - Also explores storage internals](https://www.interdb.jp/pg/)
1. [Postgres documentation](https://www.postgresql.org/docs/current/catalogs.html)
1. [Fighting PostgreSQL write amplification with HOT updates - Adyen blog](https://www.adyen.com/blog/postgresql-hot-updates)
1. [Transaction ID wraparound in Postgres - sentry.io blog](https://blog.sentry.io/2015/07/23/transaction-id-wraparound-in-postgres/)
