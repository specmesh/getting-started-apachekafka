# Getting started guide: Using SpecMesh with Apache Kafka

# Introduction

This guide provides the simplest way to understand how to get started with SpecMesh (think HelloWorld or Hello SpecMesh!). It will guide you through the process of writing a spec file (also known as a SpecMesh app, or Data Product). You will also learn how to deploy it, write data into the spec topics, and check storage and production metrics

## Requirements
- Access to a Apache Kafka Cluster (without security enabled)
- A Local docker environment (producers, consumers and the specmesh cli will be executed using docker containers)

## Limitations & Prerequisites
In order to keep it as simple as possible ACLs and Schemas are not used.

Instructions are provided for bash scripts (mac)

Ensure you have Java installed on your machine (preferably Java 8 or later). You can check this by running java -version in your command prompt or terminal.


# 1. Installation & setup

## Running a local Kafka Environment (optional)

1. Download Apache Kafka from the [official site](https://kafka.apache.org/downloads): this guide uses  kafka_2.12-3.4.0
1. Expand it into an appropriate location.
1. Start ZooKeeper (from the installation directory)
```bash
./bin/zookeeper-server-start.sh ./config/zookeeper.properties 
```
This will start Zookeeper on the default port (2181)
4. Start Kafka Server. <br>
   Open another terminal or command prompt window, navigate to the Kafka directory, and run the following command:
```bash
./bin/kafka-server-start.sh ./config/server.properties

 ```
The broker will be available on `localhost:9092` except we want to use the host ip address so the docker command can access it. `10.0.0.23:9092`

# Steps

## 1. Checkout this repository

`https://github.com/specmesh/getting-started-apachekafka.git`


## 2. Create or modify a SpecMesh spec file `simple_spec_demo-api.yml`

See (`resources` and `resources/schema`)

```yaml
asyncapi: '2.5.0'
id: 'urn:simple:spec_demo'
info:
  title: Simple Spec for CLI demo
  version: '1.0.0'
  description: |
    A bunch of random topics and configs
  license:
    name: Apache 2.0
    url: 'https://www.apache.org/licenses/LICENSE-2.0'
servers:
  test:
    url: test.mykafkacluster.org:8092
    protocol: kafka-secure
    description: Test broker

channels:
  _public/user_signed_up:
    bindings:
      kafka:
        envs:
          - staging
          - prod
        partitions: 3
        replicas: 1
        retention: 1
        configs:
          cleanup.policy: delete

    publish:
      summary: Inform about signup
      operationId: onSignup
      message:
        bindings:
          kafka:
            schemaIdLocation: "payload"
        schemaFormat: "application/vnd.apache.avro+json;version=1.9.0"
        contentType: "application/octet-stream"
        payload:
          $ref: "/schema/simple.schema_demo._public.user_signed_up.avsc"

  _private/user_checkout:
    bindings:
      kafka:
        envs:
          - staging
          - prod
        partitions: 3
        replicas: 1
        retention: 1
        configs:
          cleanup.policy: delete

    publish:
      summary: User purchase confirmation
      operationId: onUserCheckout
      message:
        bindings:
          kafka:
            schemaIdLocation: "payload"
        schemaFormat: "application/json;version=1.9.0"
        contentType: "application/json"
        payload:
          $ref: "/schema/simple.schema_demo._public.user_checkout.yml"


  _protected/purchased:
    bindings:
      kafka:
        partitions: 3

    publish:
      summary: Humans purchasing food - note - restricting access to other domain principles
      tags: [
        name: "grant-access:.some.other.domain.root"
      ]
      message:
        name: Food Item
        tags: [
          name: "human",
          name: "purchase"
        ]
        payload:
          type: object
          properties:
            id:
              type: integer
              minimum: 0
              description: Id of the food
            cost:
              type: number
              minimum: 0
              description: GBP cost of the food item
            human_id:
              type: integer
              minimum: 0
              description: Id of the human purchasing the food
    /london/hammersmith/transport/_public/tube:
    subscribe:
      summary: Humans arriving in the borough
      bindings:
        kafka:
          groupId: 'aConsumerGroupId'
      message:
        schemaFormat: "application/vnd.apache.avro+json;version=1.9.0"
        contentType: "application/octet-stream"
        bindings:
          kafka:
            schemaIdLocation: "header"
            key:
              type: string
        payload:
          # client should lookup this schema remotely from the schema registry - it is owned by the publisher
          $ref: "london.hammersmith.transport._public.tube.passenger.avsc"
```


### Note:
This spec will create 3 topics by prepending the id of the app to the 'owned' topics (channels that use a relative path and start with `_private`, `_protected` or `_public`):
- simple.spec_demo._public.user_signed_up
- simple.spec_demo._private.user_checkout
- simple.spec_demo._protected.purchased (notice the tags - only principles `.some.other.domain.root` will be granted access )

## 3. Provision the spec  `simple_spec_demo-api.yml`

```bash
 docker run --rm  -v "$(pwd)/resources:/app" ghcr.io/specmesh/specmesh-build-cli  provision -bs 10.0.0.23:9092  -sr http://10.0.0.23:8081 -spec /app/simple_spec_demo-api.yaml -schemaPath /app
```

SpecMesh will output a substantial amount of status text, is will include sections for topics, ACLs and schemas.


Truncated Output

```yaml
{
   "topics" : [ {
      "name" : "simple.spec_demo._public.user_signed_up",
      "state" : "CREATED",
      "partitions" : 3,
      "replication" : 1,
      "config" : {
         "cleanup.policy" : "delete"
      },
      "exception" : null,
      "messages" : ""
   }, {
      "schemas" : null,
      "acls" : [ {
```                    


> Note: if running Kafka locally, ensure the ip listener address is not `localhost`, otherwise docker cannot establish a connection - a Timeout will occur where it looks like the container process tried to connect to 127.0.0.1/localhost. The broker will return the 'leader' election address as localhost and SpecMesh will fail to connect.

### 4. Verify the spec topics were created

```bash
 docker run --rm --name test-listing  -it  confluentinc/cp-kafka:latest  /bin/bash -c "/usr/bin/kafka-topics --list --bootstrap-server 10.0.0.23:9092" 
```
Output
```bash

__consumer_offsets
_schema_encoders
_schemas
simple.spec_demo._private.user_checkout
simple.spec_demo._protected.purchased
simple.spec_demo._public.user_signed_up
```

### 4. Verify the schemas were published
```bash
curl -X GET http://10.0.0.23:8081/subjects
```
Output
```json
["simple.spec_demo._private.user_checkout-value",
   "simple.spec_demo._public.user_signed_up-value"]
```

Retrieve a single schema
```bash
curl -X GET http://10.0.0.23:8081/subjects/simple.spec_demo._private.user_checkout-value/versions/latest
```











