# LogDna System Design

This is my best guess as at how LogDna handles log injest and search. There as likely been many adaptations to the system based off how the business requirements change and the system behaves. I've used some of the user guides and jobs descriptions to get an understanding of what the system design could be like. 
(This document is a bit unrefined so there are likely errors)

### Business requirements
Log injest much have low latency and high throughput.
Logs must be searchable within 1 second with 10 seconds of tail latency
Logs must remain searchable for a configurable period of time

Notes:
Logs searches tend to be time based. 
High indexing workload.
Log GUID should be based on the timestamp customer and app. Possibly additional atributes. 
## Design Diagram

![System Diagram](/diagram.png)
All servcies could run on AWS solutions such as EKS, ECS and Lambda. 
### Parsing service
This service is responsible for parsing the json input string from the external clients. There may be mulitple input formats but should produce a one format. Data for this service is sort lived and may present issues with by causing a high amount of garbage collection. If the language used handles garbage collection on it's own then there may be periods of higher latency while the machine cleans up the memory that is no longer needed. 

From reading the documentation it looks like the parsing service is written in nodejs. While nodejs is very quick to setup it does come with some downsides that could cause issues for this particular use case. Namely garbage collection and dynamic typing. Because JS is dynamicly typed it speeds resources and type trying to infer the type of object you pass in. This leads to high ram and cpu usage. Garbage collection requires the service to hault all processing to free memory it is not longer using back to the OS. 

[A great read on Discord switching from golang to rust for one of their services](https://discord.com/blog/why-discord-is-switching-from-go-to-rust)



### Message Streaming
This is a great use case for Kafka/Data streaming. Kafka implements the pub/sub pattern. In my design, the parsing service produces messeages on the Parsing stream, the Filtering service consumes the Parsing stream and produces messages on the Filtering stream, and the Index service consumes the Filtering stream. 

Key features:
1. in-order messaging
2. message retention
3. allows for multiple consumers

Features 1 and 2 would would allow us to replay events/messages in the event there is a failure we need to correct. 
Feature 3 would allow us to split out the Index service into 2 different services. One that would writ to Dynamo and one that would write to ES. In a similar way, new funtionality could be added by listening for events on any give stream. An example would be alerting.

### Elastic Search
[ILM](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)

This feature give us a simple way to implement retention periods which is a critical feature. 

[Ultrawarm ES instance](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ultrawarm.html)

This feature would allow us to push indexes that get few requests off to cheaper instances/storage

[Near realtime search](https://www.elastic.co/guide/en/elasticsearch/reference/master/near-real-time.html)
Based on the business requriements these messages should not be spending much time in the other services. We need to get the logs into ES before the next 1 second refresh happens.

It is important to implement some sort of rate limiting when writing to elastic search. Search latency can take a huge hit when large amounts of write requests are coming in. 

### Storage
ES does not have strong guarantees around data loss. ES uses a filesytem cache so not all changes make it to disk. AWS does instance level backups by default. This is a major reason for written to dynamo db. Dynamo would likely be used for exporting logs. 

### Comments on how this architecture would scale
While this design does not explicitly handle a mulit region requirement the existance of the API gateway and dynamo would aid in this. Dynamo offers cross region  replication and API gateway could be used to route customers to the correct region.  

Kafka topics do not auto scale but increasing resources the avaliable resources is pretty straight foward. 

Services could scale based on message volume, cpu, or other metrics. Sytem logs tend to have huge spikes in volume when things go wrong. EKS is a great solution to host these services. 