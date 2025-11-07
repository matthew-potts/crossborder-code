### Race Conditions in Batch LLM Assessment and Messaging Workflows
<small>*23 Feb 2025*</small>

Consider the process of a consumer reading data from a Kafka queue and invoking an API call to an LLM to perform some operation on that data. The consumer ```acks()``` the message to tell the broker that it has been processed[^1].

```python
def process(consumer: Consumer, llm_client: LLMClient) -> None:
    while True:
        msg = consumer.poll(timeout=1.0)
        if msg is None:
            continue
        response = llm_client.evaluate(msg.value())
        publish(response)
        consumer.ack(msg)
```

With large volumes of data, we'd think that we can make an improvement upon this. The implementation is synchronous and single-threaded: the consumer must 'wait' for the whole workflow to complete - i.e. the LLM assessment to return - before it can start to pick more data off the queue. 

Consider, then, the following: the consumer no longer calls the LLM client process until a certain volume of data has been read. Once a certain limit has been reached, the consumer 'flushes' its accumulated data to the LLM by invoking a batch API call, offering two obvious benefits for throughput: first, a number of evaluations can be completed in parallel by the LLM service; and second, the consumer can keep picking items off the queue whilst the evaluation occurs.

```python
import threading

BATCH_SIZE = 100

def process(consumer: Consumer, llm_client: LLMClient) -> None:
    buffer = []
    while True:
        msg = consumer.poll(timeout=1.0)
        if msg is None:
            continue
        buffer.append(msg)
        if len(buffer) >= BATCH_SIZE:
            batch = buffer
            buffer = []
            thread = threading.Thread(
                target=evaluate_and_publish,
                args=(consumer, llm_client, batch),
            )
            thread.start()

def evaluate_and_publish(consumer: Consumer, llm_client: LLMClient, batch: list) -> None:
    responses = llm_client.evaluate_batch(batch)
    for msg, response in zip(batch, responses):
        publish(response)
        consumer.ack(msg)
```

This improves throughput, but introduces a classic problem in distributed systems: the ```evaluate_and_publish()``` step on the background thread now only ```acks()``` once the entire batch has been processed, introducing a longer delay between the consumer receiving the message and sending the acknowledgement.

In particular, if the configured ```_ACK_TIMEOUT``` is less than the time it takes for ```evaluate_batch(batch)``` to execute, then the broker will redeliver messages already being processed by ```llm_client``` - a case of duplication.

#### Keeping track of duplicates

Let's try and prevent this duplication by introducing a ```DedupTracker``` class, accessible to both the main and background processes, to hold data about messages that have already been seen.

```python
import threading

BATCH_SIZE = 100

def process(consumer: Consumer, llm_client: LLMClient, dedup: DedupTracker) -> None:
    ...
        if dedup.is_seen(msg.key()): # <-- only process messages not seen
            consumer.ack(msg)
            continue
        buffer.append(msg)
        ...

def evaluate_and_publish(
    consumer: Consumer, llm_client: LLMClient, dedup: DedupTracker, batch: list
) -> None:
    responses = llm_client.evaluate_batch(batch)
    for msg, response in zip(batch, responses):
        dedup.mark_seen(msg.key()) # <-- mark the message as seen after processing
        consumer.ack(msg)
```

However, this doesn't really solve the problem. Again, if the background batch does not complete before the ```_ACK_TIMEOUT```, ```dedup.is_seen(msg.key())``` will evaluate to ```False``` for messages still being processed, and duplication will occur when the broker redelivers. 

#### Using an 'in-flight' set

To resolve the possibility of duplication, consider creating an ```in_flight: set[str]``` variable to hold the IDs of objects that have been sent to be processed in the background.

```python
import threading

BATCH_SIZE = 100

def process(consumer: Consumer, llm_client: LLMClient, dedup: DedupTracker) -> None:
    buffer = []
    in_flight: set[str] = set() # <-- create set to store IDs
    lock = threading.Lock()

    while True:
        ...

        with lock:
            if dedup.is_seen(msg_id) or msg_id in in_flight:
                consumer.ack(msg) # <-- if message has been seen OR it's currently being processed
                continue 
            in_flight.add(msg_id) # <-- if not, add to in_flight

        ... # <-- submit work to background thread

def evaluate_and_publish(
    consumer: Consumer,
    llm_client: LLMClient,
    dedup: DedupTracker,
    in_flight: set[str],
    lock: threading.Lock,
    batch: list,
) -> None:
    responses = llm_client.evaluate_batch(batch)
    for msg, response in zip(batch, responses):
        publish(response)
        with lock: # <-- ensure operations happen atomically
            dedup.mark_seen(msg.key()) 
            in_flight.discard(msg.key()) # <-- clear message from in_flight
        consumer.ack(msg)
```

Because both the main and background threads access both ```dedup``` and ```in_flight```, we use a ```lock``` to make the 'check-and-add' operations atomic and ensure thread-safety.


[^1]: Because the LLM client might fail and the producer might never receive the ```ack``` for a given message. This sequence leads to *at-least-once* delivery - a pattern favoured in modern messaging systems (e.g. PubSub, Kafka). The alternative is that the consumer ```acks``` the message *before* sending it downstream, a scenario that would guarantee *at most once* delivery.