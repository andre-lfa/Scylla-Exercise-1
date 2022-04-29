# Tasks

1. Create a 3 nodes cluster, with one datacenter. Each of the nodes in a different rack, dc: north, racks north1, north2, north3. 

2. Create a keyspace with 3 tables, one of the tables using STCS, another LCS, another TWCS. 

3. Insert 10,000 records in each of the tables, using loop and cqlsh. 

4. In the TWCS table, when creating the table and inserting data use time-window 30 minutes and the data to expire with 1 hour TTL.

5. Add a DC with 3 more nodes, each of the nodes in a different rack, dc: south, racks south1, south2, south3.

6. Install Scylla Manager.

7. Run repair using Scylla manager.

8. Decommission the old DC, keeping only the new created DC.

9. Add a node, decommission a node.

10. Then kill one of the nodes, destroy one of the containers (kill the seed node).

11. Replace procedure to replace this node we've killed.

### OBS.: 

All required files are in this repo, just clone it, cd into the directory and run the commands below according to the exercises.

### TASK 1:

Create a 3 nodes cluster, with one datacenter. Each of the nodes in a different rack, dc: north, racks north1, north2, north3.
```sh
docker-compose -f docker-compose-dc1.yml up -d
```

### TASK 2 & 4:

Create a keyspace with 3 tables, one of the tables using STCS, another LCS, another TWCS. In the TWCS table, when creating the table and inserting data use time-window 30 minutes and the data to expire with 1 hour TTL.

```sh
andre@ScyllaPC:~/scylla-exercise_1$ docker exec -it scylla-node1 cqlsh
Connected to  at 172.19.0.3:9042.
[cqlsh 5.0.1 | Cassandra 3.0.8 | CQL spec 3.3.1 | Native protocol v4]
Use HELP for help.
cqlsh> CREATE KEYSPACE exercise1 WITH replication = { 'class' : 'NetworkTopologyStrategy', 'north' : 3};
cqlsh> use exercise1;
cqlsh> CREATE TABLE exerciseLCS (
     field1 int,
     field2 int,
     field3 text,
     PRIMARY KEY (field1)
 ) WITH compaction = { 'class' : 'LeveledCompactionStrategy' };
 
cqlsh> CREATE TABLE exerciseSTCS (
     field1 int,
     field2 int,
     field3 text,
     PRIMARY KEY (field1)
 ) WITH compaction = { 'class' : 'SizeTieredCompactionStrategy' };
 
 cqlsh> CREATE TABLE exerciseTWCS (
     field1 int,
     field2 int,
     field3 text,
     PRIMARY KEY (field1)
 ) WITH compaction = { 'class' : 'TimeWindowCompactionStrategy', 'compaction_window_unit' : 'MINUTES', 'compaction_window_size' : '30' } AND default_time_to_live = 3600;  
```

### TASK 3:

Insert 10,000 records in each of the tables. I've used a CSV file for each table. 

Copy the files from the files folder to '/tmp/scylla-exercise/' in your machine or just create them using a generator or something. Just remember to change the file name in the command below.

```sh
cqlsh> COPY exerciseLCS (field1 , field2 , field3) FROM '/tmp/scylla-exercise/data.csv' WITH DELIMITER = ',' AND HEADER = true;

cqlsh> COPY exerciseSTCS (field1 , field2 , field3) FROM '/tmp/scylla-exercise/data0.csv' WITH DELIMITER = ',' AND HEADER = true;

cqlsh> COPY exerciseTWCS (field1 , field2 , field3) FROM '/tmp/scylla-exercise/data1.csv' WITH DELIMITER = ',' AND HEADER = true;
```

### TASK 5:

Add a DC with 3 more nodes, each of the nodes in a different rack, dc: south, racks south1, south2, south3.

```sh
docker-compose -f docker-compose-dc2.yml up -d
cqlsh> ALTER KEYSPACE system_schema WITH replication = { 'class' : 'NetworkTopologyStrategy', 'north' : 3, 'south' : '3'};
```

### TASK 6:

