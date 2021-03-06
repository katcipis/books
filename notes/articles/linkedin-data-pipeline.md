# LinkedIn Data Pipeline

These notes are bases on the 
[Building LinkedIn’s Real-time Activity Data Pipeline](http://sites.computer.org/debull/A12june/pipeline.pdf) article.

It is like a continuation of [I lov logs](notes/i-lov-logs.md) notes, since the book
talks more about the idea of the pub/sub system for solving data integration, and the idea
of logs as data structures ordered by time (on this case a logical timestamp).

The article deepens on the problem and on the implementation details of Kafka, since it is
the materialization of these ideas.


## The problem

The problem they had was data integration, not just any kind of data integration, but a data integration
system that could support:

* Real time processes
* Batch processes
* High throughput
* Low latency
* Horizontal scalability
* Easy to evolve schemas

Usual database integration, using relational databases, can provide a reasonable service to batch processes,
but it is not well built for real time processes, it is more optimized on how you are going to search
and aggregate the data in various ways than providing a common integration point where all data
can be written and read, with high throughput.

Scaling horizontal is also challenging for most classical relational databases. Other databases like MongoDB and
ElasticSearch can scale better, but may fall short on some of other of these requirements (ElasticSearch does
not handle well schema changes, depending on the change, demanding a full re-indexing).

How much throughput are we talking about ? On the time the article was written it was 10 billion messages per day,
the read ratio generated 55 billion messages read per day.

It was with these problems at hand that LinkedIn started to build Kafka, I have great respect for tools that
are built to solve a real business problem instead of being a academic exercise.


## The solution: Kafka

The article has lots of details on tests made on other tools, like ActiveMQ. Message queues seems to optimize for
latency and not for throughput, and on their case throughput was more important than latency. Also the control
of each consumer queue generates data duplication or a lot of pointers/references for data across the queues,
which usually incurs on unneeded I/O and makes it harder do scale.

The solution for those problems, at least for LinkedIn case, was Kafka. Now we can get on some cool details about it.


### Overview

To attack all those requirements, it makes sense that the Kakfa model is extremely simple.
It is a write ahead log, where the messages are opaque byte arrays, identified by a message id.
This write ahead log is ordered, by a logical timestamp, and that is pretty much it.
Thinks start to get complex to enable better scalability, but the model is simple.
It is a basic publish / subscribe system, messages are always persistent.

Although Kafka is a kind of database, it does not seem to have been built to be used as one,
the idea is that N consumers will be reading from this write ahead log and save the data where
they want, the idea is to distribute data, not store it.

It has the notion of topics, where messages of the same kind are published.
Each topic has a cluster of Kafka brokers handling it, and each broker can have
multiple partitions for the same topic. This is the source of some complexity but it 
is also how it scales so well.

Another interesting point is the notion of subscriber (the consumer). A subscriber is a group
of cooperating processes, a cluster. So in the end, for each topic you have:

* A cluster of brokers (where each has N partitions)
* N consumer groups, where each consumer group is a cluster of consuming processes

This was very confusing at start for me, the idea that consumers also forms clusters
instead of thinking about individual consumers. But this is actually what happens when multiple
instances of the same process consume from the same queue, all these processes are part of
the cluster of consumers from queue X. The difference is that on Kafka this is explicit on the
integration protocol, and it is very important for scalability.

When you have a cluster of consumers, all processes that are part of
this cluster will be assigned to different partitions of that topic, this enables a great deal
of parallelism, since each process is processing is parallel different partitions, instead of
competing for data on the same partition.

One key difference to other messaging brokers (at least the ones I know) is that each consumer
must store where it has stopped consuming so it can recover from failures. Kafka does not store any kind
of client state (again, just a write ahead log), this reminds me a lot of the stateless idea
from REST, which makes sense as a way to achieve scalability. Also, since there is no state being
stored, there is no form of acknowledgment, which greatly simplifies the broker.

One of the key concepts is the partition, since at least for me it is one of the most
different things that Kafka provides, compared to other brokers I know.


#### The Partition

Each partition is consumed by exactly one consumer in the group.
By doing this it is ensured that the consumer is the only reader of that partition and consumes the data in order.
This is important, because ordering is guaranteed only within a partition, not across them.
Avoiding global ordering and consumption of a topic seems like a good way to scale.

Since there are many partitions you still get load balancing over many consumer instances.
One limitation is that you cannot have more consumer instances in a consumer group than partitions.

The obvious trade off is that if you require a global order over messages this
can only be achieved with a topic that has only one partition, though this will mean
only one consumer process per consumer group.

Since partitioning is so important, how do you control how things get partitioned ?
It is based on a key that the producer will provided, that will be used to hash the
messages. This seems to be analogous to hash maps, depending on how you hash your keys
you may end with a uniform distributed set of buckets or with lots of empty buckets and a small
set of gigantic buckets, since almost all keys clash on the same bucket. It is the same problem,
only that on Kafka the buckets are the partitions, and they are distributed.

![Anatomy of the topic](http://kafka.apache.org/images/log_anatomy.png)

If the producer chooses to not provide a key, a uniform load balancing will be done through
all the partitions (like a default hash map). If a key is provided, like a user id for example,
you will have a guarantee that all messages from user id X will be on the same partition.

This can make some kind of processing easier, since all messages from user X will be
consumed by the same process on a consumer group. With this guarantee processing session
information of a user can be done all in memory, for example, since all information from a
specific user will always be delivered to the same process.


#### Consistency

From what I understand, Kafka can be configured for different levels of consistency,
perhaps even having a full consistent mode, but to be honest this does not seem
to be the idea, and it is not the way the authors use the system.

According to them, they replace the requirement for consistency with good monitoring
and alerts on all logic tiers (producer / broker / consumer). So they have information
about delays and loss of messages, and can alert and act on that. But the system is optmized
for high throughput and low latency, characteristics that are usually not friends with
complete consistence.


### Engineering for high throughput

Some decisions on how Kafka enables high throughput shows how the throughput and latency
are opposing forces (and it allows you to configure how much you want from each).

There is 3 main approaches to achieve high throughput:

* Partitioning (already talked a lot about this)
* Batching
* Compression

Partitioning has been explained already, it enables a high throughput because it enables
a high level of parallelism, as far as you can come up with new partitions you can start
more consuming processes for the new partitions and these scale almost linearly and without
interference on current partitions (depending on how you do it of course, it is not magic).


#### Batching

Batching is the idea of accumulating multiple messages together to send them together
as one big chunk. There is nothing novel on this idea but if you want to trade off
more latency for a higher throughput it makes total sense.

This is tunable on the application level, you are able to configure how much you
want to batch and the timeout that will be used to send the messages if
they never get to the desired size (a source of latency).

There is a multisend feature that enables you to mix multiple topics together, this
is useful when you have topics with low volume of data.

There is no double buffering of messages on the broker, when data is received it is
directly written. Although data is not directly flushed to disk, this is delegated to
the operational system, relying on linux pagecache to make good choices about this.

You can also configure per topic to all writes to be flushed, if desired.
Being a little more inconsistent and not flushing writes pay off ?
According to the article they got an increment of 51,7x on messages per second
being written leveraging the pagecache instead of flushing all writes.

The same technique is used on the consumer side to read messages. Since writes and reads
are linear (a log) and usually consumers drift from each other, but not that much, you have a
very descent chance of always hitting the cache when reading data, so readers usually do
not incur any I/O operation on disk (depending on how much RAM is available for pagecache of course).


#### Shrinking data

No rocket science here, just batch all messages in one message set to compress them.
Obvious advantage of compressing multiple messages is that there is much more
redundancy on multiple messages than individual messages (the schema).

The algorithm used on LinkedIn is GZIP, and even with the increased CPU usage the
overall throughput has increased 30% using compression, due to economy on network
and disk I/O.


### Handling diverse data

To substitute XML they chose [Apache Avro](https://avro.apache.org/).
Of course that this alone does not prevent breakages, so to evolve schemas
without breaking consumers an automated code review system was put in place,
together with some form of central schema registry server.

If your schema has changed on a way that is not compatible with previous versions
(like removing a field, or changing its type) the registration process fails.
This registration process is automated on the build of the data producers.


## Conclusion

On the end it seems to me that Kafka is the child of a database with a
message broker, the child got the write ahead log from the database and the
publish/subscribe model from the message broker. Specially with the addition
of configurable consistency level makes Kafka a really interesting tool.

One of the operational remarks mentioned on the article is that there is no
system that cant suffer from data loss or some other kind of failure given
that the error is sufficiently rare, the impact is bounded to a part of the system,
and the failure can be monitored and measured.

The entire data pipeline is measured for data loss and failures, instead of trying to
guarantee that no loss ever happens. This path seems to enable good automation and
automated failure detection and failure recovery, it seems to me as a better option
than just relying on the database to never fail, although it is harder, which explains
why most people choose to just trust the database.

Also Kafka seems to be a pretty big gun, if you just need pub/sub for a small amount
of messages, or you dont have a N x M producers/consumers problem it may be an
overkill. A simple API on top of a database may be enough, and it really seems to
be simpler to maintain and use.
