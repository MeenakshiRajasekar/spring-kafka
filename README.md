1. Generate Cluster ID

Generate a cluster ID:

.\bin\windows\kafka-storage.bat random-uuid

Example:

LJUsjEtmT5KBwP4984btBw

Use the same cluster ID when formatting all controllers and brokers.

For an existing formatted cluster, do not generate a new cluster ID.

2. Controller Configuration
Controller 1

config/controller1.properties

process.roles=controller
node.id=1

listeners=CONTROLLER://:9093
controller.listener.names=CONTROLLER

controller.quorum.voters=1@localhost:9093,2@localhost:9193

listener.security.protocol.map=CONTROLLER:PLAINTEXT

log.dirs=C:/kafka/kafka_2.13-4.3.1/data/controller1-logs

num.partitions=3
default.replication.factor=2
Controller 2

config/controller2.properties

process.roles=controller
node.id=2

listeners=CONTROLLER://:9193
controller.listener.names=CONTROLLER

controller.quorum.voters=1@localhost:9093,2@localhost:9193

listener.security.protocol.map=CONTROLLER:PLAINTEXT

log.dirs=C:/kafka/kafka_2.13-4.3.1/data/controller2-logs

num.partitions=3
default.replication.factor=2
3. Broker Configuration
Broker 1

config/broker1.properties

process.roles=broker
node.id=3

controller.quorum.voters=1@localhost:9093,2@localhost:9193

listeners=BROKER://:9092
advertised.listeners=BROKER://localhost:9092

inter.broker.listener.name=BROKER
controller.listener.names=CONTROLLER

listener.security.protocol.map=CONTROLLER:PLAINTEXT,BROKER:PLAINTEXT

log.dirs=C:/kafka/kafka_2.13-4.3.1/data/broker1-logs

log.retention.hours=168
log.segment.bytes=1073741824

offsets.topic.replication.factor=2
Broker 2

config/broker2.properties

process.roles=broker
node.id=4

controller.quorum.voters=1@localhost:9093,2@localhost:9193

listeners=BROKER://:9192
advertised.listeners=BROKER://localhost:9192

inter.broker.listener.name=BROKER
controller.listener.names=CONTROLLER

listener.security.protocol.map=CONTROLLER:PLAINTEXT,BROKER:PLAINTEXT

log.dirs=C:/kafka/kafka_2.13-4.3.1/data/broker2-logs

log.retention.hours=168
log.segment.bytes=1073741824

offsets.topic.replication.factor=2
4. Format Kafka Storage

Use the same cluster ID for every node.

Controller 1
.\bin\windows\kafka-storage.bat format `
  -t LJUsjEtmT5KBwP4984btBw `
  -c .\config\controller1.properties
Controller 2
.\bin\windows\kafka-storage.bat format `
  -t LJUsjEtmT5KBwP4984btBw `
  -c .\config\controller2.properties
Broker 1
.\bin\windows\kafka-storage.bat format `
  -t LJUsjEtmT5KBwP4984btBw `
  -c .\config\broker1.properties
Broker 2
.\bin\windows\kafka-storage.bat format `
  -t LJUsjEtmT5KBwP4984btBw `
  -c .\config\broker2.properties

If Kafka reports:

Log directory ... is already formatted.

the node has already been initialized. Do not reformat an already-running/initialized node unnecessarily.

5. Start Controller 1

Open a separate PowerShell window:

cd C:\kafka\kafka_2.13-4.3.1

.\bin\windows\kafka-server-start.bat .\config\controller1.properties

Controller 1:

Node ID: 1
Port: 9093

Leave this terminal running.

6. Start Controller 2

Open another PowerShell window:

cd C:\kafka\kafka_2.13-4.3.1

.\bin\windows\kafka-server-start.bat .\config\controller2.properties

Controller 2:

Node ID: 2
Port: 9193

Leave this terminal running.

7. Verify Controller Ports

Check Controller 1:

Test-NetConnection localhost -Port 9093

Check Controller 2:

Test-NetConnection localhost -Port 9193

Expected:

TcpTestSucceeded : True
8. Check KRaft Metadata Quorum

Use the Windows .bat command:

.\bin\windows\kafka-metadata-quorum.bat `
  --bootstrap-controller localhost:9093 `
  describe --status

Example output:

ClusterId:              LJUsjEtmT5KBwP4984btBw
LeaderId:               1
LeaderEpoch:            1
HighWatermark:          650
MaxFollowerLag:         0
MaxFollowerLagTimeMs:   0
CurrentVoters:          [{"id": 1, ...}, {"id": 2, ...}]
CurrentObservers:       [{"id": 3, ...}, {"id": 4, ...}]
Important fields
LeaderId – current KRaft controller leader.
CurrentVoters – controllers participating in the metadata quorum.
CurrentObservers – brokers connected to the controller quorum.
HighWatermark – committed position of the KRaft metadata log.
MaxFollowerLag – maximum metadata replication lag among followers.

The KRaft metadata High Water Mark is different from the High Water Mark of a Kafka topic partition.

9. Start Broker 1

Open another PowerShell window:

cd C:\kafka\kafka_2.13-4.3.1

