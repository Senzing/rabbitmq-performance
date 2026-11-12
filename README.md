# rabbitmq-performance

Configuration and tuning recommendations for RabbitMQ when used with Senzing.

## Overview

RabbitMQ performs best when queues are shallow. At Senzing, we've observed severe bottlenecks when queues grow to 40M+ messages as RabbitMQ struggles to manage the index overhead. Use RabbitMQ as a **buffer**, not deep storage—if you need to queue hundreds of millions of messages, consider a system designed for that (e.g., NATS JetStream, Pulsar, or AWS SQS).

## Version Compatibility

This guide applies to both **RabbitMQ 3.x and 4.x**. The queue/exchange setup, consumer patterns,
and producer backpressure handling below are identical across both. Two things differ by major
version and are called out inline where relevant:

- **Classic queue storage mode** (see [Queue Storage Mode](#queue-storage-mode-and-disk-io))
- **Classic queue mirroring / HA** (see [High Availability](#high-availability))

If you're on RabbitMQ 3.13 with Khepri enabled, note that in-place upgrade to 4.x is not
supported — that path requires a fresh-cluster migration, not an upgrade. RabbitMQ 4.x requires
Erlang/OTP 26 or 27.

If you're deploying via Docker: Bitnami has removed `bitnami/rabbitmq` from Docker Hub entirely
(no tags remain as of this writing). Use the official `rabbitmq` image instead
(e.g. `rabbitmq:4.3-management`) — same env var conventions for default user/pass, different data
directory (`/var/lib/rabbitmq` vs Bitnami's `/bitnami/rabbitmq/mnesia`).

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

**This is a separate mechanism from per-queue disk paging below — don't conflate the two.** The
watermark is a broker-wide safety valve: it blocks *all* publishing once total broker memory/disk
crosses the threshold. It does **not** prevent an individual backlogged queue from paging its own
messages to disk well before that threshold is ever reached.

### Queue Storage Mode and Disk I/O

Classic queues page message bodies to disk once a queue's backlog grows past a small in-RAM
window, **independent of delivery mode (persistent vs. transient) and independent of how much
broker memory is actually available.** We measured this directly on a live queue capped at
`max-length: 50000`: at 38,886 messages queued, 37,101 (95%) were paged to disk while the broker
was using 0.52 GB of a 53.9 GB memory high watermark — nowhere near memory pressure. Raising the
watermark or adding RAM does not change this; it's routine per-queue behavior for classic queues
(CQv2), not a reaction to load.

- **RabbitMQ 3.x**: you can opt a queue into the older `x-queue-version: 1` storage mode, which
  stays RAM-resident until a real memory alarm rather than paging proactively. Given typical
  Senzing queue depths (thousands, capped by `max-length`), this comfortably fits in memory on any
  modern host.
  ```json
  {"arguments": {"x-queue-version": 1, "...": "..."}}
  ```
- **RabbitMQ 4.x**: classic queue v1 has been **removed**. A 4.0+ node automatically migrates any
  v1 queue to v2 on startup (RabbitMQ's own benchmark: ~9s for a queue with 1M 100-byte messages).
  There is no way to opt out of the paging behavior above on 4.x.

**This only matters for small messages.** Any message larger than
`queue_index_embed_msgs_below` (default 4096 bytes) is **always** written to the on-disk message
store, regardless of delivery mode, queue depth, or classic queue version — this is fundamental
classic-queue architecture, not a paging behavior you can tune around. Senzing records can be
100MB+ (large entities with extensive relationship/alias data). For those, disk writes are
unavoidable by design; keeping `max-length` low does not help. `queue_index_embed_msgs_below` is
itself configurable, but raising it to cover 100MB messages means embedding huge blobs directly in
the queue index rather than the message store — we have no evidence this is a good idea and are
not recommending it. If your workload's message sizes are consistently small (under a few KB),
the RAM-residency options above are worth using; if they're consistently large, budget disk
throughput for the message store instead of trying to avoid it.

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

## High Availability

- **RabbitMQ 3.x**: classic queue mirroring (`ha-mode`/`ha-params` policies) replicates a classic
  queue across nodes.
- **RabbitMQ 4.x**: classic queue mirroring is **removed**. Any mirroring policy becomes a no-op
  after upgrading — queues silently drop to a single replica with **no error raised**. If you rely
  on HA, migrate to **quorum queues** before or during the upgrade; see RabbitMQ's own
  [migration guide](https://www.rabbitmq.com/docs/3.13/migrate-mcq-to-qq). Note quorum queues have
  different storage semantics than classic queues — every message goes through a replicated,
  disk-backed log regardless of delivery mode, so they trade the RAM-residency option above for
  durability + HA.

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
