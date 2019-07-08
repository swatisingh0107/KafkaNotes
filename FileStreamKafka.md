#Module 6: Kafka Connect Source
# Practical 1
## Objective
1. Understand how to configure a connector in standalone mode
2. Read and load a file in kafka

```
cd c/Users/swati/Box\ Sync/Independent\ Study/SpecialTopics/summer2019_swati_scalasparkpythonkafka/Kafka/
docker run --rm -it -v "$(pwd)":/practical --net=host landoop/fast-data-dev:cp3.3.0 bash
```

### Configure worker properties
```
cat > worker.properties
# from more information, visit: http://docs.confluent.io/3.2.0/connect/userguide.html#common-worker-configs
bootstrap.servers=192.168.99.100:9092
key.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=false
value.converter=org.apache.kafka.connect.json.JsonConverter
value.converter.schemas.enable=false
# we always leave the internal key to JsonConverter
internal.key.converter=org.apache.kafka.connect.json.JsonConverter
internal.key.converter.schemas.enable=false
internal.value.converter=org.apache.kafka.connect.json.JsonConverter
internal.value.converter.schemas.enable=false
# Rest API
rest.port=8086
rest.host.name=192.168.99.100
# this config is only for standalone workers
offset.storage.file.filename=standalone.offsets
offset.flush.interval.ms=10000
```
Ctrl+D to save

```
cat > file-stream-demo-standalone.properties
# These are standard kafka connect parameters, need for ALL connectors
name=file-stream-demo-standalone
connector.class=org.apache.kafka.connect.file.FileStreamSourceConnector
tasks.max=1
# Parameters can be found here: https://github.com/apache/kafka/blob/trunk/connect/file/src/main/java/org/apache/kafka/connect/file/FileStreamSourceConnector.java
file=demo-file.txt
topic=demo-1-standalone
```

Ctrl+D to save

### Create Kafka Topic  

```kafka-topics --create --topic demo-1-standalone  --partitions 3 --replication-factor 1 --zookeeper 192.168.99.100:2181```  

### Connect the worker and filestream to process messages
 ```connect-standalone worker.properties file-stream-demo-standalone.properties```

## Additional:
### View list of topics
```kafka-topics --zookeeper 192.168.99.100:2181 --list```

### Delete topics
```kafka-topics --zookeeper 192.168.99.100:2181 --delete --topic demo-1.0-standalone```

Open new terminal
```
docker exec -it elastic_hamilton bash #elastic_hamilton is container name
#View kafka-message in console
kafka-console-consumer --bootstrap-server 192.168.99.100:9092 --topic demo-1-standalone --from-beginning
```

### Change retention period of messages in Kafka stream
```
kafka-console-consumer.sh --bootstrap-server 192.168.99.100:9092 --new-consumer --topic demo-1-standalone --from-beginning --timeout-ms 2000
# Since no consumer is consuming messages, an error is thrown. 5 messages were processed in 2000ms
```

# Practical 2
## Objective
1. Read file in Kafka
2. Run in distributed mode on already setup kafka cluster  

```
kafka-topics --create --topic demo-2-distributed --partitions 3 --replication-factor 1 --zookeeper 192.168.99.100:2181
```
Create connector in UI

![alt text](/Images/FiletoKafkaDistributed.JPG)

Now add data to the file
```
touch demo-file.txt
echo "Sydney, Australia" >> demo-file.txt
echo "Cairo, Egypt" >> demo-file.txt
echo "Taiwan, Japan" >> demo-file.txt
kafka-console-consumer --bootstrap-server 192.168.99.100:9092  --topic demo-2-distributed --from-beginning --timeout-ms 2000
```

# Practical 3
## Objective Gather real time streaming data

**twitter-source-distributed.properties**
```
{
  "connector.class": "com.eneco.trading.kafka.connect.twitter.TwitterSourceConnector",
  "twitter.token": "1620255258-YKdlSIGuSCfxxATzThdfMoA6pkAX6AK5xupVVhO",
  "tasks.max": "1",
  "twitter.secret": "5602d5B8mmoR9CMRoTIFcXwSOW049I1hnnxyVp9RyAw11",
  "track.terms": "nike,justdoit",
  "name": "source-twitter-distributed",
  "topic": "demo-3-twitter",
  "language": "en",
  "twitter.consumersecret": "IDoHyuyoPnwmJ9AYZ9DPFtx2IsVGQNqXIwlLKU0h14uaUXF1D4",
  "value.converter": "org.apache.kafka.connect.json.JsonConverter",
  "key.converter": "org.apache.kafka.connect.json.JsonConverter",
  "twitter.consumerkey": "02Ga8ZMlZrt47HI4feVvQFrL4"
}
```

