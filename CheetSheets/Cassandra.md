#-#
#-# Introduction
#-# 

#-# History

Originally created at Facebook.
Open sourced and now an official apache project.

#-# When to use Cassandra

Cassandra is a great choice for a database when...
- speed is the top goal and eventual consistency is acceptable.
- table schema is stable and unlikely to change.

#-# Overview

Apache Cassandra is a high-performance, extremely scalable, fault tolerant (i.e., no single point of failure), distributed non-relational database solution.
Cassandra combines all the benefits of Google Bigtable and Amazon Dynamo to handle the types of database management needs that traditional RDBMS vendors
cannot support. DataStax is the leading worldwide commercial provider of Cassandra products, services, support, and training.



#-#
#-# Terminology
#-#

Term				Meaning
----				-------
node				A server running an instance of cassandra.
token				A single token associated with a piece of data. "Token" is also often used to refer to a range of tokens.
vnode				A virtual node. Virtual nodes are used to distribute token values on a single physical node.
snitch				Snitches are used by cassandra internally to learn about the nodes. See SimpleSnitch, GossipingPropertyFileSnitch, PropetyFileSnitch, etc.
dc				A data center is a grouping of nodes. Each data center has its own set of tokens.
rack				Each node in a data center is part of a rack. Racks do not require an even distribution of nodes. Data center replication will be evenly distributed among racks. Each rack will evenly distribute the replication among the nodes on the rack.
keyspace			Data is organized at the highest level into keyspaces. The oracle or mysql analog is tablespace. Replication is specified at the keyspace level.
table				Pretty close to mysql definition - columns are defined, and rows of data are stored in the table.
partition			All data is associated with a partition key. A table contains partitions.
row				Also close to mysql definition. Rows represent data within paritions.



#-#
#-# Architecture
#-#

Cassandra nodes are typically represented in a ring because there is no master node - it is a true peer to peer system.
All data stored in Cassandra is associated with a token value.
Each data center has a set of token values, and each node in the data center is assigned to manage a portion of the token values. 
Virtual nodes can exist within each node and distribute the load of token values somewhat randomly, making it so that each vnode is replicated on a different node.

Property			Relational database 								Cassandra
--------			-------------------								---------
Atomicity			Enforced at every scope, at the cost of performance and scalability.		Success or failure on a row-by-row basis.
Sharding			Often forced when scaling, partitioned by key or function			Native share-nothing architecture, inherently partitioned by a configurable strategy.
Consistency 			Favors consistency over availability tunable via isolation levels.		No consistency in the ACID sense. Can be tuned to provide consistency in the CAP sense--data is consistent across all the nodes in a distributed database cluster ,guaranteeing read-after-write or eventual readability.
Durability			Typically, data is written to a single master node, sometimes configured	Writes are durable to a replica node, being recorded in memory and the commit log before acknowledged. In the event of a crash, the commit log replays on restart to recover any lost writes before data is flushed to disk.
				with synchronous replication at the cost of performance and cumbersome 
				data restoration.
Multi-Datacenter		Typically only limited long-distance replication to read-only slaves		Native capabilities for data replication over lower bandwidth, higher latency, less reliable connections.
Replication 			receiving asynchronous updates.
Security			Fine-grained access control to objects.						Coarse-grained and primitive.


#-# Keyspace

Replication strategy is defined at the keyspace level.

# SimpleStrategy
The replication_factor tells cassandra how many copies of each partition in the keyspace to store.
create keyspace <keyspace-name> with replication = {'class': 'SimpleStrategy', 'replication_factor': 3};

# NetworkTopologyStrategy
When using NetworkTopologyStrategy, you specify how many copies of each partition to store in each data center.
create keyspace <keyspace-name> with replication = {'class': 'NetworkTopologyStrategy', '<dc-name>': 3, '<dc-name>': 2};


#-# Tables

# Partition keys
The primary key is defined as the first partition key.
A single primary key can be specified using CREATE TABLE my_table (id varchar PRIMARY KEY, title varchar); or CREATE TABLE my_table (id varchar, title varchar, PRIMARY KEY (id)).
A composite primary key is specified using CREATE TABLE my_table (id varchar, title varchar, name varchar, PRIMARY KEY ((id, title)));

# Clustering keys
A composite key is specified using PRIMARY KEY(partition_key, clustering_key). 
For example: CREATE TABLE my_table (id varchar, title varchar, name varchar, PRIMARY KEY (id, title));

