# Smart buildings simplified - Envio

* <b>Status</b>: Proposed 
* <b>Deciders</b>: Matthias Utke (CTO) 
* <b>Date</b>: 2021-07-11 

<b>Technical Story</b>: [Technical Assessment with Envio Systems](../tech-assessment/SBSE.Practical.docx.pdf) 

## Context and Problem Statement

<p>We have multiple buildings located internationally. Each building has multiple devices. Each building
has a minimum of 100 devices. With each device generating data every 10 seconds.</p>

<p>Within each building there is a Gateway server (device in which any piece of software can be
deployed), which sends and receives data to and from the devices.</p>

<p>We provide our clients with a Web app to view a full summary of the data (averages, min, max, charts,
etc) as they don't require all the extracted information.</p>

<p>For security reasons, the Web app can’t have direct access to the service where raw data (data coming
directly from the devices) is stored. In addition, we are legally required to process and handle data
within the same country in which it originated. For example, if the building is in the U.S., the data
should be stored only within the U.S.</p>

<p>And finally, we would like to provide our teams with an endpoint (a specific URL) through which they
can access raw data or more advanced analysis for one or more buildings.</p>

<p>How would you design a solution with the following criteria?</p>

## Decision Drivers 
* Data Protection:
  * process and handle data within the same country in which it originated 
* Easy horizontal scalability:
  * multiple buildings with multiple devices (minimum of 100 devices)
  * easy to extend for new buildings
* High throughput:
  * At least 100 buildings (assumption) x 100 devices 
* Medium latency:
  * 10 seconds
* Durable storage
* Zero downtime:
  * IoT devices usually can't handle cache even in case of failure
* Provide a Landing Zone layer:
  * raw data
* Provide a transformation continuous process. 
* Provide a Serving layer:
  * transformed data for advanced analysis
* Provide an Api gateway management:
  * to be possible expose a REST API 

## Considered Options

Acquisition Data + Data Hub:

