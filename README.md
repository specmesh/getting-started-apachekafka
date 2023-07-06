# Getting started: SpecMesh with Apache Kafka
### Without security or ACLs

# Introduction

This guide provides the simplest way to understand how to get started with SpecMesh - think of it as your HelloWorld or Hello SpecMesh! It will guide you through the process of writing a SpecMesh compliant AsyncAPI spec for your 'product'. We also call this as a SpecMesh app, or a Data Product - it provides a data model your topic structure, permissions and ownership. 

You'll learn how to drive SpecMesh using the CLI commands to `provision` your app, write data into the SpecMesh created topics, and check `storage` and `consumption` metrics for building chargeback.

Read the [CLI page](https://github.com/specmesh/specmesh-build/tree/main/cli) for more details on commands and associated flags.

## Requirements
- Access to a Apache Kafka Cluster (without security enabled)
- A local docker environment (producers, consumers and the specmesh cli will be executed using docker containers)

## Limitations & Prerequisites
In order to keep it as simple as possible ACLs are not used.

Instructions are provided for bash scripts (mac)

Ensure you have Java installed on your machine (preferably Java 8 or later). You can check this by running java -version in your command prompt or terminal.


# 1. Installation & setup

## Running a local Kafka Environment (optional)

See https://github.com/specmesh/specmesh-build/tree/main/cli#quickstart-using-docker-on-the-local-machine

Note - the docker environment uses the `kafka_network`

Login to Github Container Registry
```bash
% docker login ghcr.io -u <<username>>
>> <<token>>
``` 

Optional - pull the image
```bash
% docker pull ghcr.io/specmesh/specmesh-build-cli 
sing default tag: latest
latest: Pulling from specmesh/specmesh-build-cli
128d54f2c9b1: Pull complete 
<snip>
``` 
 

# Steps

## 1. Checkout this repository

`https://github.com/specmesh/getting-started-apachekafka.git`


## 2. Create or modify a SpecMesh spec file `acme_simple_range_life_enhancer-api.yaml` 

Links: 
- [resources/acme_simple_range_life_enhancer-api.yaml](resources/acme_simple_range_life_enhancer-api.yaml)
- Short version [resources/snippet-api.yaml](resources/snippet-api.yaml)

In SpecMesh terms - this file, and what it represents, is considered to be:
- a data product (Data Mesh terminology)
- streaming api
- a policy, or a contract of shared, private and protected related data that is self governed and available to consumers
- gitops state capture (as part of a git workflow)
- part of an ecosystem of related dataflow centric apps that are domain centric (each app captures part of the businesses' functionality)


See (`resources` and `resources/schema`)

```yaml
asyncapi: '2.5.0'
id: 'urn:acme.simple_range:life_enhancer'
info:
  title: ACME Life Enhancer
  version: '1.0.0'
  description: |
   ACMEs Life enhancer records and predicts how ones life will change due to many events that are experienced - see http://acme.org/life_range for more info
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
          $ref: "/schema/acme.simple_range.life_enhancer._public.user_signed_up.avsc"

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
          $ref: "/schema/acme.simple_range.life_enhancer._public.user_checkout.yml"


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

This spec will create 3 topics by prepending the id of the app to the 'owned' topics (channels that use a relative path and start with `_private`, `_protected` or `_public`):
- acme.simple_range.life_enhancer._public.user_signed_up
- acme.simple_range.life_enhancer._private.user_checkout
- acme.simple_range.life_enhancer._protected.purchased (notice the tags - only principles `.some.other.domain.root` will be granted access )

When security is enabled only the appropriate principle will be granted access. For example,
only `acme.simple_range.life_enhancer` principles are
- granted read-write access to _private topics prefixed with `acme.simple_range.life_enhancer._private`
- granted read-write access to _protected prefixed with `acme.simple_range.life_enhancer._protected`
- granted read-write access to _public prefixed with `acme.simple_range.life_enhancer._public`

Note, `_protected` also grants (self-governed) access to other principles where a tag is used. For example
```yaml
  _protected/purchased:
     publish:
        tags: [
           name: "grant-access:.some.other.principle"
        ]
```

## 3. Provision the spec  `acme_simple_range_life_enhancer-api.yml`

Run the SpecMesh CLI `provision` command via Docker. A `dryRun` flag is also supported.

```bash
docker run -rm --network kafka_network  -v "$(pwd)/resources:/app" ghcr.io/specmesh/specmesh-build-cli  provision -bs kafka:9092  -sr http://schema-registry:8081 -spec /app/acme_simple_range_life_enhancer-api.yaml -schemaPath /app
```

SpecMesh will output a substantial amount of status text, it will include sections for topics, ACLs and schemas.


Truncated Output

```yaml
{
   "topics" : [ {
      "name" : "simple_range.life_enhancer._public.user_signed_up",
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


## 4. Verify the spec topics were created

```bash
 docker run --rm --network kafka_network -it  confluentinc/cp-kafka:latest  /bin/bash -c "/usr/bin/kafka-topics --list --bootstrap-server kafka:9092" 
```
Output
```bash

__consumer_offsets
_schema_encoders
_schemas
simple_range.life_enhancer._private.user_checkout
simple_range.life_enhancer._protected.purchased
simple_range.life_enhancer._public.user_signed_up
```

## 4. Verify the schemas were published
```bash
 docker run --rm --network kafka_network -it confluentinc/cp-kafka:latest curl -X GET http://schema-registry:8081/subjects
```
Output
```json
["acme.simple_range.life_enhancer._private.user_checkout-value",
   "acme.simple_range.life_enhancer._public.user_signed_up-value"]
```

Retrieve a single schema
```bash
docker run --rm --network kafka_network -it confluentinc/cp-kafka:latest  curl -X GET http://schema-registry:8081/subjects/acme.simple_range.life_enhancer._private.user_checkout-value/versions/latest
```




## 5. Write data into the public topic

Note: No ACls in this unsecured kafka, docker environment and also no schema enforcement - this is terrible. Run this command inside the existing `kafka` container.

```bash
% docker exec -it kafka /bin/bash -c "/usr/bin/kafka-console-producer --broker-list kafka:9092 --topic acme.simple_range.life_enhancer._public.user_signed_up"
>user-1
>user-2
>user-3
>user-4
```

## 6. Check `Storage` metrics for this principle. Chargeback against product owner

Run the SpecMesh CLI `storage` command via Docker

```bash
 % docker run --rm --network kafka_network -v "$(pwd)/resources:/app" ghcr.io/specmesh/specmesh-build-cli storage -bs kafka:9092 -spec /app/acme_simple_range_life_enhancer-api.yaml
```

```json
{
  "acme.simple_range.life_enhancer._private.user_checkout": {
    "offset-total": 0,
    "storage": 0
  },
  "acme.simple_range.life_enhancer._public.user_signed_up": {
    "offset-total": 4,
    "storage": 296
  }
}
```
## 6. Check `Consumption` metrics against consuming principles. Chargeback against downstream consumers

Run the SpecMesh CLI `consumption` command via Docker


Run an active consumer - leave it running
```bash
 docker exec -it kafka /bin/bash -c "/usr/bin/kafka-console-consumer --group other.domain.processor --bootstrap-server kafka:9092 --topic acme.simple_range.life_enhancer._public.user_signed_up --from-beginning" 
\ 
user-1
user-2
user-3
user-4
```

Check `Consumption` metrics to see that the `other.domain.processor` is consuming events

```bash
% docker run --rm --network kafka_network -v "$(pwd)/resources:/app" ghcr.io/specmesh/specmesh-build-cli consumption -bs kafka:9092 -spec /app/acme_simple_range_life_enhancer-api.yaml

2023-06-28 12:39:21.543 [main] INFO  org.apache.kafka.common.utils.AppInfoParser - Kafka startTimeMs: 1687955961529
{
  "acme.simple_range.life_enhancer._public.user_signed_up": {
    "id": "other.domain.processor",
    "members": [
      {
        "id": "console-consumer-ce807758-8129-457e-9bf8-af6e478993f4",
        "clientId": "console-consumer",
        "host": "/172.19.0.3",
        "partitions": [
          {
            "id": 0,
            "topic": "acme.simple_range.life_enhancer._public.user_signed_up",
            "offset": 4,
            "timestamp": -1
          },
          {
            "id": 1,
            "topic": "acme.simple_range.life_enhancer._public.user_signed_up",
            "offset": 0,
            "timestamp": -1
          },
          {
            "id": 2,
            "topic": "acme.simple_range.life_enhancer._public.user_signed_up",
            "offset": 0,
            "timestamp": -1
          }
        ]
      }
    ],
    "offsetTotal": 4
  }
}
```