When clustering keys are used, the data inserted can be visualized as an entirely separate row. 
All of the common data must be inserted each time unless static columns are used.

A partition can hold a maximum of 2 billion cells. All of the data from a single partition must fit on a single node in the cluster. 
To address these limitations, consider ways to appropriately bucket the data. For example, when storing timestamps for events, 
buckets could be created for year, month, and day, depending on the volume that is anticipated. 
Another approach would be to create a bucket_id key that is a string of your choosing (e.g. "2016-12-07").

# Static
The STATIC modifying makes static fields associated with the partition key rather than including all of that data in each of the clusterking key rows.

# Secondary index
A secondary index allows you to query based on columns that are not part of the primary or clustering keys.
Do not use a secondary index when:
- the column has high (or very low) cardinality. (Cardinality is a measure of the number of elements in a set.) e.g., it would not be a good idea to make an index on an email column, where every entry will be unique. Also not a good idea to make an index on a boolean column where each entry will be one of two values.
- the table contains a counter column.
- the column is frequently updated or deleted.
- the table has very large partitions.
- Cassandra currently does not support indexing static columns.
- in cases where a secondary index cannot, or should not, be used, a separate table can be used as a manual index. For example, if you have a courses table with a static tags collections and you want to be able to query for courses by tag, you can create a separate table that has a primary key of (tag, course_id) which can serve as an index.


#-# Reads and Writes

Each node in the cluster functions the same. 
The node that you connect to when making a query becomes the 'coordinator node'. 
Nodes that get written to in a statement are called 'replica nodes'.

# Single data center consistency options
Option			Description
------			-----------
ONE			Write/read succeeds after writing/reading data to/from a single node.
TWO			Write/read succeeds after writing/reading data to/from two nodes.
THREE			Write/read succeeds after writing/reading data to/from three nodes.
QUORUM			Write/read succeeds after writing/reading data to/from a majority of the available nodes.
ALL			Write/read succeeds after writing/reading data to/from all of the available nodes.
ANY			Write/read succeeds after writing/reading data to/from any of the available nodes (including just making it to the coordinator node).

# Multiple data centers consistency options
Option			Description
------			-----------
EACH_QUORUM		A quorum succeeds in each data center before returning success
LOCAL_QUORUM		A quorum succeeds in the data center that contains the coordinator node.
LOCAL_ONE		Same as ONE, but only looks in the data center of the coordinator node.

If a node is down when a write occurs so that the node misses data, the data will be repaired during a read.
To see what the default consistency is, run consistency;
To set the default consistency to use from the command line, use consistency <level>; e.g. consistency quorum;
If the consistency required cannot be achieved, the read or write will fail.


#-# Batches

A batch is not a transaction - it does not have rollback support. 
A batch does guarantee that every command in the batch will be executed. 
Batches are useful for keeping related data in sync.


#-# Tombstones

When something is deleted in Cassandra, a tombstone is created. 
The tombstone is treated just a like a write, so that nodes that are down can be repaired. 
When the node that was down when the delete occurred comes back online, the normal read repair process will update the node with the tombstone.

Tombstones are automatically garbage collected on a configurable interval (default is 10 days). 
It is important not to clean up tombstones too quickly, because if the tombstones are removed, and then a node that was offline when the tombstone was 
created comes back online, the the read repair will restore the data from the node that was offline when the delete occurred, making it so that the data 
that was supposed to be deleted comes back into the database.



#-#
#-# What’s happening inside Cassandra?
#-#

#-# On write

Request goes to coordinator node. Cassandra may have multiple coordinators.
- hash value of the (Composite) Partition Key is calculated based on the chosen Partitioner.
- nodes to which the data needs to be written are determined based on the replication, snitch and the range of data the node is responsible for (pre-determined based on num_tokens or vnodes).
- coordinator waits for one or more nodes to respond (based on the consistency level) before responding back to the client.
- if one or more of the data nodes are down, determined by gossip, or if the node fails to respond, the coordinator may write a hint and will retry when the node comes back. Write is deemed successful if the required consistency level is met.
- for BATCH requests, request is stored as a blob in the batchlog before being processed. This is avoided when using UNLOGGED.

