# Kafka: The Definitive Guide
*Neha Narkhede, Gwen Shapira and Todd Palino*

# Chapter 1: Meet Kafka

## Publish/Subscribe Messaging
* A pattern characterized by the sender (publisher) of a piece of data (message) not specifically directing it to a receiver. Instead, the publisher classifies the message, and the receiver (subscriber) subscribes to receive certain classes of message.
 
## Enter Kafka

### Messages and Batches
* The unit of data within Kafka is a message.
  * An array of bytes.
* A message can have an optional key.
  * Also a byte array.
  * No specific meaning ot Kafka.
  * Used to control the writing of messages to partitions, e.g. by hashing the key selecting a partition using the result of the hash modulo.
* Messages are written in batched for efficiency.
  * All messages in a batch are produced to the same topic and partition.
  * Tradeoff between latency and throughput.

### Schemas
* Schemas are imposed on message content to enable them to be easily understood.
* A consistent data format allows writing and reading messages to be decoupled.

### Topics and Partitions
* Messages are catagorized into topics, like a database table or folder in a filesystem.
* Topics are broken down into partitions.
* Messages are written to a partition in append-only fashion and read in order from beginning to end.
* There is no guarantee of messages time-ordering across a topic with multiple partitions
* Each partition can be hosted on a different server, which means that a single topic can be scaled horizontally across multiple servers.
* A stream is considered to be a single topic of data, regardless of the number of partitions.
  * Represents a stream of data moving from the producers to the consumers.

### Producers and Consumers
* By default, a producer will balance messages over all partitions of a topic evenly.
* A consumer subscribes to one or more topic and reads messages in the order they were produced.
  * Keeps track of messages already consumed by recording the offset of messages, an increasing integer value appended to each message.
  * A consumer can stop and restart without losing its place.
  * Work as part of a consumer group: one or more consumers working together to consume a topic.
    * Assures each partition is only consumes by one member.
    * Ownership: the mapping of a consumer to a partition.
    * If a single consumer fails, the remaining members of the group rebalance the partitions to take over.

### Brokers and Clusters
* A single Kafka server is a broker.
* Brokers receive messages from producers, assign offsets to them and commit them to storage on disk.
* Brokers service consumers by responding to fetch requests for partitions and responding with the messages that have been committed to disk.
* Brokers are designed to operate as part of a cluster.
  * One broker functions as the cluster controller.
    * Elected automatically from live members of the cluster.
    * Responsible for cluster admin, inc. assigning partitions to brokers and monitoring for broker failures.
  * A partition is owned by a single broker in the cluster, called the leader of the partition.
    * May be assigned to multiple brokers, resulting in replication for the partition.
      * Provides redundancy.
      * All producers and consumers must connect to the leader.


# Chapter 2: Installing Kafka

## Installing Zookeeper
* Zookeeper stores metadata about the Kafka cluster and consumer client details.
* A Zookeeper cluster is called an ensemble.
  * Recommended to contain an odd number of servers to achieve quorom required to respond to requests.

## Installing a Kafka Broker
* Create and verify a topic:
  ```
  /usr/local/bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
  /usr/local/bin/kafka-topics.sh --zookeeper localhost:2181 --describe --topic test
  ```
* Produce messages to a test topic:
  ```
  /usr/local/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
  ```
* Consume messages from a test topic:
  ```
  /usr/local/bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning
  ```

## Broker Configuration

### General Broker
* `broker.id`:
  * Every broker must have an integer id.
  * Defaults to 0.
  * Must be unique within the cluster.
* `port`:
  * For ports lower than 1024, Kafka must be run as root, which is not recommended.
* `zookeeper.connect`:
  * The address of the Zookeeper used for storing broker metadata.
  * A semicolon-separated list of the form `hostname:port/path`.
    * `path`: An optional Zookeeper path to use as a chroot environment for the Kafka cluster.
      * Recommended. Allows the Zookeeper ensemble to be shared with other applications, including other Kafka clusters, without conflict.
* `log.dirs`:
  * Kafka persists all log segments to disk in this location.
  * Comma-separated list of paths.
  * If more than one path is specified, the broker stores partitions on them in "least-used" fashion with one partition's log segments always kept within the same path.
    * Broker places a new partition in the path that has the fewest partitions currently stored in it, not the least amount of disk space used.
* `num.recovery.threads.per.data.dir`:
  * Kafka uses a configurable pool of threads for handling log segments, used for:
    * When starting normally, to open each partition's log segments.
    * When starting after failure, to check and truncate each parition's log segments.
    * When shutting down, to cleanly close log segments.
  * By default, only one thread per log directory is used.
    * Recommended to increase; makes a large difference when recovering from unclean shutdowns.
* `auto.create.topics.enable`:
  * By default, brokers create topics automatically when:
    * A producer starts writing messages to the topic.
    * A consumer starts reading messages from the topic.
    * When any client requests metadata for the topic.
