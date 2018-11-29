# Cassandra Cluster in Docker #
A `docker-compose` blueprint that describes a 3 node Cassandra cluster.
It only exposes important Cassandra ports on the seed node to the host
machine. Internally, all of the nodes will be a part of the same Docker
network and will form a cluster using that Docker network.

## Instructions ##

### Getting Started ###

In order to bring up the cluster:
- Use `docker-compose up` to see the logs of all the containers
- Use `docker-compose up -d` if you want it to run in the foreground

In order to clean up the cluster, use `docker-compose down`

### CQLSH ###

Installing [Apache Cassandra](http://cassandra.apache.org/) locally means that you can 
access the `cqlsh` application.  Install/unpack the Cassandra binary distribution and run the following

```
$CASSANDRA_HOME/bin/cqlsh
```

At the prompt you can run the following commands

``` 
cqlsh> describe cluster

Cluster: Test Cluster
Partitioner: Murmur3Partitioner

cqlsh> describe keyspaces

system_traces  system_schema  system_auth  system  system_distributed

cqlsh> 
```

Create a new keyspace

```
CREATE KEYSPACE counters WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 4};
```

The `replication.class` of `SimpleStrategy` just means that Cassandra will blindly try and replicate the data to 
the given `replication_factor` number of hosts.  In this case the same data will be copied to 4 nodes in the cluster 
(which is the current size of the setup).

This is about ensuring there are backups of the data such that the loss of some nodes doesn't render information 
inaccessible or lost/corrupt.  

Create a new table

```
USE counters;

CREATE TABLE eventCounters ( "counterName" ascii PRIMARY KEY, "counterValue" counter );
```

Insert (update is the only allowed operation for counter tables) some data and check it is there

``` 
UPDATE eventCounters SET "counterValue" = "counterValue" + 1 WHERE "counterName" = 'cqrsEvent';
SELECT * FROM eventCounters;

 counterName | counterValue
-------------+--------------
   cqrsEvent |            1

(1 rows)
```

Create some more conventional tables in a different key spaces

``` 
CREATE KEYSPACE eventstores WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 3};
```

Only need to replicate to some of the machines, it's likely to be alot of data.

```
USE eventstores
CREATE TABLE events ( "eventCount" bigint PRIMARY KEY, "eventType" int, "payload" ascii); 
```



### JMX ###

This took a while to figure out, trying to connect to the 'local' JMX port, even when mapped in the docker-compose file, 
doesn't work.  In the end the instructions 
[here](https://docs.datastax.com/en/cassandra/3.0/cassandra/configuration/secureJmxAuthentication.html)
gave some clues.  The trick is to add the `cassandra.jmx.remote.port` property which then enables remote 
RMI connectivity.  Presumably Cassandra is using this to simplify setting up the relevant javax/sun
properties under the hood.

The current setup exposes a JMX port for every node on 1890, 1891, 1892, and 1893
respectively.


## Notes ##

### Docker ###

You need to make sure that the Docker daemon has enough of resources otherwise you will encounter exit code 137 
(Out of Memory Killer) on your containers.

When you create a single node cluster and you try to add more nodes to the cluster, you must add them one by one. 
This means that you cannot have multiple nodes join the cluster (by pointing to the seed node) at
the same time. You must add a cluster, wait for it to join the ring and stabilize before you can begin to add another 
cluster. This is why you will see an additional delay on start up between the non-seed nodes.
If you attempt to join a new node whilst stabilization has not yet been achieved, you will see an error like this:

```
ERROR [main] 2017-08-22 23:19:11,055 CassandraDaemon.java:706 - Exception encountered during startup
java.lang.UnsupportedOperationException: Other bootstrapping/leaving/moving nodes detected, cannot bootstrap while cassandra.consistent.rangemovement is true
```

### Cassandra ###

This [article](http://thelastpickle.com/blog/2017/05/23/auto-bootstrapping-part1.html)
discusses the implications of turning `cassandra.consistent.rangemovement`off.


## References ##

### Cassandra ###

- [CQLSH Documentation](http://cassandra.apache.org/doc/latest/cql/index.html)
- [Cassandra book](https://teddyma.gitbooks.io/learncassandra/content/index.html)


## Credits ##

- [Calvin's original cluster setup](https://github.com/calvinlfer/compose-cassandra-cluster) - this repo forked from that
- [The Last Pickle](http://thelastpickle.com/blog)