.\bin\windows\kafka-server-start.bat .\config\broker1.properties

Broker 1:

Node ID: 3
Port: 9092
10. Start Broker 2

Open another PowerShell window:

cd C:\kafka\kafka_2.13-4.3.1

.\bin\windows\kafka-server-start.bat .\config\broker2.properties

Broker 2:

Node ID: 4
Port: 9192

Keep both broker terminals running.

11. Verify Broker Ports

Broker 1:

Test-NetConnection localhost -Port 9092

Broker 2:

Test-NetConnection localhost -Port 9192

Expected:

TcpTestSucceeded : True
12. Create a Topic

Create a topic with 2 partitions and replication factor 2:

.\bin\windows\kafka-topics.bat `
  --bootstrap-server localhost:9092 `
  --create `
  --topic order-events-topic `
  --partitions 2 `
  --replication-factor 2

Expected:

Created topic order-events-topic.
13. Describe the Topic
.\bin\windows\kafka-topics.bat `
  --bootstrap-server localhost:9092 `
  --describe `
  --topic order-events-topic

Example:

Topic: order-events-topic
PartitionCount: 2
ReplicationFactor: 2

Partition: 0
Leader: 4
Replicas: 4,3
Isr: 4,3

Partition: 1
Leader: 3
Replicas: 3,4
Isr: 3,4
Understanding the output
Partition 0
    Leader   → Broker 4
    Follower → Broker 3

Partition 1
    Leader   → Broker 3
    Follower → Broker 4

ISR means In-Sync Replicas.

For example:

Isr: 4,3

means both Broker 4 and Broker 3 are currently synchronized for that partition.

14. Start a Producer
.\bin\windows\kafka-console-producer.bat `
  --bootstrap-server localhost:9092 `
  --topic order-events-topic

Enter messages:

order-1
order-2
order-3
order-4
order-5

Press Ctrl+C to exit.

15. Read Messages Using a Consumer
.\bin\windows\kafka-console-consumer.bat `
  --bootstrap-server localhost:9092 `
  --topic order-events-topic `
  --from-beginning

Press Ctrl+C to stop the consumer.

16. High Water Mark

There are two different High Water Marks to understand.

KRaft Metadata HWM

Check:

.\bin\windows\kafka-metadata-quorum.bat `
  --bootstrap-controller localhost:9093 `
  describe --status

Example:

HighWatermark: 650

This represents the committed position of the KRaft metadata log.

Partition HWM

The partition-level HWM is associated with replicated topic data.

For a partition:

Producer
   |
   v
Leader Broker
   |
   +------> Follower Broker
              |
              v
         Replica catches up
              |
              v
         HWM advances

The HWM advances as records become safely replicated according to Kafka's replication rules.

Cluster Startup Order

For this setup, start the nodes in this order:

1. Controller 1
       |
2. Controller 2
       |
3. Verify KRaft quorum
       |
4. Broker 1
       |
5. Broker 2
       |
6. Verify broker ports
       |
7. Create topic
       |
8. Produce messages
       |
9. Consume messages
Important Windows Troubleshooting
wmic is not recognized

Some newer Windows installations don't include wmic.

Kafka's Windows startup script may contain:

wmic os get osarchitecture | find /i "32-bit" >nul 2>&1

If this prevents Kafka from starting, edit:

bin\windows\kafka-server-start.bat

and remove/comment the WMIC architecture detection line.

The PROCESSOR_ARCHITECTURE environment variable can be used instead.

.lock File Error

If you see:

Failed to acquire lock on file .lock
A Kafka instance in another process or thread is using this directory.

it usually means another Kafka process is already using that log directory.

Check Java processes:

Get-Process java -ErrorAction SilentlyContinue

Find the Kafka command:

Get-CimInstance Win32_Process -Filter "Name = 'java.exe'" |
Select-Object ProcessId, CommandLine

Do not start the same controller or broker twice using the same log.dirs.

No readable meta.properties files found

This generally means the configured log.dirs has not been formatted.

Format the appropriate node:

.\bin\windows\kafka-storage.bat format `
  -t <CLUSTER_ID> `
  -c .\config\<node>.properties
Useful Commands

Check Java:

java -version

Check running Kafka Java processes:

Get-CimInstance Win32_Process -Filter "Name = 'java.exe'" |
Select-Object ProcessId, CommandLine

Check Controller 1:

Test-NetConnection localhost -Port 9093

Check Controller 2:

Test-NetConnection localhost -Port 9193

Check Broker 1:

Test-NetConnection localhost -Port 9092

Check Broker 2:

Test-NetConnection localhost -Port 9192
Cluster Summary
Component	Node ID	Port	Role
Controller 1	1	9093	KRaft Controller
Controller 2	2	9193	KRaft Controller
Broker 1	3	9092	Kafka Broker
Broker 2	4	9192	Kafka Broker

Sample topic:

Topic: order-events-topic
Partitions: 2
Replication Factor: 2

This setup provides a local Kafka KRaft environment for learning:

KRaft architecture
Controller quorum
Broker registration
Topic partitions
Leader/follower replication
ISR
Producer/consumer flow
High Water Mark
Kafka cluster administration
