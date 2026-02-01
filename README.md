# rabbitmq-performance

Configuration and tuning recommendations for RabbitMQ when used with Senzing.

## Overview

RabbitMQ performs best when queues are shallow. At Senzing, we've observed severe bottlenecks when queues grow to 40M+ messages as RabbitMQ struggles to manage the index overhead. Use RabbitMQ as a **buffer**, not deep storage—if you need to queue hundreds of millions of messages, consider a system designed for that (e.g., NATS JetStream, Pulsar, or SQS).

## Broker Configuration

### Consumer Timeout

RabbitMQ's default `consumer_timeout` (30 minutes in recent versions) will close connections if a message takes too long to acknowledge. This can happen with huge entities or during PostgreSQL XID pauses.

For Senzing workloads with variable processing times, increase or disable this limit:
```ini
# rabbitmq.conf

# 30 minutes (in milliseconds)
consumer_timeout = 1800000

# Or disable entirely for very long processing
# consumer_timeout = infinity
```

### Memory and Disk Watermarks

Prevent RabbitMQ from accepting messages when resources are constrained:
```ini
# rabbitmq.conf
vm_memory_high_watermark.relative = 0.6
disk_free_limit.relative = 1.5
```

## Queue Setup

### Dead Letter Exchange

A destination for rejected messages during processing. Messages responded to with `basic_reject(requeue=False)` route here. Do not size-limit this queue.
```
Queue Name:     senzing-dlx-queue
Durable:        yes

Exchange Name:  senzing-dlx-exchange
Type:           direct
Routing Key:    senzing.deadletter
```

### Main Input Queue
```
Queue Name:     senzing-rabbitmq-queue
Durable:        yes
max-length:     50000
overflow:       reject-publish
x-dead-letter-exchange:     senzing-dlx-exchange
x-dead-letter-routing-key:  senzing.deadletter

Exchange Name:  senzing-rabbitmq-exchange
Type:           direct
Routing Key:    senzing.records
```

**Why 50,000 max-length?**

RabbitMQ performance degrades significantly with deep queues. This cap keeps the broker responsive. When the queue is full, `reject-publish` causes the broker to reject new messages—producers must handle this via publisher confirms and implement backpressure (pause/retry).

## Consumer Tuning

### Prefetch

For multi-threaded consumers, set prefetch to **2x your thread count**:
```python
channel.basic_qos(prefetch_count=num_threads * 2)
```

This ensures a message is always queued locally when a thread completes, eliminating broker round-trip latency between tasks.

| Prefetch Setting | Result |
|------------------|--------|
| Too low | Threads starve waiting for messages |
| Too high | One consumer hogs messages; increases redelivery volume if consumer dies |
| 2x threads | Optimal throughput with controlled buffering |

### Threading Pattern

Use an executor pool for parallel processing, but **ack from the main thread only**:
```
1. Main thread: fetch messages (basic_consume)
2. Executor pool: process messages in parallel  
3. Main thread: ack on future completion callback
```

This avoids Pika thread-safety issues and keeps the channel state consistent.
```python
# Example pattern (Pika)
executor = ThreadPoolExecutor(max_workers=num_threads)

def on_message(channel, method, properties, body):
    future = executor.submit(process_message, body)
    future.add_done_callback(
        lambda f: channel.basic_ack(delivery_tag=method.delivery_tag)
    )

channel.basic_qos(prefetch_count=num_threads * 2)
channel.basic_consume(queue='senzing-rabbitmq-queue', on_message_callback=on_message)
```

## Producer Considerations

When `overflow: reject-publish` triggers, the broker will NACK new publishes. Producers should:

1. Enable publisher confirms
2. Handle NACKs by pausing/retrying with backoff
3. Monitor for sustained backpressure (indicates consumers can't keep up)
```python
channel.confirm_delivery()

try:
    channel.basic_publish(exchange='senzing-rabbitmq-exchange',
                          routing_key='senzing.records',
                          body=message,
                          mandatory=True)
except pika.exceptions.NackError:
    # Queue full—backoff and retry
    time.sleep(backoff_seconds)
```

## References

- [RabbitMQ Queue Length Limits](https://www.rabbitmq.com/maxlength.html)
- [RabbitMQ Dead Letter Exchanges](https://www.rabbitmq.com/dlx.html)
- [RabbitMQ Consumer Timeout](https://www.rabbitmq.com/consumers.html#acknowledgement-timeout)
