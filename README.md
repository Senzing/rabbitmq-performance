# rabbitmq-performance

Configuration and tuning recommendations for RabbitMQ when used with Senzing.

## Overview

RabbitMQ performs best when queues are shallow. At Senzing, we've observed severe bottlenecks when queues grow to 40M+ messages as RabbitMQ struggles to manage the index overhead. Use RabbitMQ as a **buffer**, not deep storage—if you need to queue hundreds of millions of messages, consider a system designed for that (e.g., NATS JetStream, Pulsar, or AWS SQS).

## Broker Configuration

### Consumer Timeout

RabbitMQ's default `consumer_timeout` (30 minutes in recent versions) will close connections if a message takes too long to acknowledge. This can happen with huge entities.

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

For multi-threaded consumers, set prefetch to match your thread count:
```python
ch.basic_qos(prefetch_count=num_threads)
```

### Threading Pattern

Use an executor pool for parallel processing, but **ack from the main thread only**:
```
1. Main thread: fetch messages via consume() with inactivity_timeout
2. Executor pool: submit processing tasks
3. Main thread: poll futures, ack/reject on completion
```

This avoids Pika thread-safety issues and keeps the channel state consistent.
```python
with concurrent.futures.ThreadPoolExecutor(max_workers) as executor:
    ch.basic_qos(prefetch_count=max_workers)
    futures = {}
    
    while True:
        # Poll for completed work
        done, _ = concurrent.futures.wait(
            futures, timeout=10, return_when=concurrent.futures.FIRST_COMPLETED
        )
        
        for fut in done:
            msg = futures.pop(fut)
            try:
                fut.result()
                ch.basic_ack(msg.delivery_tag)
            except (SzBadInputError, SzRetryTimeoutExceededError):
                ch.basic_reject(msg.delivery_tag, requeue=False)  # -> DLQ
        
        # Fetch more work
        while len(futures) < max_workers:
            msg = next(ch.consume(queue, inactivity_timeout=10))
            if not msg or not msg[0]:
                break
            futures[executor.submit(process, msg)] = msg
```

### Long Record Handling

Records that take too long to process should be rejected to the DLQ to prevent blocking other work and to avoid `consumer_timeout` disconnects.
```python
LONG_RECORD = 300      # 5 minutes - log warning
REJECT_THRESHOLD = 600 # 10 minutes - reject to DLQ

for fut, msg in futures.items():
    if not fut.done():
        duration = time.time() - msg.start_time
        if duration > REJECT_THRESHOLD and not msg.acked:
            ch.basic_reject(msg.delivery_tag, requeue=False)
            msg.acked = True  # mark so we don't double-ack later
        elif duration > LONG_RECORD:
            print(f"Still processing ({duration/60:.1f} min): {msg.record_id}")
```

This pattern:
- Warns about slow records at 5 minutes
- Rejects to DLQ at 10 minutes
- Prevents double-ack/reject by tracking state
- Keeps threads from stalling on one slow record

## Producer Considerations

When `overflow: reject-publish` triggers, the broker will NACK new publishes. Producers should:

1. Enable publisher confirms
2. Handle NACKs by pausing/retrying with backoff
3. Monitor for sustained backpressure (indicates consumers can't keep up)
```python
channel.confirm_delivery()

try:
    channel.basic_publish(
        exchange='senzing-rabbitmq-exchange',
        routing_key='senzing.records',
        body=message,
        mandatory=True
    )
except pika.exceptions.NackError:
    # Queue full—backoff and retry
    time.sleep(backoff_seconds)
```

## Reference Implementation

See [sz_rabbit_consumer](https://github.com/brianmacy/sz_rabbit_consumer-v4) for a complete example implementing these patterns.

## References

- [RabbitMQ Queue Length Limits](https://www.rabbitmq.com/maxlength.html)
- [RabbitMQ Dead Letter Exchanges](https://www.rabbitmq.com/dlx.html)
- [RabbitMQ Consumer Timeout](https://www.rabbitmq.com/consumers.html#acknowledgement-timeout)