```
kafka-console-consumer --topic demo-3-twitter --bootstrap-server 192.168.99.100:9092
```
![alt text](/Images/TwitterSourceConnector.JPG)

**Additional**  
Fixed issues with the script related to schemas


# Module7: Kafka Connect Sink

ElasticSearch is an easy way to store JSON data and search across it

## Objective
1. Start an ElasticSearch instance using docker
2. Sink a topic with multiple partitions to ElasticSearch
3. Run in distributed mode with multiple tasks

# Practical1: ElasticSearch Sink Connector

```
#Start Elasticsearch Postgres
docker-compose up elasticsearch postgres
```
![alt text](/Images/StartElasticSearch.JPG)

sink-elastic-twitter-distributed configuration

```
{
  "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
  "type.name": "kafka-connect",
  "tasks.max": "2",
  "topics": "demo-3-twitter",
  "key.ignore": "true",
  "key.converter.schemas.enable": "true",
  "topic.index.map": "\"demo-3-twitter:index1\"",
  "topic.key.ignore": "true",
  "value.converter.schemas.enable": "true",
  "name": "sink-elastic-twitter-distributed",
  "value.converter": "org.apache.kafka.connect.json.JsonConverter",
  "connection.url": "http://elasticsearch:9200",
  "key.converter": "org.apache.kafka.connect.json.JsonConverter",
  "topic.schema.ignore": "true"
}
```

Run http://192.168.99.100:9200/_plugin/dejavu in browser
### Queries
**View Retweets**
```
{
  "query":{
    "term" : {
      "is_retweet" : true
		}
  }
}
```

**High Friends count**
```
{
  "query":{
    "range" : {
      "user.friends_count" : {
          "gt" : 500
      }
		}
  }
}

```

# Additional Queries
**Tweets with hashtags**
```
{
  "query":{
    "exists":{
      "field" : "entities.hashtags.text"
    }
  }
}
```

**Most followed user**
```
{
  "query":{
    "range" : {
      "user.followers_count": {
          "gt" : 50000
     }
		}
  }
}
```
![alt text](/Images/ESQueryView.JPG)

# Practical 2: Kafka connect Rest API
## Objective
Perform standard commands on creating/modifying/deleting and viewing connector configuration  


**Get worker information**  
```curl -s 192.168.99.100:8083/ | jq```  

{  
  "version": "0.11.0.0-cp1",  
  "commit": "5cadaa94d0a69e0d"  
}  

**List connector available on a worker**  
``` curl -s 192.168.99.100:8083/connector-plugins | jq```  

**Ask about active connectors**  
``` curl -s 192.168.99.100:8083/connectors | jq ```  

[  
  "sink-elastic-twitter-distributed",  
  "source-twitter-distributed",  
  "file-stream-demo-distributed"  
]  

**Get information about a connector tasks and config**  
``` curl -s 192.168.99.100:8083/connectors/source-twitter-distributed/tasks | jq```
```
[  
  {  
    "id": {  
      "connector": "source-twitter-distributed",  
      "task": 0  
    },  
    "config": {  
      "connector.class":   "com.eneco.trading.kafka.connect.twitter.TwitterSourceConnector",  
      "twitter.token": "1620255258-YKdlSIGuSCfxxATzThdfMoA6pkAX6AK5xupVVhO",  
      "tasks.max": "1",  
      "twitter.secret": "5602d5B8mmoR9CMRoTIFcXwSOW049I1hnnxyVp9RyAw11",  
      "track.terms": "nike,justdoit",  
      "language": "en",  
      "task.class": "com.eneco.trading.kafka.connect.twitter.TwitterSourceTask",  
      "name": "source-twitter-distributed",  
      "topic": "demo-3-twitter",  
      "twitter.consumersecret":   "IDoHyuyoPnwmJ9AYZ9DPFtx2IsVGQNqXIwlLKU0h14uaUXF1D4",  
      "value.converter": "org.apache.kafka.connect.json.JsonConverter",  
      "key.converter": "org.apache.kafka.connect.json.JsonConverter",  
      "twitter.consumerkey": "02Ga8ZMlZrt47HI4feVvQFrL4"  
    }  
  }  
]  
```

**Get connector status**
```curl -s 192.168.99.100:8083/connectors/file-stream-demo-distributed/status | jq```  

```
{  
  "name": "file-stream-demo-distributed",  
  "connector": {  
    "state": "RUNNING",  
    "worker_id": "192.168.99.100:8083"  
  },  
  "tasks": [  
    {  
      "state": "RUNNING",  
      "id": 0,  
      "worker_id": "192.168.99.100:8083"  
    }  
  ]  
}  
```

**Pause / Resume a Connector (no response if the call is succesful)**

```
 curl -s -X PUT 192.168.99.100:8083/connectors/file-stream-demo-distributed/pause
 curl -s -X PUT 192.168.99.100:8083/connectors/file-stream-demo-distributed/resume
```

