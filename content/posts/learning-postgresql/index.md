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
![Postgres physical storage](./pics/postgres-physical-summary.svg)

[Open image](./pics/postgres-physical-summary.svg)



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

* **Write-Ahead Logging** (WAL) - All database state changes requiring high durability guarantees are logged before the associated data structures like heap and indexes are updated. Think of it as an append-only ledger recording all the changes done to the database. On crash, WAL is read to recover the state of the database and to rollback any of the uncommitted transactions.There are background threads/processes to flush changes to the disk and to truncate the logs so that it does not grow indefinitely and occupy all the space. As such, it is important for the WAL-entry to hit the disk before acknowledging the write. The directory `$PGDATA/pg_wal` contains the Write-Ahead log.

* **MVCC** - This is required for implementing certain isolation levels where each transaction would be working on a "certain snapshot/version of the database". Hence...The tables are [persistent data structures](https://en.wikipedia.org/wiki/Persistent_data_structure) on disk - i.e modifying/deleting data creates "new snapshot/version" of the table with the changes applied and transactions working on old version can proceed in many cases. This also enables querying on a snapshot. But...storing all versions wastes a lot of space and outdated tuples must be **garbage collected**.

* The **garbage collection** where old tuples are removed and the space used by them are reclaimed is called **Vacuuming**. [Autovacuum](https://www.postgresql.org/docs/current/runtime-config-autovacuum.html) is where this is automated. This is analogous to reclaiming unreachable memory in Garbage collected languages like Java, Golang etc. Hence, this also has similar issues like eating up CPU, kind of stop-the-world pauses (called `VACUUM FULL`).

* **Freezing** tuples is necessary to preserve MVCC semantics in the face of transaction ID wraparound. If freezing is not in-time, then the only option is to stop accepting writes and vacuum the table, which means downtime.

* **TOAST** is used to store long attributes that don't fit into a page. Depending on the configuration, Postgres can try to compress or store in uncompressed format. TOAST table is largely invisible to the user while querying and is created transparently. Nevertheless, the toast table associated with a relation can be found in `pg_class` table's `reltoastrelid` attribute. The particular TOAST strategy of a column can be found out by looking up `pg_attribute` table's `attstorage` attribute. Also note that reading a TOASTed attribute might cause **read amplication** as Postgres has to read a different table to serve the request. Different TOAST strategies
  * `p:PLAIN` - no compression or out-of-line storage. No toasting here.
  * `x:EXTENDED`(mostly default) - allows compression and storing in TOAST table. First compress and then move to TOAST table if the data is big.
  * `e:EXTERNAL` - does not allow compression, but allows storing in TOAST table. It is fast if the data is searched for substrings as only the chunks required can be looked up unlike `EXTENDED` strategy where the entire data has to be fetched, decompressed before such queries.
  * `m:MAIN` - compression, but out-of-line storage in TOAST table is only done as last resort.

## Data directory
Data storage is not magic. It is just a bunch of directories and files on the filesystem. Each table, index maps to certain number of files. As you may expect, the mapping should be stored somewhere persistently and that metadata is the **catalog**. In `psql`, the following command lists all such tables
```
=# \dt+ pg_catalog.* 
```
But where are all the files though? It is in the directory specified by `PG_DATA` or the `data_directory` in the datbase configuration file, say "postgresql.conf". From `psql`, it can be found out as follows
```
=# SHOW data_directory;
-[ RECORD 1 ]--+------------------------------------------------
data_directory | /home/suchith/workspace/postgresql/devtest/data
```
Navigating to the directory, we can see all the data stored by Postgresql
```
$ ls -l /home/suchith/workspace/postgresql/devtest/data
total 128
drwx------ 5 suchith suchith  4096 Dec 29 21:00 base
drwx------ 2 suchith suchith  4096 Dec 29 21:24 global
drwx------ 2 suchith suchith  4096 Dec 29 21:00 pg_commit_ts
drwx------ 2 suchith suchith  4096 Dec 29 21:00 pg_dynshmem
....# Output omitted
```

### Structure
Now, it is time to lookup the documentation for some of them to understand what they mean. I'll describe only some of them. The full list of directories is documented [here](https://www.postgresql.org/docs/current/storage-file-layout.html)

| File/Directory | Purpose |
|----------------|---------|
| PG_VERSION | File with major version number of Postgresql |
| base | Subdirectories containing per-database subdirectories. This is one of the places where postgres inserts the table data as part of the `INSERT` command. This is also one of the places where it looks up data as part of executing `SELECT` query. `base` is  just one of the [Tablespaces](#tablespaces): directories where data is kept. |
| global | Subdirectories containing global, database cluster-level data like `pg_database`. |
| pg_wal | Write-Ahead logging files. For durability and recovery purposes, all changes to the database are recorded in the WAL first before writing it down to the disk. A WAL is also a bunch of files inside the `PGDATA` directory. Think of it as a ledger recording all the changes, transaction status etc. On recovery during the start, WAL is consulted to make sure that the effects of the uncommitted transactions are rolled back on recovery during startup |
| pg_xact | Records transaction state. A transaction is said to be committed if the corresponding flag here is set. This is the authoritative source of transaction commit related data |
| pg_tblspc | Contains symbolic links to tables in the tablespsaces other than `global` and `base` which are inserted directly

### Tablespace (pg_tblspc)
A Tablespace is used for keeping data in different directories depending on the application needed. It has `global` and `base` by default and additional tablespaces can be created by the command shown below
```sql
CREATE TABLESPACE otherspace LOCATION '/home/suchith/workspace/postgrestblspace' 
```
It should be noted that the tablespace contains tables, not databases although the layout inside it is `database/table`. This is because there can be two tables in two different databases with the same name. Tablespace is a **purely physical concept** allowing users to take advantage of different speed and price of the underlying storage volume (could be a disk in the DC or the cloud provider). The relation can be associated with a tablespace on creation as follows.
```sql
CREATE TABLE X(i int, j bool) TABLESPACE otherspace;
```
Postgres accesses tablespaces through symbolic links from `PGDATA/pg_tblspc` directory. Let us peek into it, follow the path and see what's in there
```
$ ls -l pg_tblspc
total 0
lrwxrwxrwx 1 suchith suchith 40 Jan  2 20:00 16389 -> /home/suchith/workspace/postgrestblspace
```
The number `16389` is the oid (Identifier used by postgres internally) assigned by postgres for the tablespace we just created. That can be queried as follows
```
=# SELECT * FROM pg_tablespace WHERE spcname = 'otherspace';
-[ RECORD 1 ]----------
oid        | 16389
spcname    | otherspace
spcowner   | 10
spcacl     | 
spcoptions | 
```
Let us dive deeper. According to the documentation, inside the tablespace, there would be directories for the version of the database that created the tablespace. In my case it is `PG_16_202212241` which is unreleased (at the time of writing) postgres 16.
```
$ ls -lL pg_tblspc/16389
total 4
drwx------ 3 suchith suchith 4096 Jan  2 20:08 PG_16_202212241
```
Then, we see two more identifiers inside it - namely 5 and 16390.
```
$ ls -lL pg_tblspc/16389/PG_16_202212241/
total 4
drwx------ 2 suchith suchith 4096 Jan  2 20:08 5

$ ls -lL pg_tblspc/16389/PG_16_202212241/5
total 0
-rw------- 1 suchith suchith 0 Jan  2 20:08 16390
```
By now, you could guess that these are OIDs of the database and the table we just created in the tablespace respectively. But..Still let us check that once and you can see that it is the default `postgres` database that the system created for us when the database was installed and configured. For now, we don't need to worry about what all those attributes mean: only focus on `oid` and `datname`
```
# SELECT * FROM pg_database WHERE oid = 5;
-[ RECORD 1 ]--+---------
oid            | 5
datname        | postgres
datdba         | 10
encoding       | 6
datlocprovider | c
datistemplate  | f
datallowconn   | t
datconnlimit   | -1
datfrozenxid   | 720
datminmxid     | 1
dattablespace  | 1663
datcollate     | en_IN
datctype       | en_IN
daticulocale   | 
datcollversion | 2.35
datacl         | 
```
So...Is 16390 the OID of the table `X` we created in the `otherspace`? Let us verify that. This information is present in `pg_class` catalog table.
```
# SELECT oid, relname, reltablespace, relfilenode FROM pg_class WHERE oid = 16390;
-[ RECORD 1 ]-+------
oid           | 16390
relname       | x
reltablespace | 16389
relfilenode   | 16390
```
Indeed! But there is a catch. The actual file name is **not** necessarily the OID of the table, but `relfilenode`. You can also see the OID reference to its tablenamespace `reltablespace`. There is a function to do this. You can see the path we followed (However, the symlink is not shown) relative to `PGDATA`.
```
=# SELECT pg_relation_filepath('X');
-[ RECORD 1 ]--------+----------------------------------------
pg_relation_filepath | pg_tblspc/16389/PG_16_202212241/5/16390
```

So..We can now answer the question of how postgres figures out which file to pick? More precisely, we will derive the mapping from `X` to `pg_tblspc/16389/PG_16_202212241/5/16390`. This is a mental model and not necessarily what the code does.
* Lookup the OID of the database we are logged into from `pg_database` table. It is 5.
* Lookup `reltablespace`, `oid` and `relfilenode` for the table `X` (the actual lookup is more complicated).
* Then look for `database_oid/relfilenode` in different directories. It is not expected to have lots of directories.

So...Finally, we have `<pg_tblspc>/<tblspc_oid>/<pg_version>/<db_oid>/<relfilenode>`. There is more to this because a relation is actually a set of files called **segments** which, by default, is a 1GB chunk. So suffixes like `_1`, `_2` should be added to get the actual file. The file picked further depends on the **block number**. A **block** is a fundamental I/O unit (default: 8KB).

### Files and Forks

#### Main fork

#### Free-space map fork (_fsm)

#### Visibility map fork (_vm)

#### Initialization fork (_init)

### File segments

### Page layout

### Tuple layout
TODO:
1. Show with a diagram, how attribute order matters
2. How variadic data types look like?
3. How toasted attributes look like?

### Line pointers and types

### HOT optimization

### TOAST
TODO: Show linkage between main and TOAST table
#### Different TOAST options

#### When is it applied?

### Write-Ahead log (pg_wal)

#### WAL Record structure

#### Compacting and snapshot

### In-memory shared buffer-cache

### Transaction status metadata (pg_xact)

## Multi-Version Concurrency Control (MVCC) implementation

### Isolation levels

### MVCC update/delete and TOASTing
TODO: Determine relation size to see where it is accounted

### Garbage collection and vacuuming

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
