# rabbitmq-performance

A couple of things to keep in mind with RabbitMQ is that it performs better when it is smaller.  I've let Qs grow to 40M+ records in it and RabbitMQ ends up being a severe bottleneck as it tries to manage it.  Also, RabbitMQ has a consumer_timeout which will cause connections to get closed if a record takes too long to process (like might happen in huge entities or PostgreSQL XID pauses).

## Setup the dead letter exchange
This is a place for rejected messages to go during processing.  Messages responded to with pika.basic_reject() go here.  You don't want to size limit this.
```
Queue Name: senzing-dlx-queue
durable: yes

Exchange Name: senzing-dlx-exchange
Routing Key: senzing.deadletter
```

## Setup the main input data exchange
```
Queue Name: senzing-rabbitmq-queue
max-length: 50,000
overflow: reject-publish
DLX Exchange Name: senzing-dlx-exchange
DLX Routing Key: senzing.deadletter

Exchange Name: senzing-rabbitmq-exchange
Routing Key: senzing.records
```


