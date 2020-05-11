# Learning Kafka

## What is it?

"a high-throughput distributed messaging system"

(created by linkedin, now maintained by confluent, under apache)

concepts to learn?

 * broker
 * cluster
 * partition
 * topic
 * zookeeper

what is it?

 * distributed messaging system
 * distributed, resilient, fault tolerant architecture
 * high performance - < 10ms latency
 * horizontal scalability

use cases:

 * messaging
 * activity tracking
 * gather metrics
 * app logs
 * stream processing
 * de-coupling
 * integration with Big Data
 
Use case example:

  * uber gather real-time user/taxi/trip data to calculate surge pricing/ forecast demand

## Fundamentals

### Topics:

what is a topic?

 * it represents a stream of data.
 * topics split into partitions
 * partitions manage balancing of load


               |---
               |
               | partition 0
Kafka Topic -> |
               | partition 1
               |
               | partition 2
               |
               |---

each partition is ordered
each message within a partition gets an incremental id, called an offset

Specify partition setup when creating the topic. (can be changed later)

So a message location would be `topic.partition.offset` to identiy an individual message.

messages expire (default is 1 week) but offset always increasing
data written to partition is immutable
by default data assigned randomnly to a partition unless a key is provided


### Brokers:

A Kafka cluster contains multiple brokers. A broker is basically a server, identified with an integer ID.
A broker contains certain topic partitions.
If you connect to a broker, you are connected to the whole cluster.

When creating topics + partitions, kafka automatically distributes the partitions across brokers.

### Topic replication:

resilience over a machine failing:

topic has a replication factor (> 1 usually 2 and 3) - 3 = gold standard
if a broker goes down - another broker can handle the protection if there is replication setup because of the replication factor.

A partition has a leader broker for a partition. Rules:

 * only 1 broker can be leader for a given partition
 * only that leader can receive and serve data for a partition
 * Other brokers are passively synchronizing data
 * each partition has 1 leader and potentially multiple ISRs (in-sync replicas)

selecting/ switching of leaders when brokers go down is handled by zookeeper in kafka

### Producers

Producers write data to topics
Producers automatically know which broker and partition to write to (just have to know how to connect to kafka)
Producers should automatically recover in case of broker failures

Producers manage the load balancing.

Producers can receive an ACK of a write.

acks=0: Producers send and don't wait for ack - possibility of data loss as the broker might not receive and we wouldn't know.
acks=1: (default) Producers wait for leader ACK - limited data loss (no guarantee it's replicated)
acks=all: Leader + replicas ACK - (no data loss - hopefully :) ) as it's replicated

Message Keys:

Prodcuers can send a key with the message.
If key = null, data is sent "round robin" across brokers (load balancing)
If key sent, then all messages with that key sent to same partition.

Used if you want specific ordering for a specific field.

### Consumers

Consumers read data from a topic
Consumers automatically know which broker to read from
Consumers know how to recover from broker failures
Data is read in order within each partition

Consumers can read from multiple partitions.
No guarantee of reads across partitions.

### Consumer Groups

Consumers read data in consumer groups
Each consumer within a group reads from exclusive partitions
Consumer group represents an application

GroupCoordinator and ConsumerCoordinator within kafka work to assign consumers to partitions
if you have more consumers than partitions, some consumers will be inactive

usually if you want a high number of partitions you need a high number of consumers

### Consumer offsets

Kafka stores the offsets at which a consumer group has been reading - like a bookmark of where a consumer has read to.
the offset commited live in a kafka topic named __consumer_offsets (__!)

when a consumer in a group has processed data if should commit the offset (kafka manages this)
therefore is a consumer dies it can pick up from where it left off

committing a consumer offset implies delivery semantics:

You have to choose 1 of these:

At most once: when offsets are committed as soon as the message is received (i.e. if lost it won't be read again)
At least once: (usually prefered): do something with data *then* commit the offset - if it goes wrong, it will be read again - If you use this - ensure your processing is idempotent!
Exactly once: only achieved using kafka => kafka workflows using streams api:
  e.g. for kafka -> external system workflows would have to use an idempotent consumer.

### Kafka broker discovery:

Every kafka broker is a bootstrap server. - you connect to 1 broker and it can tell you how to connect to all brokers.
Each broker knows about all brokers, topics and partitions (the metadata of it)

Kafka client: connects to 1 broker and gets metadata request, broker returns broker list, client then knows which broker to connect to.

### Zookeeper:

It manages brokers (keeps a list of them)
manages leader election for partitions
if a broker dies, comes up, topic generation/ deletions: notifies kafka.
kafka requires zookeeper.

Zookeeper must operate with an odd number of servers: 1 (or preferably 3,5,7)
Zookeeper has a leader (handles writes) - rest are for handling reads
since 0.10 zookeeper does not store consumer offsets (instead in a kafka topic)

### Guarantees:

 * Messages are appended to a topic-partition in the order they are sent
 * Consumers read messages in the order stored in a topic-partition
 * With a replicatoin factor of N, producers and consumers can tolerate up to N - 1 brokers being down (therefore 3 is a good idea)
 * The same key will always go to the same partition (As long as the number of partitions remains constant for a topic)


