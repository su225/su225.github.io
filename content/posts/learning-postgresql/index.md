---
title: "Diving into the internals of Postgresql"
date: 2022-10-23T20:00:17+05:30
draft: true
tags: [database,postgres]
---

Materials
* [Internals book by Egor Rogov (Postgres professional)](https://edu.postgrespro.com/postgresql_internals-14_parts1-4_en.pdf)
* [InterDB](https://www.interdb.jp/pg/)
* [Postgres documentation](https://www.postgresql.org/docs/current/catalogs.html)

{{< toc >}}

## System Catalog
* Keeps all the information related to all the objects in the cluster like databases, tables, indexes, functions etc.
* Many are common to all the functions
* They are accessible through SQL and the table names start with `pg_`
* `oid` - name of the primary key and its type in all the system catalog tables. Internally, it is a 32-bit integer.
* [Documentation on system catalog tables](https://www.postgresql.org/docs/current/catalogs.html)
* Why is this needed? It keeps all the metadata associated with the database. It is required for various stages of query execution to look up name of the tables, indexes.
* Some important catalog tables
  * `pg_class` - contains information about all the objects in the database.
  * `pg_database` - contains the list of databases.
  * `pg_index` - contains information about indices.
  * `pg_attribute` - contains information about attributes of a relation.

## Databases and Schemas
* Database and Schema are **logical** constructs.
* Schemas are namespaces that store all objects of a database.
* A database consists of multiple schemas, each containing tables, indexes, types etc.
* Two objects (like tables) in the same schema don't conflict.

![Database logical structure [credits: interdb.jp](https://www.interdb.jp/pg/pgsql01.html)]

#### [Hands-on] List databases
```
suchith=# select oid, datname from pg_database;
  oid  |  datname  
-------+-----------
     5 | postgres
 16389 | suchith
     1 | template1
     4 | template0
```

#### [Hands-on] List schemas
```
suchith=# select * from pg_namespace;
  oid  |      nspname       | nspowner |                            nspacl                             
-------+--------------------+----------+---------------------------------------------------------------
    99 | pg_toast           |       10 | 
    11 | pg_catalog         |       10 | {postgres=UC/postgres,=U/postgres}
  2200 | public             |     6171 | {pg_database_owner=UC/pg_database_owner,=U/pg_database_owner}
 13224 | information_schema |       10 | {postgres=UC/postgres,=U/postgres}
(4 rows)
```

#### [Hands-on] Querying for attributes of a relation
```sql
SELECT attname, atttypid::regtype,
  CASE attstorage
    WHEN 'p' THEN 'plain'
    WHEN 'e' THEN 'external'
    WHEN 'm' THEN 'main'
    WHEN 'x' THEN 'extended'
  END AS storage
FROM pg_attribute
WHERE attrelid = 'employee'::regclass
  AND attnum > 0;
```
* The above query lists only the attributes defined with `CREATE TABLE`. 
* You can remove `attnum > 0` to see the full list of attributes including `xmax`, `xmin` which are needed to implement Multi-Version Concurrency Control (MVCC).
* `storage` is related to TOAST (related to how data is stored physically).

## Physical database layout
* `PGDATA` directory consists of directory for each tablespace.
* Tablespace directory contains directory for each database.
* Each database directory consists of data files for each of the database objects.
* Each relation is represented by one or more files, together called **forks**.
* Each file is divided into minimum I/O blocks (8KB by default) called **page**.
* Different fork types: main, initialization, free-space-map, visibility-map.
* `pg_relation_filenode` returns the filenode number for a given relation.
* `pg_relation_filepath` returns the filepath relative to `PGDATA` directory.
* Each row must fit a single page. Postgres does not store a row in multiple pages. The contents of long columns are stored in a separate table called **TOAST** tables. TOAST stands for "The Oversized Attributes Storage Technique".
  * There are several TOAST strategies offering compression.
  * Pluggable TOAST APIs are also supported.
  * It limits the key size in the index as only compression is supported.

### Tablespace
Tablespace is a **physical** construct (A directory in the filesystem). The objects can be distributed across tablespaces depending on the IOPS requirement for the underlying data. When creating tables, the tablespace can be selected.

List tablespaces
```
suchith=# \db+
                                  List of tablespaces
    Name    |  Owner   | Location | Access privileges | Options |  Size  | Description 
------------+----------+----------+-------------------+---------+--------+-------------
 pg_default | postgres |          |                   |         | 29 MB  | 
 pg_global  | postgres |          |                   |         | 531 kB | 
(2 rows)
```

Where is data stored? Postgres data directory is defined by `PGDATA`. In my system (Ubuntu), it is in `/var/lib/postgresql/15/main` directory.
```
postgres@su225:~/15/main$ ls -l
total 84
drwx------ 6 postgres postgres 4096 Dec 19 19:43 base
drwx------ 2 postgres postgres 4096 Dec 19 19:43 global
...
```
The above command lists even more directories. See [here](https://www.postgresql.org/docs/current/storage-file-layout.html) for what each of those directories mean. 
* `base` is the tablespace `pg_default` which is used when tablespace is not explicity specified. 
* `global` contains the files related to global tables like `pg_database`.

Peeking inside `base` shows the directory for the database. Notice the `oid` corresponds to the OIDs of the databases.
```
postgres@su225:~/15/main$ ls -l base
total 16
drwx------ 2 postgres postgres 4096 Dec 19 19:41 1
drwx------ 2 postgres postgres 4096 Dec 19 19:43 16389
drwx------ 2 postgres postgres 4096 Dec 19 19:38 4
drwx------ 2 postgres postgres 4096 Dec 19 19:41 5
```

### Table files and forks
* All relations are stored in different **forks** each containing a particular type.
* Files are divided into **pages**
* A **page** is a minimum amount of data that can be read or written.
* A **fork** consists of several files.
  * Initially it is a single file with the filename being numeric oid called **filenode number**.
  * The file grows and when it hits 1GB, another file of this fork is created, called **segment**.
  * The sequence number of the segment is appended to the filename.
* There are different types of forks
  1. **main** - contains actual data
  2. **initialization** - for `UNLOGGED` tables where operations on them are not written to the Write-Ahead Log (WAL). During recovery, data in the main fork is replaced by the data in the initialization fork as it is not possible to recover the data without WAL. It is suffixed `_init`
  3. **freespace-map** - keeps track of available spaces within pages (a file is divided into pages - 8KB by default, although that can be changed during the build time). It is suffixed `_fsm`.
  4. **visibility-map** - two bits per page. It shows whether vacuuming needs to be performed or frozen. It is suffixed `_vm`.
* **filenode number** that can be found in `pg_class.relfilenode`.

#### [Hands-on] Locating data files - `filenode` and `filepath`
Let us create a table and locate its files
```sql
CREATE TABLE employee (
  id INT PRIMARY KEY,
  name VARCHAR(50)
);
```
It is created in the default tablespace, in the default database for the user. Hence, its data goes in `$PGDATA/base/16389`. Then we lookup its `oid` and filenode number using the following query
```
suchith=# select oid, relname, reltype, relfilenode from pg_class where relname = 'employee';
  oid  | relname  | reltype | relfilenode 
-------+----------+---------+-------------
 16390 | employee |   16392 |       16390
(1 row)
```
Now checking in the directory
```
$ ls -l base/16389/16390
-rw------- 1 postgres postgres 0 Dec 19 21:05 base/16389/16390
```

This can also be found out as follows
* Finding `filepath` relative to `PGDATA` directory
```
suchith=# select pg_relation_filepath('employee');
 pg_relation_filepath 
----------------------
 base/16389/16390
(1 row)
```
* Finding `filenode` within the `<tablespace>/<database-oid>` directory
```
suchith=# select pg_relation_filenode('employee');
 pg_relation_filenode 
----------------------
                16390
(1 row)
```

## TOAST for long attributes
* TOAST is for storing attributes which are long and don't fit in a page.
* Postgres tries to fit at least 4 tuples in a page. If the tuple does not fit, then some rows are moved to the TOAST table. The threshold is `2000B` by default, but can be redefined at the table level using `toast_tuple_target` storage parameter.
* TOAST tables are usually hidden and reside in a separate schema called `pg_toast`.
* Recursive "TOAST" is not supported.
* TOAST table for a relation is in `pg_class.reltoastrelid`.
* TOAST table increases the minimum number of fork files used by the table up to 8.
  * 3 for the table.
  * 3 for the TOAST table.
  * 2 for the TOAST index.

| TOAST strategy| Code | Description |
|---------------|------|-------------|
| plain | p | TOAST strategy is not used |
| extended | x | Allows both compressing attributes and storing them in a separate TOAST table |
| external | e | Long attributes are stored in the TOAST table in an uncompressed state |
| main | m | Long attributes are compressed first and they will be moved to the TOAST table only if the compression does not help |

### When is it stored in the TOAST table?
1. Go through the attributes with external and extended strategies, starting with the longest ones. Extended attributes get compressed and if the resulting value exceeds 1/4th of the page then it is moved to the TOAST table straight away. External attributes are handled in the same way, but without compression.
2. If the row does not fit in the page after the first pass, then we move the remaining attributes that use external or extended strategies into the TOAST table.
3. If that does not help, then compress the attributes that use the "main" strategy and keep them in the table page.
4. If the row is still not short enough, the main attributes are moved into the TOAST table.

### Illustration of TOAST table in action
1. Create a table called `toasttest` for testing
```sql
CREATE TABLE toasttest(
  a INTEGER,
  b NUMERIC,
  c TEXT,
  d JSON
);
```
2. Check the storage type with the query mentioned before
```sql
SELECT attname, atttypid::regtype, attstorage
FROM pg_attribute
WHERE attrelid = 'toasttest'::regclass
  AND attnum > 0;
```
It displays the following
```
 attname | atttypid | attstorage 
---------+----------+------------
 a       | integer  | p
 b       | numeric  | m
 c       | text     | x
 d       | json     | x
(4 rows)
```
3. You can change the storage type as follows
```sql
ALTER TABLE toasttest
  ALTER COLUMN d SET STORAGE external;
```
Then run the query on `pg_attribute` again
```
 attname | atttypid | attstorage 
---------+----------+------------
 a       | integer  | p
 b       | numeric  | m
 c       | text     | x
 d       | json     | e
(4 rows)
```
4. When the table is created, it also creates TOAST table automatically. This is because there are 3 potentially long attributes - b, c, d. If at least one attribute is potentially long, then postgres creates a TOAST table right away during table creation.
```sql
SELECT relnamespace::regnamespace, relname
FROM pg_class
WHERE oid = (
  SELECT reltoastrelid
  FROM pg_class
  WHERE relname = 'toasttest'
);
```
Toast table
```
 relnamespace |    relname     
--------------+----------------
 pg_toast     | pg_toast_16395
(1 row)
```
4.a. Peek into its structure
```
\d+ pg_toast.pg_toast_16395

TOAST table "pg_toast.pg_toast_16395"
   Column   |  Type   | Storage 
------------+---------+---------
 chunk_id   | oid     | plain
 chunk_seq  | integer | plain
 chunk_data | bytea   | plain
Owning table: "public.toasttest"
Indexes:
    "pg_toast_16395_index" PRIMARY KEY, btree (chunk_id, chunk_seq)
Access method: heap
```
4.b. Check the corresponding TOAST index
```sql
SELECT indexrelid::regclass
FROM pg_index
WHERE indrelid = (
  SELECT oid
  FROM pg_class
  WHERE relname = 'pg_toast_16395'
);
```
Which gives
```
          indexrelid           
-------------------------------
 pg_toast.pg_toast_16395_index
(1 row)
```
4.c. Peeking into the index
```
\d+ pg_toast.pg_toast_16395_index
              Index "pg_toast.pg_toast_16395_index"
  Column   |  Type   | Key? | Definition | Storage | Stats target 
-----------+---------+------+------------+---------+--------------
 chunk_id  | oid     | yes  | chunk_id   | plain   | 
 chunk_seq | integer | yes  | chunk_seq  | plain   | 
primary key, btree, for table "pg_toast.pg_toast_16395"
```

5. Now insert a long value for attribute `c` with the same character repeated multiple times. As `attstorage` is `extended`, postgres first tries to compress to see if it fits in the same page. In this case, it does and no toast record should be created. This stops at step (1) of the TOAST algorithm mentioned before.
```
suchith=# INSERT INTO toasttest VALUES (10, 5.5, repeat('A',10000), '{}');
INSERT 0 1

suchith=# SELECT * FROM pg_toast.pg_toast_16395;
 chunk_id | chunk_seq | chunk_data 
----------+-----------+------------
(0 rows)

```
Although `c` is 10,000 characters long (which is more than a page), it can be compressed easily as characters are repetitive. However, when characters are random, this cannot be done. Hence, the value of the attribute would be moved to the corresponding TOAST table.

6. Now insert a long random string which is hard to compress. This should be seen in the TOAST table.
```sql
INSERT INTO toasttest VALUES (
  20,
  8.7,
  -- generates a string of length 8000 with random uppercase characters.
  -- 65 is the ASCII code of 'A' and random() generates a uniformly random
  -- number between 0 and 1. trunc() truncates the output to some decimal
  -- digits (0 if it is not specified). chr() converts number to the corresponding
  -- character in the defined encoding. string_agg() joins strings according to
  -- the given delimiter: here just blank. generate_series() generates rows from
  -- [start,stop]: this is like range() function in python.
  (SELECT string_agg(chr(trunc(65+random()*26)::integer), '')
   FROM generate_series(1,8000)),
  '{}'
);
```
7. Now, we will check the corresponding TOAST table
```sql
SELECT chunk_id, chunk_seq, pg_column_size(chunk_data)
FROM pg_toast.pg_toast_16395;
```
It shows
```
 chunk_id | chunk_seq | pg_column_size 
----------+-----------+----------------
    16400 |         0 |           2000
    16400 |         1 |           2000
    16400 |         2 |           2000
    16400 |         3 |           2000
    16400 |         4 |             20
(5 rows)
```
* `chunk_id` is the same for all of them indicating that all the chunks are part of the same attribute value. 
* `chunk_seq` is for ordering the chunks within the same `chunk_id`.
* `chunk_data` contains the actual data.
* `pg_column_size` function shows the size of the column in bytes. This is used here for illustration purposes as the actual data is very long.