* [Option 1] [Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html) + Confluent [MQTT source Connector](https://docs.confluent.io/kafka-connect-mqtt/current/mqtt-source-connector/index.html) + [MQTT broker](https://en.wikipedia.org/wiki/MQTT) + Kafka Cluster:
  * Kafka Connect for MQTT acts as an MQTT client that subscribes to all the messages from an MQTT broker.
* [Option 2] Confluent  [MQTT Proxy](https://docs.confluent.io/platform/current/kafka-mqtt/index.html) + Kafka cluster:  
  * The IoT devices connect to the MQTT proxy, which then pushes the MQTT messages into the Kafka broker.
* [Option 3] [HiveMQ](https://www.hivemq.com/docs/kafka/4.6/enterprise-extension-for-kafka/kafka.html) + Kafka Cluster:
  * HiveMQ Enterprise Extension for Kafka implements the native Kafka protocol inside the HiveMQ broker that writes directly to kafka node(s). 
* [Option 4] [Kafka-native implementation of MQTT](https://www.confluent.io/hub/simplemattersrl/waterstream-kafka-mqtt) + Kafka Cluster:
  * IoT devices use an MQTT client to send data to a fully featured MQTT broker. The MQTT broker is extended to include a native Kafka client and transposes the MQTT message to the Kafka protocol. This allows the IoT data to be routed to multiple Kafka clusters and non-Kafka applications at the same time. Waterstream is an MQTT broker that runs Kafka natively as a Kafka Streams application

Processing Layer:

* [Apache Spark Streaming using Kubernetes Operator](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator)

Serving Layer:

* [Cloud Storage as Data-API's](https://towardsdatascience.com/data-mesh-applied-21bed87876f2)

Api Gateway Management:
 * [Apigee API Management](https://cloud.google.com/apigee)

## Decision Outcome

Chosen option:<b>[Option 4], a kafka-native implementation: Waterstream MQTT Broker</b>, because only this option (considered) is a kafka-native implementation of MQTT focuses in one vendor. At the same time simplify architecture with fewer components.      

A MQTT broker that runs Kafka natively as a Kafka Streams application is the best option because simplify removing the usual MQTT broker or a MQTT proxy. There are no external MQTT clusters to manage or integration pipelines to develop in order to move data from devices to topics. 

Our decision includes an Apache Kafka because:
* it scales very well over large workloads and can handle extreme-scale deployments;
* has a high performance with large amount of data throughput in different environments;
* because has a low latency (unit of 10 milliseconds);
* has a feature to store data on permanent storage;

In order to naive in this ADR won't discuss options for processing, serving layer, and the API gateway. Here is suggested approach: 

* From kafka to landing zone a [Google Cloud Storage Sink Connector](https://docs.confluent.io/kafka-connect-gcs-sink/current/overview.html) will place the data in lading zone.
* So for be scalable we proposed uses a [cluster k8s with spark operator](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator) in order to transform the data from landing zone to serving layer using spark streaming. 
* The [Data-APIs](https://towardsdatascience.com/data-mesh-applied-21bed87876f2) could be implemented in any reasonable form like that (Data mesh approach): 
  * As CSV/parquet files located in a bucket (endpoints separated by subfolders, APIs separated by top-level folders);
  * As REST APIs via (static) JSON/ JSON lines
* In order secure and scale APIs we proposed use [Apigee Api Management](https://cloud.google.com/apigee). 

For each country we proposed launch cloud resources in different zones. Here is a list of [availability zones](https://cloud.google.com/compute/docs/regions-zones). Confluent Kafka SaaS can follow the same availability zones.     

### Positive Consequences 

* Waterstream can manage the MQTT state: subscriptions, QoS message status, retained message, etc.
* kafka Confluent SaaS is a solution can be provisioned for each region easily, and the amount paid will be per use.  
* Kafka can handle huge volumes of data, and it is highly reliable system, fault tolerant, and scalable.
* Reduces the need for multiple integrations i.e harvest data from different systems, you only have to create one integration with Apache Kafka for each producing system and each consuming system.
* Kafka can handle messages with very low latency.
* Usability: Very high, kafka is distributed, multiple copies of one data, a few machines down, no data loss, no unavailability
* Costs: The Data APIs are read-only, so cloud storage can be use store all static files. The Data APIs can handle https requests, be organized by domain, by product, or other approaches. SLAs can be manage to them: check their usage. 

### Negative Consequences 

* If you want to use Apache Kafka to deliver messages as they are, you will have no issues performance-wise. However, the problem occurs once you wish to modify the messages before you deliver them.
* Manipulating data on the fly is possible with Kafka, but the system it uses has some limits. It uses system calls to do it, and modifying messages makes the entire platform perform significantly slower.

## Overall architecture

![](./images/adr-envio.png?raw=true)

## Pros and Cons of the Options 

### [Option 1] [Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html) + Confluent [MQTT source Connector](https://docs.confluent.io/kafka-connect-mqtt/current/mqtt-source-connector/index.html) + [MQTT broker](https://en.wikipedia.org/wiki/MQTT) + Kafka Cluster:

* Good, because [kafka Connect](https://docs.confluent.io/platform/current/connect/index.html) source connectors are well-tested plugins that use abstractions to push data to kafka. 
* Good, because [Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html) cluster can be scalable to support more connectors. Each connector can scalable to have multiples tasks.
* Bad, [MQTT Source Connector](https://docs.confluent.io/kafka-connect-mqtt/current/mqtt-source-connector/index.html) is proprietary and requires a license, so a Confluent Enterprise licensed is needed.
* Bad, a MQTT broker is needed as an intermediary layer.

### [Option 2] Confluent [MQTT Proxy](https://docs.confluent.io/platform/current/kafka-mqtt/index.html) + Kafka cluster: 

* Good, because Confluent [MQTT Proxy](https://docs.confluent.io/platform/current/kafka-mqtt/index.html) delivers a Kafka-native MQTT proxy that allows organizations to eliminate the additional cost and lag of intermediate MQTT brokers.
* Good, because MQTT Proxy accesses, combines, and guarantees that IoT data flows into the business without adding additional layers of complexity.
* Good, MQTT Proxy is horizontally scalable, consumes push data from IoT devices, and forwards it to Kafka brokers with low latency.
* Bad, MQTT Proxy is proprietary and requires a license, so a Confluent Enterprise licensed is needed

### [Option 3] [HiveMQ](https://www.hivemq.com/docs/kafka/4.6/enterprise-extension-for-kafka/kafka.html) + Kafka Cluster 

* Good, because can buffer messages on the HiveMQ MQTT broker to ensure high-availability and failure tolerance whenever a Kafka cluster is temporarily unavailable.
* Good, because validate messages from Kafka with the help of a schema registry.
* Good, because have some features:
  * topic filters with full support of wildcards to route MQTT messages to the desired Kafka topic.
  * custom-handling transformation between HiveMQ and Kafka.
* Bad, HiveMQ  is proprietary and requires a license. 

### [Option 4] [Kafka-native implementation of MQTT](https://www.confluent.io/hub/simplemattersrl/waterstream-kafka-mqtt) + Kafka Cluster

A kafka-native implementation considered here is the Waterstream MQTT Broker. Waterstream is an MQTT broker that runs Kafka natively as a Kafka Streams application.

* Good, Waterstream works a bidirectional layer between Kafka and IoT devices. 
* Good, messages coming from MQTT clients are immediately written to Kafka.
* Good, MQTT state is stored in Kafka with no need for additional storage solution.
* Bad, is proprietary and requires a license, so a Confluent Enterprise licensed is needed.