Inside the data node:
- data is written to commit log if DURABLE_WRITES is true.
- a copy is written to the Memtable
- background activities
 -- commit log sync to disk based on commitlog_sync
 -- memtable flushed to disk as SSTable based on commitlog_total_space_in_mb
 -- compaction brings together latest values of columns spread across SSTables to one SSTable. This is required because Cassandra does not read before write before INSERT or UPDATE.
- write may fail if Memtable flush queue is having memtable_flush_queue_size Memtables.
- tombstones mark data as deleted
 -- marker on a row indicating if a column is deleted
 -- actual delete happens during compaction
 -- if you have a delete heavy workload, they may cause performance degradation if using range reads and compaction is infrequent


#-# On read

Request goes to coordinator node.
- coordinator determines the required nodes based on the snitch, hash of the Partition Key and required consistency level.
- conflict resolution based on latest timestamp of the given value.

Inside the data node:
- consult bloom filters for the probability of finding data in a table
- check if key offset is cached, if not, determine key from SSTable index
- fetch latest version of the data (updates would have caused multiple versions spread across SSTables), uncompress if required and return results; ignore values which have tombstones set.


#-# Background activity

Gossip
- determines which nodes are up or down
- determines health and load of the nodes

Transfer request data between coordinator and data nodes.
Transfer data between nodes when new nodes are added or nodes are being decommissioned.
Hinted handoff
Nodetool repair
- required if node is down for more than max_hint_window_in_ms


#-# Logical storage

ColumnFamily data model similar to Google’s Bigtable.

Think of it as a MySQL/SQLite index or a map of a map: an outer map keyed by a partition key, and an inner map keyed by a column key. Both maps are sorted.
- SortedMap<PrimaryKey, SortedMap<ColumnKey, ColumnValue>>

Supports upto 2 Billion columns
- Millions recommended
- Split rows if necessary, via buckets or other natural keys like “year,month,date”

SSTables are immutable

Keyspace
- contains Tables/ColumnFamily and defines the schema
- Determines replication

Tables store column names and values
- Partition key determines unique physical row
- Primary key determines unique logical CQL row
 -- Primary key has partition key and optional clustering columns
- Clustering columns causes the physical row to expand, encompasses multiple logical CQL rows
- Static column stores a unique value per physical row


#-# Physical storage

Commit log witten to commitlog_directory
SSTables written to data_file_directories



#-#
#-# CQLSH
#-#

The cqlsh command line utility allows for interacting with Cassandra directly.

Command								Description
-------								-----------
cqlsh								Launch the shell and connect to the default node. Default is localhost:9160; this can be changed by setting $CQLSH_HOST and/or $CQLSH_PORT
cqlsh -h							Display the help docs.
cqlsh <address>							Connect to a remote node. The address can be an ip address or a url.
cqlsh <address> -u <user>					Connect to a remote with the specified user. This will open up an interactive prompt if a password is required.
cqlsh <address> -u <user> -p <pass>				Authenticated remote login. Note that password will be visible as plain text on console if -p is used.
cqlsh -k <keyspace>						Authenticate to the provided keyspace.
cqlsh -f <file-path.cql>					Execute commands from the file and then exit.
cqlsh -e "<cql-command>"					Execute a CQL command and then exit. For example, cqlsh -e "SELECT id FROM sample_keyspace.my_table"
cqlsh --cqlversion=<version>					Use the specified version of CQL. For example, cqlsh --cqlversion=3.0.3

Once inside of a cqlsh prompt, these commands are available:

Command 							Description
-------								-----------
help								Show the help docs.
use <keyspace>							Use the specified keyspace. Allows accessing tables without prefixing with keyspace..
tracing <on/off>						Turn tracing on or off
desc								Describe a keyspace or table.
source <filepath>						Execute a file containing CQL statements.
expand <on/off>							If expand is on, each row printed to the console will be on its own line. Much easier to read!
nodetool							Useful tool for interacting with nodes. The nodetool command can be run from cqlsh on any node in the cluster and will yield the same results.
nodetool help							Display help docs.
nodetool status							Shows the status of the cassandra nodes in the cluster
nodetool status <keyspace>					Shows the status of the cassandra nodes that own the specified keyspace.
nodetool ring							Shows the token ranges for the node. Each row represents a vnode
nodetool describering <keyspace>				Shows the token ranges where all data would go for the specified keyspace.
nodetool repair							Repair inconsistencies in data across nodes.
nodetool pausehandoff						Pauses automatic hand-offs, or data repair. Can be useful when trying to trace hand-offs.
nodetool cfstats <table-name>					Displays detailed stats on the table, including read and write activity. Useful for ensuring that a table really is not in use before dropping it.



