# 🚨 CDC for Cassandra using Pulsar
The goal of this project is to create a super simple sandbox to try out CDC for Cassandra using Pulsar in a near real-time manner.

It consists of the following components:
- [Cassandra 4.0](https://www.datastax.com/cassandra-4)
- [DataStax Change Agent for Cassandra](https://github.com/datastax/cdc-apache-cassandra)
- [Pulsar](https://pulsar.apache.org/)
- [DataStax Cassandra Source Connector for Pulsar](https://github.com/datastax/cdc-apache-cassandra)

## 1️⃣ About CDC
CDC for Cassandra used to be pretty complex to use until the release of the [DataStax CDC implementation](https://github.com/datastax/cdc-apache-cassandra) based on two components:
1. The DataStax Change Agent for Cassandra
2. The DataStax Cassandra Source Connector for Pulsar

### DataStax Change Agent for Cassandra
The role of the Change Agent is to alert on changes on candidate tables and then publish these towards Pulsar.

### DataStax Cassandra Source Connector
Then the role of the Cassandra Source Connector is to pick up on these topic, deduplicate them (as Cassandra typically runs in a distributed fashion) and the npush them on a new data topic to be consumed.

From here it is possible to send them to a sink for post processing. For instance to and Elastic Search instance or for purposes of Analytics, Machine Learning, etc.

## 2️⃣ Run the sandbox
First start the docker containers for this sandbox: Cassandra, Pulsar and the Pulsar Dashboard:
```sh
docker-compose up
```
What happens now is the following:

🅰️ **Cassandra** is started with CDC enabled and the Change Agent installed.

The Change Agent is configured using `./config/jvm-server.options` where the following line is added to configure the Agent:
```
# Enable the CDC Java Agent
-javaagent:/etc/cassandra-source-agent/agent-c4-pulsar-1.0.1-all.jar=pulsarServiceUrl=pulsar://pulsar:6650
```
This allows the change agent (`agent-c4-pulsar-1.0.1-all.jar`) to send data to Pulsar (running on docker host `pulsar` and port `6650`).

Additionally Cassandra is configured to enable CDC using `./config/cassandra.yaml`:
```yaml
# Enable / disable CDC functionality on a per-node basis. This modifies the logic used
# for write path allocation rejection (standard: never reject. cdc: reject Mutation
# containing a CDC-enabled table if at space limit in cdc_raw_directory).
cdc_enabled: true
```

🅱️ **Pulsar** is started and ready to receive data


## 3️⃣ Create a Cassandra Source in Pulsar
Now create a source based on the DataStax Cassandra Source Connector. This allows Pulsar to receive data from the Change Agent.
```sh
docker exec -it pulsar sh -c "/pulsar/bin/pulsar-admin source create \
--name cassandra-source \
--archive /var/cassandra-source-connector/pulsar-cassandra-source-1.0.1.nar \
--tenant public \
--namespace default \
--destination-topic-name public/default/data-ks1.table1 \
--parallelism 1 \
--source-config '{
    \"events.topic\": \"persistent://public/default/events-ks1.table1\",
    \"keyspace\": \"ks1\",
    \"table\": \"table1\",
    \"contactPoints\": \"cassandra\",
    \"port\": 9042,
    \"loadBalancing.localDc\": \"datacenter1\",
    \"auth.provider\": \"PLAIN\"
}'"
```
In this configuration we set the agent up to listen on:

Origin | Value
--- | ---
Keyspace | ks1
Table | table1
Host | cassandra (which is the docker hostname)
Port | 9042 (native transport protocol)
Datacenter | datacenter1

Additionally we set up the following topics:

Topic | Value | Notes
--- | --- | ---
Origin | public/default/events-ks1.table1 | This is the topic the Change Agent on Cassandra pushed data into
Destination | public/default/data-ks1.table1 | The Source Connector pushes deduplicated data into this topic which can them be consumed for instance by a sink


## 4️⃣ Consume the data from CDC
We want to see the output of destination topic on Pulsar. To do this, in a new terminal, run:
```sh
docker exec -it pulsar sh -c "/pulsar/bin/pulsar-client consume public/default/data-ks1.table1 -s 'ks1-table1' -n 60 -r 1"
```
Now watch this space for incoming messages managed by the Source Connector.

## 5️⃣ Create some data in Cassandra
In a new terminal, run:
```sh
docker exec -it cassandra sh -c "cqlsh -e \
CREATE KEYSPACE ks1 WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1}; \
CREATE TABLE ks1.table1 (a int, b text, PRIMARY KEY(a)) WITH cdc=true; \
INSERT INTO ks1.table1 (a, b) VALUES ( 1, 'one'); \
INSERT INTO ks1.table1 (a, b) VALUES ( 2, 'two'); \
INSERT INTO ks1.table1 (a, b) VALUES ( 3, 'three');"
```
And watch the data being made available in the `public/default/data-ks1.table1` destination topic.

## 6️⃣ Start dreaming of your new use-case
Now that CDC is working and available in a scalable and robust way it's your turn to start dreaming of your new use case.

For instance, think about crowd control based on passenger data streaming in in real-time.

Or what about providing real time updates on the location of parcel delivery.

The use-cases are endless!
