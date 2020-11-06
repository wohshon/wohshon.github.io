---
layout: post
title:  AMQ Streams - MirrorMaker 2 on Openshift Container Platform
date:   2020-11-06
categories: tech
tags: amq-streams strimzi cross-site openshift
#image: /assets/article_images/2014-11-30-mediator_features/night-track.JPG
#image2: /assets/article_images/2014-11-30-mediator_features/night-track-mobile.JPG
# credit https://strimzi.io/blog/2020/03/30/introducing-mirrormaker2/
---



After trying out Red Hat data grid's cross site replication feature, another cross site scenario on Openshift Container Platform I want to explore is Kafka Mirrormaker 2.0. 

A quick intro on running Apache Kafka on Openshift with the `Strimzi` project : [Strimzi](https://strimzi.io/) is the opensource project for managing Apacje Kafka deployments on Kubernetes, it uses the operator framework; and makes the work of deploying Apache Kafka a breeze on K8s. 

**AMQ Streams** (not to be confused with the Artemis based message product, AMQ Broker) is Red Hat's distribution of the Strimzi project. Needless to say, it is packaged as a operator deployable product on Openshift Container Platform.

Totay I am going to show a simple demo of setting up MirrorMaker 2.0 (MM2) across 2 Openshift Container Platform (OCP) clusters

The setup looks like this 
<pre>
  +--------------------+                                 +-------------------+
  
  | my-cluster-source  |      <--------------------->    | my-cluster-target |
  
  +--------------------+                                 +-------------------+
  
+--------------------------+                          +-------------------------+

|        Openshift         |                          |        Openshift        |

+--------------------------+                          +-------------------------+
</pre>

For a component level view:

![source of image : https://strimzi.io/blog/2020/03/30/introducing-mirrormaker2](/assets/article_images/2020-11-06-amq-streams-mirrormaker2/2020-03-30-mirrormaker.png)


#### The Environmet:

- 2 x OCP 4.5.x clusters (let's call them cluster 1 and cluster 2)
- AMQ Streams 1.5 

##### Cluster 1
Cluster 1 will be used as the source Kafka cluster. 

The high level steps to setup Cluster 1:
1. Create a namespace, e.g. `amqstreams` is used in this demo
2. Install the AMQ Streams Operator, I won't go into the details as it is very well documented [here](https://access.redhat.com/documentation/en-us/red_hat_amq/7.7/html-single/using_amq_streams_on_openshift/index])
3. We will make use of the Operator to create the following objects:
   - Kafka Cluster
   - Kafka Topic

The CRD definitions are as follows:

In case you are new to OCP, You can save them into a yaml file and invoke `oc create -f` or copy the content to replace the original contents in the `yaml view` in the web console's operator page.

- Create the `Kafka` Object

        apiVersion: kafka.strimzi.io/v1beta1
        kind: Kafka
        metadata:
        name: my-cluster-source
        namespace: amqstreams
        spec:
        kafka:
            config:
            offsets.topic.replication.factor: 3
            transaction.state.log.replication.factor: 3
            transaction.state.log.min.isr: 2
            log.message.format.version: '2.5'
            version: 2.5.0
            storage:
            type: ephemeral
            replicas: 3
            listeners:
            external:
                type: route
            plain:
                authentiation:
                type: scram-sha-512
            tls:
                authentiation:
                type: tls
        entityOperator:
            topicOperator:
            reconciliationIntervalSeconds: 90
            userOperator:
            reconciliationIntervalSeconds: 120
        zookeeper:
            storage:
            type: ephemeral
            replicas: 3
   
- Create the `KafkaTopic` Object

        apiVersion: kafka.strimzi.io/v1beta1
        kind: KafkaTopic
        metadata:
        name: my-topic
        labels:
            strimzi.io/cluster: my-cluster-source
        namespace: amqstreams
        spec:
        config:
            retention.ms: 604800000
            segment.bytes: 1073741824
        partitions: 3
        replicas: 3
        topicName: mytopic
   
##### Cluster 2
Cluster 2 will be used as the source Kafka cluster. 

The high level steps to setup Cluster 2, which is largely similar to cluster1
1. Create a namespace, e.g. `amqstream`, there is no strict requirments to use the same name as cluster 1.
2. Same thing, install the AMQ Streams Operator
3. We will make use of the Operator to create the following objects:
   - Kafka Cluster
   - KafkaMirrorMaker2 

We will not be creating any Kafka Topic as MM2 will create a target topic for the `mytopic` in cluster1

For completeness, the CRD's yaml definitions

- Create the `Kafka` Object

        apiVersion: kafka.strimzi.io/v1beta1
        kind: Kafka
        metadata:
        name: my-cluster-target
        namespace: amqstreams
        spec:
        kafka:
            config:
            offsets.topic.replication.factor: 3
            transaction.state.log.replication.factor: 3
            transaction.state.log.min.isr: 2
            log.message.format.version: '2.5'
            version: 2.5.0
            storage:
            type: ephemeral
            replicas: 3
            listeners:
            external:
                type: route
            plain:
                authentiation:
                type: scram-sha-512
            tls:
                authentiation:
                type: tls
        entityOperator:
            topicOperator:
            reconciliationIntervalSeconds: 90
            userOperator:
            reconciliationIntervalSeconds: 120
        zookeeper:
            storage:
            type: ephemeral
            replicas: 3

- Create the `KafkaMirrorMaker2` object , I will talk more about the configuration

The yaml file you see below is a barebone configuration of mirror maker 2 (that works).

The key configuration parameter is the cluster endpoints, where I am using the OpenShift Routes as the endpoints of the bootstrap servers.
 This also means that in the `Kafka` configurations for both clusters, you will need to specify a `Route` as the external endpoint.
(Other supported external endpoints are `NodePort`, `Ingress` and `LoadBalancer`). If your clusters are running on the same OCP cluster, a `service` endpoint will work as well.

You will need to specify the secrets holding the ca certs for the respective route endpoints in the yaml file.
Refering to the component diagram above, this connect cluster will be deployed in cluster 2. In this case, I will need to create the secret `my-cluster-source-cluster-ca-cert` in cluster 2. 

There might be a better way to 'copy' secrets across namespaces, but for now, I am sticking to this:

        oc extract secret/my-cluster-source-cluster-ca-cert --keys=ca.p12 --to=- > /tmp/source-ca-certs/ca.p12
        oc extract secret/my-cluster-source-cluster-ca-cert --keys=ca.password --to=- > /tmp/source-ca-certs/ca.password
        oc extract secret/my-cluster-source-cluster-ca-cert --keys=ca.crt --to=- > /tmp/source-ca-certs/ca.crt

        oc create secret generic my-cluster-source-cluster-ca-cert  --from-file=/tmp/source-ca-certs/ca.crt --from-file=/tmp/source-ca-certs/ca.p12 --from-file=/tmp/source-ca-certs/ca.password

And here is The yaml definition for mirrormaker


        apiVersion: kafka.strimzi.io/v1alpha1
        kind: KafkaMirrorMaker2
        metadata:
        name: my-mirror-maker2
        spec:
        version: 2.5.0
        connectCluster: "my-cluster-target"
        clusters:
        - alias: "my-cluster-source"
            bootstrapServers: my-cluster-source-kafka-bootstrap-amqstreams.apps.ocpcluster1.domain.com:443
            tls: 
            trustedCertificates:
            - certificate: ca.crt
                secretName: my-cluster-source-cluster-ca-cert  
        - alias: "my-cluster-target"
            bootstrapServers: my-cluster-target-kafka-bootstrap-amqstreams.apps.ocpcluster2.domain.com:443
            tls: 
            trustedCertificates:
            - certificate: ca.crt
                secretName: my-cluster-target-cluster-ca-cert    
        mirrors:
        - sourceCluster: "my-cluster-source"
            targetCluster: "my-cluster-target"
            sourceConnector: {}


Once the pods for the MM2 object is started, it will create target Topics to map those at the source, with a naming convention of <source cluster>.<topic name>. (The default config is to map all topics using the wildcard * filter. You can define your own name filter to limit the topics being replicated)

e.g. in the diagram below, the `my-cluster-source.mytopic` and `my-cluster-source.mm2-topic` are topics generated by MM2

![Topics at target cluster](/assets/article_images/2020-11-06-amq-streams-mirrormaker2/mm2-topics-1.png)



##### Test Drive

Kafka distributions comes with their testing utilities (producers and consumers and performance testing tools) so we will be using them for a simple verification of the setup.

As we are testing from an external client perspective, some prework needs to be done.

- Setting up the java truststore for the certificates

First, we have to get the ca cert from the secrets, for both the clusters respectively.

On cluster 1, 

        oc extract secret/my-cluster-source-cluster-ca-cert --keys=ca.crt --to=- > /tmp/source-ca-certs/ca.crt

On cluster 2, 

        oc extract secret/my-cluster-target-cluster-ca-cert --keys=ca.crt --to=- > /tmp/target-ca-certs/ca.crt

Next we create the java key store using the `keytool` command:

 On cluster 1,

       keytool -import -trustcacerts -alias root -file /tmp/source-ca.crt -keystore source-truststore.jks -storepass password -noprompt   

 On cluster 2,

       keytool -import -trustcacerts -alias root -file /tmp/target-ca.crt -keystore target-truststore.jks -storepass password -noprompt   

- Sending message to the source cluster on cluster1

        kafka_2.12-2.5.0.redhat-00003/bin/kafka-console-producer.sh --bootstrap-server my-cluster-source-kafka-bootstrap-amqstreams.apps.ocpcluster1.domain.com:443 --producer-property security.protocol=SSL --producer-property ssl.truststore.password=password --producer-property ssl.truststore.location=./source-truststore.jks --topic mm2-topic
        >message 1
        >message 2
        >message 3

- Reading the messages off the target cluster on cluster 2, note the topic used here.

        kafka_2.12-2.5.0.redhat-00003/bin/kafka-console-consumer.sh  --bootstrap-server my-cluster-target-kafka-bootstrap-amqstreams.apps.ocpcluster2.domain.com:443 --from-beginning --consumer-property security.protocol=SSL --consumer-property ssl.truststore.password=password --consumer-property ssl.truststore.location=./target-truststore.jks --topic my-cluster-source.mm2-topic
        message 2
        message 1
        message 3


There you go, the replication took place. I am very interested in the kind of performance it can handle. This may be something I will investigate next. 

Hope this is useful, till I write again!