#-#
#-# CQL
#-#

CQL interacts with keyspaces, tables, and rows. Partitions, are implicitly handled within tables.


#-# Data types

http://docs.datastax.com/en/cql/3.3/cql/cql_reference/cql_data_types_c.html

Type 				Example
----				-------
numeric				bigint, decimal, double, float, int, varint, counter
string				ascii, text, varchar
date 				timestamp, timeuuid
other 				boolean, uuid, inet, blob
collections			TTLs work for individual elements in a collection rather than the entire collection.
					- set<int> A set allows only unique values and does not preserve order.
					- list<int> A list allows duplicate values and preserves order.
					- map<text,timestamp> A map is a dictionary of key value pairs.
					- tuple<int,text,varchar,double,int> A tuple can have many different data types, but they must always appear in the same order.
					- Complex types can be nested. e.g. map<text,frozen<tuple<int,double>>>. Whenever a complex type is nested, it must be frozen. Nested complex types are written and read as blobs, so they are not particularly efficient.
					- User Defined types
					 - CREATE TYPE clip (name varchar, duration int);
					 - CREATE TYPE person (name varchar, id varchar);
					 - CREATE TABLE courses (id text, author frozen<person>);
					 - User defined types must always use the frozen keyword.
					 - User defined types have the following advantages over using a tuple:
					  - Ability to identify components by name, not just order.
					  - Can be very helpful when multiple components have the same type.
					  - Can be used across multiple tables.


#-# Naming Conventions / Rules

Best advice: Don't get creative. Stick with alpha, lowercase, snake-case names.
- Snake-case is the preferred naming style in Cassandra (e.g. my_table)
- No hyphens (e.g. my-table)
- No spaces (e.g. my table)
- Double quotes are required if something starts with a number (e.g. "2050predictions")
- Double quotes are required if mixed case names are used (e.g. "myTable")


#-# Queries

Note - both INSERT and UPDATE are functionally upsert statements (insert or update).
Use the IN () selector with caution. An IN will cause a single coordinator node to search through multiple partitions.
It can become more efficient to perform multiple select queries rather than using a large IN clause to do a single select.

Command				Description							Examples
-------				-----------							--------
CREATE				Use to create a keyspace or table.				CREATE KEYSPACE sample WITH REPLICATION = {'class':'SimpleStrategy','replication_factor':3};
												CREATE TABLE my_table (id text, size int, PRIMARY KEY (id))
INSERT				Use to update or insert table data.				INSERT INTO my_table (id) VALUES ('guid');
UPDATE				Update or insert data in a table.				UPDATE my_table SET author = 'paul-ofallon' WHERE id = 'cassandra-developers';
DELETE				Delete data from a table.					DELETE FROM my_table WHERE id = 'asdf'
DROP				Drop / delete a table or keyspace.				DROP KEYSPACE sample;
												DROP TABLE my_table;
ALTER				Alter a table or keyspace.					ALTER KEYSPACE sample WITH DURABLE_WRITES = true;
TRUNCATE			Remove all data from a table.					TRUNCATE my_table;
WITH				Used to specify properties.					CREATE TABLE my_table (id varchar PRIMARY KEY) WITH comment='A table';
WRITETIME			Timestamp for when something was written.			SELECT id, WRITETIME(author) FROM my_table;


#-# Example

CREATE KEYSPACE pluralsight WITH REPLICATION = {'class':'SimpleStrategy','replication_factor':1};

CREATE TABLE courses (id varchar PRIMARY KEY);
CREATE TABLE IF NOT EXISTS courses (id VARCHAR PRIMARY KEY);
ALTER TABLE courses ADD duration int;
ALTER TABLE courses ADD released timestamp;
ALTER TABLE courses ADD author varchar;
DROP TABLE courses;

CREATE TABLE courses (
  id varchar PRIMARY KEY,
  name varchar,
  author varchar,
  audience int,
  duration int,
  hasClosedCaptions boolean,
  released timestamp
);

SELECT id, title FROM pluralsight.courses;
SELECT title, duration AS length FROM pluralsight.courses WHERE id = 'cassandra-developers';
SELECT title, published FROM pluralsight.courses WHERE id IN ('cassandra-developers', 'node-intro');
SELECT * FROM pluralsight.courses LIMIT 100;

