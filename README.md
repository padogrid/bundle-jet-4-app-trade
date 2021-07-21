# Jet App: trade-monitor
  
The `trade-monitor` app bundle installs the `trade-monitor` demo as part of the `hazelcast/hazelcast-platform-demos` GitHub repo maintained by Hazelcast.

[https://github.com/hazelcast/hazelcast-platform-demos/tree/master/banking/trade-monitor](https://github.com/hazelcast/hazelcast-platform-demos/tree/master/banking/trade-monitor)

## Installing Bundle

```console
install_bundle -download bundle-jet-4-app-trade
```

## Use Case

This use case demonstrates Jet aggregating Kafka streamed trade data in real time. It includes a trade blotter UI for monitoring the trade aggregations executed by Jet. All of the components are configured to run locally on your PC.

![Jet Trade Diagram](images/jet-trade-monitor.jpg)

## Required Software

- Java 11+
- Kafka 2.x

[Kafka Download](https://kafka.apache.org/downloads)

You must first install Kafka to run the `trade-monitor` app. To follow the steps shown in this document, you need to set the `KAFKA_HOME` environment variable to the Kafka installation directory. For example,

```console
export KAFKA_HOME=~/Padogrid/products/kafka_2.13-2.8.0
export PATH=$PATH:$KAFKA_HOME/bin
```

## Install and Build the `trade` Bundle

To build it, run the `build_app` script in the `trade` app's`bin_sh` directory as follows:

```bash
# Switch to the trade app
cd_app trade; cd bin_sh

# Build app
./build_app
```

The `build_app` script clones the [`hazelcast-platform-demos`](https://github.com/hazelcast/hazelcast-platform-demos/tree/master/banking/trade-monitor) repo in the `trade` app directory and builds its `banking/trade-monitor` demo.

## Configure Apps

To configure the demo, change directory to the demo directory as follows.

```bash
cd_app trade
cd hazelcast-platform-demos/banking/trade-monitor
```

All of the demo scripts are kept in the `src/main/scripts` directory.

The `localhost-*.sh` scripts may not work for your environment if you have more than one network interfaces. Edit the scripts and assign `HOST_IP` to `localhost` or host name as shown below.

```bash
cd_app trade
cd hazelcast-platform-demos/banking/trade-monitor/src/main/scripts
vi localhost-*.sh
```

Replace the entire `HOST_IP` routine with the following in the `localhost-*.sh` files:

```bash
HOST_IP=`hostname`
```

There is a string parsing bug in `com.hazelcast.platform.demos.banking.trademonitor.ApplicationConfig.java`. To fix it, edit `localhost-hazelcast-node.sh`.

```bash
cd_app trade
cd hazelcast-platform-demos/banking/trade-monitor/src/main/scripts
vi localhost-hazelcast-node.sh
```

Add `:5701` to the end of JAVA_ARGS line in the `localhost-hazelcast-node.sh` file as follows:

```bash
JAVA_ARGS="${JAVA_ARGS} -Dhazelcast.local.publicAddress=${HOST_IP}:5701"
```

If you want to run the Management Center, then create a Jet cluster and configure the HTTP port number to something other than 8080 which is used by the Javalin.

```bash
make_cluster -product jet
switch_cluster jet
vi etc/cluster.properties
```

Change 8080 to 8081 in `etc/cluster.properites`:

```properties
...
mc.http.port=8081
...
```

## Startup Sequence

Execute the following commands in sequence. For details, see [trade-monitor repo](https://github.com/hazelcast/hazelcast-platform-demos/tree/master/banking/trade-monitor).

```bash
# 1. Open at least six (6) terminals. This demo starts all components in the foreground
#    so that you can monitor the logs.

# 2. Start Zookeeper and Kafka
zookeeper-server-start.sh $KAFKA_HOME/config/zookeeper.properties 
kafka-server-start.sh $KAFKA_HOME/config/server.properties

# 3. Ingest data into Kafka
cd_app trade
cd hazelcast-platform-demos/banking/trade-monitor/src/main/scripts
./localhost-trade-producer.sh

# 4.1. Start a Jet node that submits the 'IngestTrades' and 'AggregateQuery' jobs
cd_app trade
cd hazelcast-platform-demos/banking/trade-monitor/src/main/scripts
./localhost-hazelcast-node.sh

# 4.2. Start Management Center. Note that the cluster name is 'grid'.
switch_cluster myjet
start_mc

# 5. Start web server
cd_app trade
cd hazelcast-platform-demos/banking/trade-monitor/src/main/scripts
./localhost-webapp.sh

# 6. Listen on Kafka 'kf_trades' topic
kafka-console-consumer.sh --from-beginning --bootstrap-server localhost:9092 --topic kf_trades

# 7. List jobs
jet -t grid@localhost:5701 list-jobs
```

|  App                  | URL                   | Comment                  | 
| --------------------- | --------------------- | ------------------------ |
| **Trade Blotter**     | http://localhost:8080 | Left join real-time data |
| **Management Center** | http://localhost:8081 | Cluster name: **grid**   |

## Teardown

Ctrl-C running processes in the order shown below.

   1. Kafka topic listener (6)
   2. Javalin web server (5)
   3. Jet node (4.1)
   4. trade-producer (3)
   5. Kafka server (2)
   6. Zookeeper (2)

Stop Management Center.

```bash
stop_mc
```

## Tips

```bash
# To clear the 'kf_trades' topic, change the retention perioid to 1 sec
kafka-topics.sh --zookeeper localhost:2181 --alter --topic kf_trades --config retention.ms=1000

# Wait 1+ sec and change back the retention period (5 days)
kafka-topics.sh --zookeeper localhost:2181 --alter --topic kf_trades --config retention.ms=432000000
```