**Get connector configuration**  
```curl -s 192.168.99.100:8083/connectors/file-stream-demo-distributed | jq```
```
{  
  "name": "file-stream-demo-distributed",  
  "config": {  
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",  
    "key.converter.schemas.enable": "true",  
    "file": "demo-file.txt",  
    "tasks.max": "1",  
    "value.converter.schemas.enable": "true",  
    "name": "file-stream-demo-distributed",  
    "topic": "demo-2-distributed",  
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",  
    "key.converter": "org.apache.kafka.connect.json.JsonConverter"  
  },  
  "tasks": [  
    {  
      "connector": "file-stream-demo-distributed",  
      "task": 0  
    }  
  ]  
}  
```
**Delete a connector**  
``` curl -s -X DELETE 192.168.99.100/connectors/file-stream-demo-distributed```

**Create a new connector**  

```
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "file-stream-demo-distributed", "config":{"connector.class":"org.apache.kafka.connect.file.FileStreamSourceConnector","key.converter.schemas.enable":"true","file":"demo-file.txt","tasks.max":"1","value.converter.schemas.enable":"true","name":"file-stream-demo-distributed","topic":"demo-2-distributed","value.converter":"org.apache.kafka.connect.json.JsonConverter","key.converter":"org.apache.kafka.connect.json.JsonConverter"}}' http://192.168.99.100:8083/connectors | jq
```
```
{  
  "name": "file-stream-demo-distributed",  
  "config": {  
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",  
    "key.converter.schemas.enable": "true",  
    "file": "demo-file.txt",  
    "tasks.max": "1",  
    "value.converter.schemas.enable": "true",  
    "name": "file-stream-demo-distributed",  
    "topic": "demo-2-distributed",  
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",  
    "key.converter": "org.apache.kafka.connect.json.JsonConverter"  
  },  
  "tasks": [  
    {  
      "connector": "file-stream-demo-distributed",  
      "task": 0  
    }  
  ]  
}  
```

**Update a connector configuration**  
```
curl -s -X PUT -H "Content-Type: application/json" --data '{"connector.class":"org.apache.kafka.connect.file.FileStreamSourceConnector","key.converter.schemas.enable":"true","file":"demo-file.txt","tasks.max":"2","value.converter.schemas.enable":"true","name":"file-stream-demo-distributed","topic":"demo-2-distributed","value.converter":"org.apache.kafka.connect.json.JsonConverter","key.converter":"org.apache.kafka.connect.json.JsonConverter"}' 192.168.99.100:8083/connectors/file-stream-demo-distributed/config | jq
```  
```
{  
  "name": "file-stream-demo-distributed",  
  "config": {  
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",  
    "key.converter.schemas.enable": "true",  
    "file": "demo-file.txt",  
    "tasks.max": "2",  
    "value.converter.schemas.enable": "true",  
    "name": "file-stream-demo-distributed",  
    "topic": "demo-2-distributed",  
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",  
    "key.converter": "org.apache.kafka.connect.json.JsonConverter"  
  },  
  "tasks": [  
    {  
      "connector": "file-stream-demo-distributed",  
      "task": 0  
    }  
  ]  
}  
```


# Practical 3: Pstgres sink connector
## Objective
1. lanuch Postgres instance
2. Create Postgres sink connector in distributed mode
3. Execute queries in Sqlectron on twitter data

Postegres sink connector configuration
```
# Basic configuration for our connector
name=sink-postgres-twitter-distributed
connector.class=io.confluent.connect.jdbc.JdbcSinkConnector
# We can have parallelism here so we have two tasks!
tasks.max=1
topics=demo-3-twitter
# the input topic has a schema, so we enable schemas conversion here too
key.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=true
value.converter=org.apache.kafka.connect.json.JsonConverter
value.converter.schemas.enable=true
# JDBCSink connector specific configuration
# http://docs.confluent.io/3.2.0/connect/connect-jdbc/docs/sink_config_options.html
connection.url=jdbc:postgresql://postgres:5432/postgres
connection.user=postgres
connection.password=postgres
insert.mode=upsert
# we want the primary key to be offset + partition
pk.mode=kafka
pk.fields=__connect_topic,__connect_partition,__connect_offset
fields.whitelist=id,created_at,text,lang,is_retweet
auto.create=true
auto.evolve=true
```
# Additional
**POSTGRES QUERIES**
Retweets:
```
SELECT * FROM "public"."demo-3-twitter"
where is_retweet=true
LIMIT 1000
```
Original tweets that contains mentions
```
SELECT * FROM "public"."demo-3-twitter"
where text like '%@%' and is_retweet !=true
LIMIT 1000
```
Total records in dataset
```
SELECT count(*) FROM "public"."demo-3-twitter"
```
