<!-- SPDX-License-Identifier: CC-BY-4.0 -->
<!-- Copyright Contributors to the ODPi Egeria project. -->
  
--8<-- "snippets/content-status/stable.md"

# Kafka Open Metadata Topic Connector

!!! info "Connector details"
    - Connector Category: [Open Metadata Topic Connector](/concepts/open-metadata-topic-connector)
    - Source Module: [kafka-open-metadata-topic-connector :material-github:](https://github.com/odpi/egeria/tree/main/open-metadata-implementation/adapters/open-connectors/event-bus-connectors/open-metadata-topic-connectors/kafka-open-metadata-topic-connector){ target=gh }
    - Jar File Name: `kafka-open-metadata-topic-connector.jar`

[Apache Kafka](https://kafka.apache.org/) provides a publish-subscribe service that allows a producer to publish events to subscribed consumers.  Events are organized into topics.  A producer publishes an event to a topic and the consumers register a listener to receive all events from a topic.

The Kafka Open Metadata Topic Connector implements an [Open Metadata Topic Connector](/concepts/open-metadata-topic-connector) for a `PLAINTEXT` Apache Kafka topic.  It supports both the event producer and event consumer interfaces.

![Kafka Open Metadata Topic Connector](kafka-open-metadata-topic-connector.svg)

## Configuration

The connection example shows how to configure the connection for this connector.  It is passing properties for both the producer and consumer.

!!! example "Connection configuration"
    ```json linenums="1" hl_lines="11"
    {
        "connection" :
        {
            "class": "Connection",
            "qualifiedName": "Kafka Open Metadata Topic Connector",
            "connectorType":
            {
                "class": "ConnectorType",
                "connectorProviderClassName": "org.odpi.openmetadata.adapters.eventbus.topic.kafka.KafkaOpenMetadataTopicProvider"      
            },
            "endpoint":
            {
                "class": "Endpoint",
                "address": {{KafkaTopicName}}
            },
            "configurationProperties": 
            {
                "producer": 
                {
                    "bootstrap.servers": {{kafkaEndpoint}}
                },
                "local.server.id": "{{consumerId}}",
                "consumer":
                {
                    "bootstrap.servers": {{kafkaEndpoint}}
                }
            }
        }
    }
    ```
    Add the name of the topic in {{topicName}}; a unique consumer identifier in {{consumerId}} and the endpoint for Apache Kafka (for example localhost:9092) in {{kafkaEndpoint}}.


### Default properties for the producer and consumer

The properties for the producer and consumer passed on the connection override the default set of properties coded into the connector implementation.

#### Producer

(see [Apache Kafka producer configurations](http://kafka.apache.org/0100/documentation.html#producerconfigs) for more information and options)

| Property Name | Property Value |
|---------------|----------------|
| bootstrap.servers | localhost:9092 |
| acks              | all |
| retries | 1 |
| batch.size | 16384 |
| linger.ms | 0 |
| buffer.memory | 33554432 |
| max.request.size | 10485760 |
| key.serializer | org.apache.kafka.common.serialization.StringSerializer |
| value.serializer | org.apache.kafka.common.serialization.StringSerializer |
| bring.up.retries | 10 |
| bring.up.minSleepTime | 5000 |

#### Consumer

(see [Apache Kafka consumer configurations](http://kafka.apache.org/0100/documentation.html#newconsumerconfigs) for more information and options)

| Property Name | Property Value |
|----------------|-----------------|
| bootstrap.servers | localhost:9092 |
| enable.auto.commit | true |
| auto.commit.interval.ms | 1000 |
| session.timeout.ms | 30000 |
| max.partition.fetch.bytes | 10485760 |
| key.deserializer | org.apache.kafka.common.serialization.StringDeserializer |
| value.deserializer| org.apache.kafka.common.serialization.StringDeserializer |
| bring.up.retries | 10 |
| bring.up.minSleepTime | 5000 |

#### Security

By default, kafka security is not configured. The exact configuration may depend on the specific kafka service being used. Service specific notes
are below. They may work for other providers, and feedback is welcome so that this documentation can be updated accordingly.

##### Example: IBM® Event Streams for IBM Cloud®

There are 2 key pieces of information that are provided in the documentation for your configured cloud service

 * List of brokers - try to specify at least 3. The hosts in the examples below will need updating
 * API key - referred to as MYAPIKEY below
 
 Given these, configure kafka properties for both provider and consumer as follows:
```
"broker.list: "broker-5-uniqueid.kafka.svcnn.region.eventstreams.cloud.ibm.com:9093, broker-3-uniqueid.kafka.svcnn.region.eventstreams.cloud.ibm.com:9093, broker-2-uniqueid.kafka.svcnn.region.eventstreams.cloud.ibm.com:9093, broker-0-uniqueid.kafka.svcnn.region.eventstreams.cloud.ibm.com:9093, broker-1-uniqueid.kafka.svcnn.region.eventstreams.cloud.ibm.com:9093, broker-4-uniqueid.kafka.svcnn.region.eventstreams.cloud.ibm.com:9093"
"security.protocol":"SASL_SSL",
"ssl.protocol":"TLSv1.2",
"ssl.enabled.protocols":"TLSv1.2",
"ssl.endpoint.identification.algorithm":"HTTPS",
"sasl.jaas.config":"org.apache.kafka.common.security.plain.PlainLoginModule required username='token' password='MYAPIKEY';",
"sasl.mechanism":"PLAIN"
```
An example of a use of this configuration can be found in the virtual data connector helm charts. See [odpi-egeria-vdc helm chart](https://github.com/odpi/egeria-samples/tree/main/helm-charts/odpi-egeria-vdc)

#### Handling Kafka Cluster Bring Up Issues

It's strongly recommended that you ensure Kafka is started before starting any OMAG servers, for example in Kubernetes you may add an init container into a pod running Egeria's [OMAGServerPlatform](/concepts/omag-server-platform) to validate Kafka is ok, before the main container is launched. The [Helm charts that are provided by the Egeria team](/guides/operations/kubernetes) typically take this approach. You should also ensure that you have a highly available Kafka deployment with multiple brokers.

However, in some environments users have encountered issues where the Kafka Cluster hasn't become fully available. To mitigate this, the Kafka Open Topic Connector provides a mechanism that verifies that the Kafka Cluster brokers are available before attempting to connect.

This mechanism is controlled by two producer/consumer properties.

* `bring.up.retries` defaults to 10 and specifies the number of times the Egeria KafkaTopicConnector will retry verification before reporting a failure.
* `bring.up.minSleepTime` is set to 5000ms by default and is the minimum amount of time to wait before attempting a verification retry. 
If a Kafka verification attempt takes longer than this value the KafkaTopicConnector does not pause before retrying the verification. 

Different Kafka issues will result in differing overall times. For example, if the Kafka brokers are not resolvable (typical if waiting for a Kubernetes service to start), the Kafka library API calls will immediately fail, and so the connector will impose the minimum delay of 5s, each iteration will therefore take 5s by default. If instead the network address is unreachable, the Kafka client library will typically wait for 60s. As this is greater than 5s, no additional wait will be added.

If any capabilities of Egeria require Kafka and the topic connector cannot initialize, then the initialization of that capability will fail. For example attempting to start a server which is a member of a Cohort will fail if the Kafka Cluster is not accessible.

#### Topic Creation

Many enterprise Apache Kafka services do not allow automatic topic creation and so you need to ensure that any topics 

You will need to manually create topics of the following form

BASE_TOPIC_NAME is the value used for topicURLRoot when configuring the egeria event bus. For example, the default
value is `egeria`.

##### Cohort topics

For each cohort being used (such as `cocoCohort`):
 * BASE_TOPIC_NAME.omag.openmetadata.repositoryservices.cohort.COHORT_NAME.OMRSTopic
 
##### OMAS Topics

[Open Metadata Access Services OMASs](/services/omas) use topics to support event based interaction.

These need to be done FOR EACH SERVER configured to run one or more OMASs.
(For example for Coco Pharmaceuticals this might include `cocoMDS1`, `cocoMDS2`, `cocoMDS3` etc).

FOR EACH OMAS configured (eg Asset Consumer OMAS, Data Platform OMAS, Governance Engine OMAS etc.)

 * BASE_TOPIC_NAME.omag.server.SERVER_NAME.omas.OMAS_NAME.InTopic
 * BASE_TOPIC_NAME.omag.server.SERVER_NAME.omas.OMAS_NAME.OutTopic


One way to configure is to initially run against a kafka service which allows auto topic creation, then make note of the kafka
topics that have been created - so that they can be replicated on the restricted setup.

In addition, review the Egeria Audit Log for any events beginning OCF-KAFKA-TOPIC-CONNECTOR so that
action may be taken if for example topics are found to be missing.


--8<-- "snippets/abbr.md"


