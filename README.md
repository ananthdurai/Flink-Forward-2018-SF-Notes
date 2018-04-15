# Flink Forward  -SF 2018

## **Keynote:** Flink streaming platform:

- DA platform: paid version:

- support for the versioned stateful deployment model, aka multiple version of a job can deployed at a time

- support for reprocessing snapshots.

- containerized deployment support

- auto scaling done by taking snapshot of the previous version and stop it, deploy new application, recover the state and running the new version  (available in open source as an api, need to build tooling for it)

## **Dell:** Pravega stream storage

- pravaga support long term storage aka support hot and cold storage.
- stream writes partitioned by the user defined route key
- dynamically scale partitions based on the incoming data
- note to me:  (look into pravaga search engine feature? why anyone need it?)

## **Google:** Apache Flink & Beam

![](https://d2mxuefqeaa7sj.cloudfront.net/s_5E9C03C3CBB9D39C470121DF563CEDEC02DEACC5FE56FA6AC3814E7C4AC8D781_1523419413212_ml-flow.png)

— Apache beam support multiple language abstraction.—

- Execution model of Beam. language SDK  -\> beam intermediate model  -\> Runner
- Flink runner for Apache beam to support python SDK and Go SDK  (BEAM-2889)
- ——domain specific libraries for time series and ML——

**_Tensor Flow Transform_** **-**

a consistent In-Graph transformation in training and serving.

- using beam python SDK and can be run in Flink Runner
- Feature engineering should be consistent with training and serving layer
- It is hard because training running on a batch processing phase and the model serving mostly running in the streaming processing.
- tf.Transform() in the batch processing output a tf.Graph that emit the model serving graph.
- we can leverage tf.Graph in the serving layer with Flink to serve the request.
- GitHub.com/tensor flow/transform.

**Tensor Flow model analysis:**

- _scale-able, sliced and full pass metrics_

- it is a distributed performance analyses on the tensor flow model, slice and dice by various dimensions.

- github.com/tensor flow/model-analysis

- more detail: tensor flow model analysis blog post

- watch YouTube video [Tensor flow extended](https://www.youtube.com/watch?v=vdG7uKQ2eKk).

- paper: [Tensor Flow Based production scale machine learning platform](http://delivery.acm.org/10.1145/3100000/3098021/p1387-baylor.pdf?ip=24.4.201.34&id=3098021&acc=OPENTOC&key=4D4702B0C3E38B35%2E4D4702B0C3E38B35%2E4D4702B0C3E38B35%2E054E54E275136550&__acm__=1523415103_7762bdddee2a4f7b9ea96c2ee5871ba4) KDD\[2017\]

- Beam community on going project to read/ write and analysis Genomic Projects.

## **Yelp:** Powering data infrastructure with Flink

- 8 application.

- 100 running jobs.

- Platform design constraints

- low latency

- unordered

- high throughput

- highly partitioned

- can have partition and duplication.

paastorm is yelp python stream processor to abstract kafka producer and consumer stream processing from the users.

passtorm running single python process, good for map/ flatmap operations, not for complex joins

- The lack of stream processing leaves powerful business insight sitting idle in kafka

  ###### use cases:

- Redshift connector: batch write to csv and copy into S3.

- challenges: kafka consumer group instability may crash lot of buffered data and produce duplication.

- RedshiftConnector uses GlobalWindow to buffer the day and upload the data into S3.  (very interesting)

- unwindowed join: stateful join application with out window.

- join two stream of mysql change log table and produce a single table

- \- Streaming SQL makes the join event better

- uses yaml based abstraction to define sql and deploy streaming data pipeline

- Yelp Flink connector eco system, support S3, elastic search etc using Flink Sink

- Yelp implemented Auditors to manages the pipeline correction of each connectors.

- Yelp centralized schema registry: all streaming and pipeline going through the schema registry to validate the schema.

- A common theme coming along is to consume data from the multiple kafka brokers and topics for ease of failure. We are looking into a similar solution as well.

- running on EMR cluster. a good way to start.

- run one EMR cluster per application

- one connector is one Flink job

- Yelp custom tooling called Flink supervisor launches the Flink cluster/ restart/ deploy and alerting it.

- save state stored on S3 and checkpoint stored in the DynamoDB because of the list operation consistency issue

- use RocksDB for incremental checkpoint

- implemented a custom serializer for the complex class serialization.

- Running v1.3.2

- properly sizing the task manager and memory size, disabled virtual memory and physical memory check because EMR cluster dedicated to Flink.

## **Uber:** Building Flink as a serving platform

- Uber using Flink for the mobile notification

- 1000+ streaming jobs running in multiple data center

- Challenges:

- lack of deployment tolling for real time

- logging infrastructure, storage, monitoring

- manages the life cycle of the streaming application

- duplicate code for streaming and real time

- manual deployment/ operation and scaling

- integration with peripheral IO & micro services.

- coordinating with the multiple data centers.

- money related jobs require sensitive SLAs.

![](https://d2mxuefqeaa7sj.cloudfront.net/s_5E9C03C3CBB9D39C470121DF563CEDEC02DEACC5FE56FA6AC3814E7C4AC8D781_1523419789892_uber-flink-architecture.png)

- Business use cases:

- online ML model update for the market places

- Real time feature engineering  (payments, promotion abuse)

- Architecture of Flink as a platform:

- yarn

- Flink connector to integrate other source and sink

- monitoring integration

- Deployment:

- all the streaming jobs runs in a dedicated yarn cluster

- mesos docker container manages the life cycle of the streaming jobs

- docker client submit the job and remain attach to the job and react to the change. when the user click stop, mesos kills the application

- deployment API abstraction to abstract yarn deployment

- Operation:

- scale jobs dynamically based on the IO and resource usages.

- maintain the common set of operational tooling and batch running for automate the operation.

## **Netflix:** Scaling Flink in the cloud s3

- S3 is the snapshot store for Netflix

- S3 automatically partitions bucket if the request rate high

- avoid sequential key name of request over 100 QPS

- use random prefix in the key name, but you can’t do a prefix scan.

- attach the photo of S3.

- scaling the stateless job

- Flink Router is stateless and embarrassingly parallel

- 3 trillion events/ per day

- 2,000 routing job

- 10,000 containers

- 200,000 parallel operating instances

- checkpoint path: s3://bucket/checkpoint/hashkey/deployment-timesamp/job-id

- state.backend.fs.memory-threshold = 1024

- increase the the value so that only the job manager writes the checkpoint

- reduces the checkpoint duration by 10 times.

- Hadoop S3 adopter request with slash and without slash metadata request

- [BTrace](https://github.com/btraceio/btrace): dynamic tracing tool for java to trace the java request

- FSCheckpoint constructor calling the metadata request causing the issue. Fixed by [Flink-5800](https://issues.apache.org/jira/browse/FLINK-5800)

- Fine Grained recovery  (recovery only the depend pipeline restart) [Flink-8042](https://issues.apache.org/jira/browse/FLINK-8042) & [Flip-6](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65147077)

- As of now a best practice to handle recovery spike is to run more containers than the number of Kafka partition.

- Scaling stateful jobs:

- single job can write a large state in S3

- inject dynamic entropy in the S3 write path. Flink 9061

- Each task manager has the large states

- enable incremental checkpoint with RocksDB

- FLASH\_SSD\_OPTIMIZED enable

- taskmanager.network.memory.mb=4GB

- Read through  —save point vs checkpoint—

- checkpoint interval: 15 minutes

- Connect job Graph:

- The job’s all the operators are depended on each others,

- fine grained recovery won’t work in the shuffling job since everything is connected.

- Task local recovery  ([Flink-8360](https://issues.apache.org/jira/browse/FLINK-8360)) to prevent task recovery to check for the local disk first.

- otherwise when the task manager restart in a new cluster, attach the same EBS volume so no job recovery needed.  (It is not supported now, but more of a future wish list)

## **Lyft:** Bootstrapping the state in Flink

- consistent feature generation and serving in a consistent pipeline.

- Dryft: stream processing as a service.

- what is bootstrapping:

- calculate the initial state of the job so that stream can start running from the day 1

- observe-ability, scale-ability and stability needed to bootstrap

- bootstrapping is not back filling.

  ##### Solution:

  

  ###### stream Retention:

- use the stream technology data to retain as long as you needed so that we can reprocess it if needed.

- Infinite storage in stream technology:  (like Kafka)

- It enable reprocess the state from the beginning.

  ###### Source magic.

- Write multiple sources on Flink to read from S3  -\> Kafka  -\> continue the same pipeline.

  **Application level attempt:**

- write the bootstrap and steady state program separately.

  Application Level attempt 2:

- Read from S3 and Kafka and union it to reduce the duplication.

- have the watermark period large enough for the batch and streaming system can catch up.

## **Alibaba:** Common algorithm platform

![](https://d2mxuefqeaa7sj.cloudfront.net/s_5E9C03C3CBB9D39C470121DF563CEDEC02DEACC5FE56FA6AC3814E7C4AC8D781_1523420228598_alibaba-architecture.png)

- Alibaba computing platform: a generic platform for computing needs.
- low learning, less coding, more functionality  (design goal)
- Alibaba computing platform called Alink !!!??  (attach pic)
- The platform support experiment, data source and components.  (attach pic)
- It provide drag & drop UI, client SDK to build streaming analytics engine.
- single click button in the UI to run in local and the cluster environment.
- Alibaba to open source their algorithm platform based on Flink.

## **Uber:** Scaling Uber real-time optimization with Flink

- used in uber market place  (driver/ rider pricing, dynamic pricing, driver position processing)

- Geo/ temporal event aggregation

- online model update

  ###### Challenges:

-  Event time ordering and Time sensitive output. Event spatial mapping and Locality Sensitive mapping.

- Events are not an isolated thing once you group by a dimensions.

- initial solution is to use OLAP engine, bucketed by the event time and Geo id.

- to improve the performance from OLAP to materialize a snapshot view. a periodic cron to aggregate the events.

  ###### Solution:

- Event Driven solution.

- Geo fan out first.

- aggregate events in flink.

  ###### Advantages:

- Flexible window and the trigger strategy

- compute trigger by the events only

- Materialized results pushed to the consumer.

- Avoid SOF and better isolation and scale.

  ###### Disadvantage:

- excessive fan out

- virtual key instead of physical key

- memory management.

- per-aggregated and logic embedded into the code.

- customized job per application.

- models describe the state of the world and the decision engine act on top of it.

- batch pipeline issue: something happen to the model in the middle of the week very harmful for the system

  ###### Real-time modeling challenges:

- bootstrap with the historical data

- computation cost: require iteration and convergence.

- solved by online/ offline model. offline batch parameter learning and realtime partial parameter learning.

- Real-time ML is hard due to data per-processing in real-time.

- Reduce the number of dimension to scale real-time machine learning.

## **Data Artisans:** How to build modern stream processing

- the job converted into streaming data flow

- Stateful operator

- comparison of internal vs external state management system

  ###### challenges:

- How to take consistent snapshot without stopping the system.  \[check point barrier passed to the stateful operators from the source operator\]

- The barrier can be asynchronous which minimize the pipeline stall

- support parallel checkpoint and throttling of resource utilization

- Full checkpoint every minute for a large scale snapshot still time consuming

- Flink do incremental checkpoint, find the delta between previous checkpoint

- downside, restore need to replay all the checkpoints.

- Heap based  (MVCC HashMap) and RocksDB based state management systems.

- Flink 1.5 have Local Snapshot recovery, stored on the local machine.

  ###### Rescaling stateful application:

- How to redistribute the state since there is no horizontal communication between operators.

- Re scaling reassign key group

- ###### Data transport for the batch and streaming:

- subtask output  (bounded vs unbounded vs blocking pipeline)

- scheduling type  (next stage on complete o/p or next stage on first o/p or all at once)

- transport  (high throughput vs low latency)

- use credit based flow control to handle back pressure and increase throughput.

- based on the credit got from the receiver sender decide how much data we can send. It will reduce the receiver overloaded

![](https://d2mxuefqeaa7sj.cloudfront.net/s_5E9C03C3CBB9D39C470121DF563CEDEC02DEACC5FE56FA6AC3814E7C4AC8D781_1523420327358_credit-based-optimization.png)