Install Scylla Manager. 

Procedure to follow: https://manager.docs.scylladb.com/stable/docker/index.html. 

If cloning this repo, just run: 

```sh
docker-compose up -d
```

### TASK 7:

Run repair using Scylla manager.

```sh
sctool repair --cluster <CLUSTER_ID>
```

### TASK 8:

Decommission the old DC, keeping only the new created DC.

[Remove a Data-Center from a Scylla Cluster](https://docs.scylladb.com/operating-scylla/procedures/cluster-management/decommissioning-data-center/#remove-a-data-center-from-a-scylla-cluster)

```sh
cqlsh> ALTER KEYSPACE exercise1 WITH replication = {'class': 'NetworkTopologyStrategy', 'south': '3'}  AND durable_writes = true;
```

```sh
andre@ScyllaPC:~$ docker exec -it south-node1 nodetool status
Datacenter: south
=================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns    Host ID                               Rack
UN  172.19.0.3  1.19 MB    256          ?       f9b8d021-beab-4170-a13c-7780b46d065b  south1
UN  172.19.0.7  1.19 MB    256          ?       7ddb87f7-8121-41f6-b982-d42451b1447c  south2
UN  172.19.0.4  1.19 MB    256          ?       35698bd1-cea3-40b7-9d3e-3ae4312a5246  south3
```

### TASK 9:

Add a node.

```sh
andre@ScyllaPC:~/scylla-exercise_1$ docker-compose -f docker-compose-dc2-node4.yml up -d 

andre@ScyllaPC:~/scylla-exercise_1$ docker exec -it south-node1 nodetool status 
Datacenter: south
=================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address      Load       Tokens       Owns    Host ID                               Rack
UN  172.19.0.3   1.15 MB    256          ?       f9b8d021-beab-4170-a13c-7780b46d065b  south1
UN  172.19.0.7   1.19 MB    256          ?       7ddb87f7-8121-41f6-b982-d42451b1447c  south2
UN  172.19.0.4   1.19 MB    256          ?       35698bd1-cea3-40b7-9d3e-3ae4312a5246  south3
UN  172.19.0.10  1.31 MB    256          ?       3ab11f46-6a6a-4fc4-9d0d-12187ea2bbbc  south4
```

Decommission a node.

```sh
andre@ScyllaPC:~/scylla-exercise_1$ docker exec -it south-node4 nodetool decommission

andre@ScyllaPC:~/scylla-exercise_1$ docker exec -it south-node1 nodetool status 
Datacenter: south
=================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address      Load       Tokens       Owns    Host ID                               Rack
UN  172.19.0.3   1.15 MB    256          ?       f9b8d021-beab-4170-a13c-7780b46d065b  south1
UN  172.19.0.7   1.19 MB    256          ?       7ddb87f7-8121-41f6-b982-d42451b1447c  south2
UN  172.19.0.4   1.19 MB    256          ?       35698bd1-cea3-40b7-9d3e-3ae4312a5246  south3
```

### Task 10

Then kill one of the nodes, destroy one of the containers (kill the seed node).

```sh
andre@ScyllaPC:~/scylla-exercise_1$ docker rm -f south-node1
south-node1
andre@ScyllaPC:~/scylla-exercise_1$ docker exec -it south-node2 nodetool status
Datacenter: south
=================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns    Host ID                               Rack
DN  172.19.0.2  966.96 KB  256          ?       f9b8d021-beab-4170-a13c-7780b46d065b  south1
UN  172.19.0.7  963.31 KB  256          ?       35698bd1-cea3-40b7-9d3e-3ae4312a5246  south3
UN  172.19.0.4  966.58 KB  256          ?       7ddb87f7-8121-41f6-b982-d42451b1447c  south2
UN  172.19.0.8  823.58 KB  256          ?       3ab11f46-6a6a-4fc4-9d0d-12187ea2bbbc  south4
```

### Task 11

Replace procedure to replace this node we've killed.

https://docs.scylladb.com/operating-scylla/procedures/cluster-management/replace-dead-node/