INSERT INTO pluralsight.courses (id, author) VALUES ('cassandra-developers', 'paul-ofallon');
UPDATE pluralsight.courses SET author = 'paul-ofallon' WHERE id = 'cassandra-developers';
UPDATE pluralsight.courses SET author = 'paul-ofallon' WHERE id IN ('cassandra-developers', 'node-intro');

SELECT id, WRITETIME(author) FROM pluralsight.courses;

-- Delete a row
DELETE FROM pluralsight.courses WHERE id = 'node-intro';

-- Delete a column
DELETE author FROM pluralsight.courses WHERE id = 'node-intro';
UPDATE pluralsight.courses SET author = null WHERE id = 'node-intro';
INSERT INTO pluralsight.courses (id, author) VALUES ('node-intro', null);

-- Use a TTL (length of time in seconds) for a single column
UPDATE pluralsight.users USING TTL 32400 SET reset_token = 'asdf' WHERE id = 'john-doe';
SELECT TTL(reset_token) FROM pluralsight.users WHERE id = 'john-doe';

-- Use a TTL for an entire row
-- Only insert statements can be used to set a TTL for an entire row. Can't be done with UPDATE.
INSERT INTO pluralsight.reset_tokens (id, token) VALUES ('john-doe', 'asdoij') USING TTL 10800;

-- Use a TTL for the entire table. The entire table will be tombstoned after the TTL expires.
CREATE TABLE reset_tokens (
  id varchar PRIMARY KEY,
  token varchar
) WITH default_time_to_live = 10800;

-- If you have a table of id, partition_key, and you want to get one row for each id and the first partition_key, you can use this query:
SELECT DISTINCT id, partition_key FROM table;

-- Working with Collections
-- set<type>
INSERT INTO courses (id, name, features) VALUES ('node-intro', 'Introduction to Node.js', {'cc', 'transcript'});
UPDATE courses SET features = features + {'cc'} WHERE id = 'node-intro'; -- the + symbol adds to the set.
UPDATE courses SET features = features - {'cc'} WHERE id = 'node-intro'; -- the - symbol removes from the set.
UPDATE courses SET features = {} WHERE id = 'node-intro'; -- create an empty set.

-- list<type>
INSERT INTO courses (id, clips) VALUES ('node-intro', ['Getting Started']);
UPDATE courses SET clips = ['Introduction'] + clips WHERE id = 'node-intro'; -- prepend to the front of the list
UPDATE courses SET clips = clips + ['Introduction'] WHERE id = 'node-intro'; -- append to the back of the list
UPDATE courses SET clips = clips - ['Introduction'] WHERE id = 'node-intro'; -- removes all matching elements from the list.
UPDATE courses SET clips[0] = 'Introduction' WHERE id = 'node-intro'; -- add to list by position. Use indexing as you would in an array.
DELETE clips[0] FROM courses WHERE id = 'node-intro';

-- map<key-type,value-type>
INSERT INTO users (id, last_login) VALUES ('john-doe', {'abcde': '2015-06-30 09:02:24'});
UPDATE users SET last_login['abcde'] = '2015-09-12 07:01:34' WHERE id = 'john-doe';
UPDATE users SET last_login = last_login + {'abcde': '2015-06-30 09:02:24'} WHERE id = 'john-doe';
UPDATE users SET last_login = last_login - {'abcde': '2015-06-30 09:02:24'} WHERE id = 'john-doe';
DELETE last_login['abcde'] FROM users WHERE id = 'john-doe'
UPDATE users set last_login = {} WHERE id = 'john-doe'

-- secondary indexes
CREATE TABLE users (id varchar, first_name varchar, tags set<varchar>, PRIMARY KEY (id));
CREATE INDEX ON users(tags);
INSERT INTO users (id, first_name, tags) VALUES ('john-doe','John', {'java'});
SELECT * FROM users WHERE tags CONTAINS 'java';

-- a secondary index for a map
CREATE INDEX ON table(KEYS(map_column));
SELECT * FROM table WHERE map_column CONTAINS KEY 'foo';



#-#
#-# Reference
#-#

[1] http://tech.joshegan.com/posts/yr2016/cassandra-cheatsheet
[2] https://dzone.com/refcardz/apache-cassandra