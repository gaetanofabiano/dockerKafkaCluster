# dockerKafkaCluster

A `debian:stretch` based [KAFKA](http://kafka.apache.org/) container. 
Use it in a standalone cluster with the accompanying `docker-compose.yml`.

This project is not aimed to be considered an enviorments for production products. 

It is for study and demo purpose only


## docker-compose example

To create a simplistic standalone cluster (3 Broker) with [docker-compose](http://docs.docker.com/compose): Image provided by wurstmeister

    docker-compose up

## interact with the cluster

After run the 

    docker-compose up

Use the docker ps to retreive the ID of the master node conteiner <MASTER_CONTAINER_ID>:

    docker ps

For connecting via console to the master node use:

    docker exec -it <MASTER_CONTAINER_ID> bash

For example, if my id is 59d5d229a103

    docker exec -it 59d5d229a103 bash

The image is available directly from [Docker Hub](https://hub.docker.com/r/wurstmeister/kafka/)

Tags and releases
-----------------

All versions of the image are built from the same set of scripts with only minor variations (i.e. certain features are not supported on older versions). The version format mirrors the Kafka format, `<scala version>-<kafka version>`. Initially, all images are built with the recommended version of scala documented on [http://kafka.apache.org/downloads](http://kafka.apache.org/downloads). Available tags are:

- `2.13-2.7.0`
- `2.13-2.6.0`
- `2.12-2.5.0`
- `2.12-2.4.1`
- `2.12-2.3.1`
- `2.12-2.2.2`
- `2.12-2.1.1`
- `2.12-2.0.1`
- `2.11-1.1.1`
- `2.11-1.0.2`
- `2.11-0.11.0.3`
- `2.11-0.10.2.2`
- `2.11-0.9.0.1`
- `2.10-0.8.2.2`




## Advertised port

If the required advertised port is not static, it may be necessary to determine this programatically. This can be done with the `PORT_COMMAND` environment variable.

```
PORT_COMMAND: "docker port $$(hostname) 9092/tcp | cut -d: -f2"
```

This can be then interpolated in any other `KAFKA_XXX` config using the `_{PORT_COMMAND}` string, i.e.

```
KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://1.2.3.4:_{PORT_COMMAND}
```

## Listener Configuration

It may be useful to have the [Kafka Documentation](https://kafka.apache.org/documentation/) open, to understand the various broker listener configuration options.

Since 0.9.0, Kafka has supported [multiple listener configurations](https://issues.apache.org/jira/browse/KAFKA-1809) for brokers to help support different protocols and discriminate between internal and external traffic. Later versions of Kafka have deprecated ```advertised.host.name``` and ```advertised.port```.

**NOTE:** ```advertised.host.name``` and ```advertised.port``` still work as expected, but should not be used if configuring the listeners.

### Example

The example environment below:

```
HOSTNAME_COMMAND: curl http://169.254.169.254/latest/meta-data/public-hostname
KAFKA_ADVERTISED_LISTENERS: INSIDE://:9092,OUTSIDE://_{HOSTNAME_COMMAND}:9094
KAFKA_LISTENERS: INSIDE://:9092,OUTSIDE://:9094
KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INSIDE:PLAINTEXT,OUTSIDE:PLAINTEXT
KAFKA_INTER_BROKER_LISTENER_NAME: INSIDE
```

Will result in the following broker config:

```
advertised.listeners = OUTSIDE://ec2-xx-xx-xxx-xx.us-west-2.compute.amazonaws.com:9094,INSIDE://:9092
listeners = OUTSIDE://:9094,INSIDE://:9092
inter.broker.listener.name = INSIDE
```

### Rules

* No listeners may share a port number.
* An advertised.listener must be present by protocol name and port number in the list of listeners.

## Broker Rack

You can configure the broker rack affinity in different ways

1. explicitly, using ```KAFKA_BROKER_RACK```
2. via a command, using ```RACK_COMMAND```, e.g. ```RACK_COMMAND: "curl http://169.254.169.254/latest/meta-data/placement/availability-zone"```

In the above example the AWS metadata service is used to put the instance's availability zone in the ```broker.rack``` property.

## JMX

For monitoring purposes you may wish to configure JMX. Additional to the standard JMX parameters, problems could arise from the underlying RMI protocol used to connect

* java.rmi.server.hostname - interface to bind listening port
* com.sun.management.jmxremote.rmi.port - The port to service RMI requests

For example, to connect to a kafka running locally (assumes exposing port 1099)

      KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.rmi.server.hostname=127.0.0.1 -Dcom.sun.management.jmxremote.rmi.port=1099"
      JMX_PORT: 1099

Jconsole can now connect at ```jconsole 192.168.99.100:1099```

## Docker Swarm Mode

The listener configuration above is necessary when deploying Kafka in a Docker Swarm using an overlay network. By separating OUTSIDE and INSIDE listeners, a host can communicate with clients outside the overlay network while still benefiting from it from within the swarm.

In addition to the multiple-listener configuration, additional best practices for operating Kafka in a Docker Swarm include:

* Use "deploy: global" in a compose file to launch one and only one Kafka broker per swarm node.
* Use compose file version '3.2' (minimum Docker version 16.04) and the "long" port definition with the port in "host" mode instead of the default "ingress" load-balanced port binding. This ensures that outside requests are always routed to the correct broker. For example:

```
ports:
   - target: 9094
     published: 9094
     protocol: tcp
     mode: host
```

Older compose files using the short-version of port mapping may encounter Kafka client issues if their connection to individual brokers cannot be guaranteed.

See the included sample compose file ```docker-compose-swarm.yml```

## Release process

See the [wiki](https://github.com/wurstmeister/kafka-docker/wiki/ReleaseProcess) for information on adding or updating versions to release to Dockerhub.



