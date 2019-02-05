# Cassandra Guide
## Background
Cassandra started life at Facebook where it powered the inbox search feature at Facebook where it supported 250 million users. In 2008 it became open sourced and soon after that became an Apache project.
It promises to offer linear scalability in proportion to the number of nodes in the network and is one of the most popular NOSQL databases on the market.

You can classify Cassandra as a Key Value Store, in otherwords it is just like a map in computer science, where in the case of Cassandra it is a sorted map of keys (row keys) whose values are a sorted map of column names who in turn map to the actual data values

[![Cassandra Visual Data Model](https://www.ebayinc.com/assets/Uploads/Blog/2012/07/thinkmap.png)]()

Cassandra is a NOSQL data store in the sense that it does away with Relation Database concepts like relations (i.e. no joins), atomicity (no transactions) and consistency (i.e. updates will be applied in a certain order reliably) to favour high availability for reads and writes, denormalisation and eventual consistency that is after some time updates will propogate through the system - Cassandra's update resolution is last write wins.

Cassandra has a data manipulation and definition language which is similar to SQL called ([CQL](https://docs.datastax.com/en/cql/3.3/cql/cql_reference/cqlReferenceTOC.html)). Don't be fooled by its SQL looking appearance, which a lot of the features may look similar, they are implemented internally in a completely different way that an RDBMS


## Architecture
Cassandra is designed to have no single point of failure. It is designed as a Peer to Peer network where nodes can join / leave the systems at anytime. You can picture the topology of the network as a ring of nodes.

[![Cassandra Ring Topology](https://66.media.tumblr.com/b7609bbb5eb83c804765ece19f6e8b9a/tumblr_inline_pjzq9gAxci1re4bph_540.png)]()

Cassandra relies on the concept of consistent hashing to distribute data among nodes. In consistent hashing the largest value wraps around to the smallest value. This creates a circular range which nicely mirrors the ring topology of Cassandra. Each node in the ring topology is responsible for handling certain sections commonly referred to token ranges. This information on which node owns what token range(s) is distributed among nodes, such that when a request comes in from clients a cassandra node itself can quickly calculate the hash and then route the request to the node that is responsible for the data as it can easily check its portion to see if the key is contained inside its ranges.
Cassandra achieves this through the use of a partitioner, which is set at the cluster level. It is the job of a partitioner to map data to a token (a fixed position on the circular has range).
The default partitioner in recent version of Cassandra is the [Murmur 3 Partitioner](https://docs.datastax.com/en/cassandra/3.0/cassandra/architecture/archPartitionerM3P.html?hl=murmur), which has a range of -2<sup>63</sup> to +2<sup>63</sup>-1. Generally speaking this will be fine for most use cases. You must ensure that whatever partitioner you pick roughly distributes data among nodes in the cluster such that each node get roughly the same amount of work. If this is not the case, you will get hotspots in the cluster leading to an overall decline in the efficiency of the system and of course wasted resources. There are [load testing tools](https://docs.datastax.com/en/cassandra/3.0/cassandra/tools/toolsCStress.html) that can help veryify this. 

Cassandra uses the concept of replicas as well to make data more available for reads and writes. The more replicas you have the longer it will generally take for updates to permeate through the cluster. Cassandra provides a bunch of replication strategies out of the box. For our use case we will use the [Network Topology Strategy](https://docs.datastax.com/en/cassandra/3.0/cassandra/architecture/archDataDistributeReplication.html) in production. The job of a replication strategy is to ensure there are or that they can be at least X replicas of the data, if for any case you do not have enough nodes in the cluster to satisfy the replication factor the writes will start to fail.


Cassandra is quite flexible that it allows clients to dictate how many replicas to consult when performing read or write operations. With this flexible model, normally reffered to tunable consistency you can do fire and forget operations by not waiting for responses from any replicas to being fully consistent, that is waiting for all replicas to reply. Generally speaking the best approach is somewhere in the middle. Ideally speaking the read consistency + the write consistency should be greater than the replication factor of the cluster. The main reason for this rule is that at least one node on the cluster will have up to date state of the data. Cassandra's other mechanisims (like [read repair](https://academy.datastax.com/support-blog/read-repair) among other things) which I will not discuss will take over to ensure that data is replicated to the other replicas.
## Data Modelling

With Cassandra you should first begin with what queries that I need to do. Based on how you answer this question, you will try to take advantage of how Cassandra stores data to satisfy those requirements.

As mentioned before Cassandra uses CQL as a mean to define tables. In Cassandra tables live inside a Keyspace, which is akin to a database in a RDBMS. CQL tables, just like SQL tables have names, columns fields and primary keys.

We use the tool cqlsh to interact with Cassandra. You can install Cassandra locally with brew with the following command

```bash
brew install cassandra
```

You can then start and stop cassandra using

```bahs
brew services [stop|start] cassandra
```

Lets then open up a cqlsh terminal by typing in

```bash
cqlsh
```

Firstly lets create a development keyspace called *arix*

```
CREATE KEYSPACE arix
  WITH REPLICATION = { 
   'class' : 'SimpleStrategy', 
   'replication_factor' : 1 
  };
```

This creates a single data center keyspace, which only maintains a single copy of data. This is obviously not good in a production environment, but its good enough for testing. You can follow the reference [here](https://docs.datastax.com/en/cql/3.3/cql/cql_reference/cqlCreateKeyspace.html)

We can then switch to this keyspace by using 

```
use arix;
```

Lets start with an example. Lets say I would like to satisfy the following requirement.

```
I would like to be able to fetch the number of available rooms for sale at a particular hotel, 
which belongs to a particular publisher for a specific date
```

Firstly lets extract out the entities / fields
- hotel (represented by a hotel_code e.g. 'ABC')
- date (a date in YYYY-MM-DD format e.g. 2019-01-20)
- publisher (i.e. a property management system e.g. 'OPERA')
- availability (a natural number)

Based on the criteria of the query we can safely say that the combination of a publisher, hotel and date uniquely maps to availability

We can safely say that publisher_code, hotel_code and date is the primary key.

Lets know create the table

```
CREATE TABLE availability (
   publisher_code text,
   hotel_code text, 
   date date,
   availability int, 
   PRIMARY KEY ((publisher_code, hotel_code, date)));
```

For the most part the column definitions will look exactly the same as what you would do with SQL. The main difference is with the primary key i.e.

```
  PRIMARY KEY ((publisher_code, hotel_code, date))
```

Notice that I have put an additional set of brackets, without doing this the partitioner would only take publisher_code as the input instead of the combination of all three fields.

We basically here have a very fast hashmap type look up.

Lets know insert some data

```
insert into 
availability (publisher_code, hotel_code, date, availability) 
values ('OPERA', 'ABC', '2019-01-19', 10);
```

and do a select statement

```
select * 
  from availability 
where 
  publisher_code = 'OPERA' 
and 
  hotel_code = 'ABC'
and 
  date = ' 2019-01-19';
```

which yields the following results

```
 publisher_code | hotel_code | date       | availability
----------------+------------+------------+--------------
          OPERA |        ABC | 2019-01-19 |           10
```



Lets say we now have a requirements change, that we now must specify a date range instead. 
We must first see whether that the current table can answer the question ?

The answer is no. This is because at the moment the partitioner is taking the hash of the publisher_code, hotel_code and date which means that this data will be randomly distributed among multiple cassandra nodes - which will incur additional network round trip times which is not great for performance. You could argue that it is possible to brute force the dates by running individual queries for each date in the date range but this puts complexity on the clients.

lets insert another record and run a range query on date, its informative to sometimes see and understand the error messages

```
insert into 
availability (publisher_code, hotel_code, date, availability) 
values ('OPERA', 'ABC', '2019-01-20', 5);
```

and run the query

```
select * 
from availability 
where 
  publisher_code = 'OPERA' 
and 
  hotel_code = 'ABC' 
and 
  date >= '2019-01-18' 
and 
  date <= '2019-01-22';
```

You will get the following warning

```
InvalidRequest: Error from server: code=2200 [Invalid query] message="Cannot execute this query as it might involve data filtering and thus may have unpredictable performance. If you want to execute this query despite the performance unpredictability, use ALLOW FILTERING"
```

Please do not ever use allow filtering.



Luckily for us the change is rather simple, we just need to update the definition of the primary key from

```
PRIMARY KEY ((publisher_code, hotel_code, date))
```

to 

```
PRIMARY KEY ((publisher_code, hotel_code), date)
```

It might seem like a really small change, but it will make a *HUGE* difference to performance. What we done is to instruct the partitioner to use only the publisher_code and hotel_code as input. This means that all data having the same publisher_code and hotel_code will end up on the same node. Cassandra will cluster this data on disk by the date field in ascending order. With this inplace we can now perform range queries like

```
select * 
from availability 
where 
  publisher_code = 'OPERA' 
and 
  hotel_code = 'ABC' 
and 
  date >= '2019-01-18' 
and 
  date <= '2019-01-22';
```

To test this out I ran the following

```
drop table availability;

CREATE TABLE availability (
   publisher_code text,
   hotel_code text, 
   date date,
   availability int, 
   PRIMARY KEY ((publisher_code, hotel_code), date));

insert into 
availability (publisher_code, hotel_code, date, availability) 
values ('OPERA', 'ABC', '2019-01-19', 10);

insert into 
availability (publisher_code, hotel_code, date, availability) 
values ('OPERA', 'ABC', '2019-01-20', 5);

select * 
from availability 
where 
  publisher_code = 'OPERA' 
and 
  hotel_code = 'ABC' 
and
  date >= '2019-01-18' 
and
  date <= '2019-01-22';

```

which yields correctly

```
 publisher_code | hotel_code | date       | availability
----------------+------------+------------+--------------
          OPERA |        ABC | 2019-01-19 |           10
          OPERA |        ABC | 2019-01-20 |            5

```

There are many more feature available to you in CQL like list, sets, maps, secondary indexes. The list literally goes on and on. I reccomend that you consult the [CQL reference](https://docs.datastax.com/en/cql/3.3/cql/cql_reference/cqlReferenceTOC.html). Please stick to features that are ideally CQL 3.0 compatible.
