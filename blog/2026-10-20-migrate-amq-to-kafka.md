---
slug: migrate-amq-to-kafka
title: Migrate AMQ to Kafka
authors: [joung]
tags: [kafka, amq, event-driven]
---

As our system moved from a monolith towards microservices, services needed a way to communicate without depending on one another directly. We used ActiveMQ, managed through Amazon MQ, for that messaging. Later, we moved to Kafka on Amazon MSK.

This was not simply a change of hosting provider. ActiveMQ and Kafka have different messaging models, so the migration made us revisit how we publish, consume, and operate events. Here is why we made the move, what changed, and what I would keep in mind when planning a similar migration.

{/* truncate */}

## Why move from ActiveMQ to Kafka?

Amazon MQ gave us a managed ActiveMQ broker and a familiar message-broker model. That was a reasonable starting point: services could send messages through a broker instead of calling each other directly.

As we moved further towards event-driven microservices, Kafka became a better fit for the kind of communication we wanted to build around. Kafka stores records in durable, partitioned logs. Consumers read those records at their own pace, and separate consumer groups can each process the same stream independently.

That model is useful when an event should be available to more than one downstream service, or when we want consumers to be able to catch up by reading retained events. It also gives us a common platform for event-driven workflows and stream processing.

The key point is that Kafka was not automatically better for every message. We chose it because its log-and-consumer model fit the direction of the architecture. A traditional broker can still be the right choice for workloads that need its queueing and delivery semantics.

## The important difference: queue versus log

The biggest conceptual change was moving from thinking of messages as work handed to a broker queue to thinking of events as records in a log.

With a queue, a message is typically delivered to a consumer, acknowledged, and removed from the work waiting to be processed. With Kafka, a record remains in a topic according to its retention policy. Each consumer group tracks its own position, so multiple groups can independently process the same records.

```text
ActiveMQ queue
Producer ──► Queue ──► Consumer

Kafka topic
Producer ──► Topic (retained log) ──► Consumer group A
                              └─────► Consumer group B
```

This changes what consumers are responsible for. A Kafka consumer tracks offsets and must handle the possibility of seeing a record again, for example after a restart or a failure before its progress is committed. In practice, consumers need to be designed for safe retries and duplicate delivery; moving to Kafka does not make processing exactly-once by default.

Kafka also orders records within a partition, not across an entire topic. If related events need to be processed in order, the producer needs to use a consistent key so those events are routed to the same partition. Ordering is a design decision, not something to assume globally.

## Considerations when moving from ActiveMQ to Kafka

Two requirements shaped our migration: **keep the live system running without downtime** and **preserve the ordering our scoring logic depends on**. Changing the broker was only useful if we could maintain both.

### Migrating a live system without downtime

Our system was already live, so we needed to move traffic without interrupting message processing. We brought Kafka online, switched the producer over, and kept the ActiveMQ listener available during the monitoring and rollback period:

1. **Create the Kafka topic** with the required configuration.
2. **Start the Kafka listener** and confirm it is healthy before sending it production messages.
3. **Stop publishing to ActiveMQ and start publishing to Kafka.** This switches new messages to the new path without intentionally sending each message to both brokers.
4. **Monitor the ActiveMQ queue** to confirm no new messages are arriving and that any messages already queued are draining.
5. **Keep the ActiveMQ listener running while monitoring Kafka.** If an issue appears during the cutover, switch publishing back to ActiveMQ while investigating and debugging the Kafka path.
6. **Stop the ActiveMQ listener** only after the Kafka path has been verified and the rollback window has passed.
7. **Continue monitoring Kafka.** Once the new path is stable, remove the old ActiveMQ queue and publishing/listener logic.

Keeping the ActiveMQ listener available gives us a way to back out while the old path is still available. Before stopping it, confirm that the old queue has drained and no messages remain in flight. Monitor Kafka consumer lag, processing errors, and successful downstream score updates—not just whether the listener is running. Growing lag can mean scores are delayed even when the brokers appear healthy.

#### Tip: keep message handling separate from business logic

One thing that made this migration easier was separating the **service layer** from the **controller/presentation layer**. Our scoring logic lived in the service layer, while the message listener acted as an entry point that translated an incoming message into a service call.

Instead of changing the scoring service, we added a Kafka listener alongside the existing ActiveMQ listener. Both called the same service:

```text
ActiveMQ listener ──┐
                    ├──► Scoring service
Kafka listener ─────┘
```

This kept the migration focused on the messaging boundary:

- **Controller/presentation layer:** receive and translate messages, and handle broker-specific acknowledgements or offset commits.
- **Service layer:** apply the scoring rules without depending on whether the request came from ActiveMQ or Kafka.

We could deploy the Kafka entry point before switching publishers, keep the ActiveMQ entry point available for rollback, and remove it after the cutover without rewriting the scoring logic. The practical tip is to **add a new entry point, not a second copy of the business logic**.

### Ordering and grouping messages

Our scoring flow needs related messages to be processed in strict order. A delta between snapshots needs the previous state, and a win streak depends on the order of outcomes. Processing those messages concurrently or out of order could cause race conditions and inconsistent score calculations.

The grouping key must identify the **scoring subject**—for example, a player ID, match ID, or another stable identifier for the state being calculated. We used:

- **ActiveMQ:** `JMSXGroupID`
- **Kafka:** the message key

The technology changed, but the reason for grouping did not: all events contributing to the same score must be processed sequentially.

#### How it works under the hood

##### ActiveMQ (`JMSXGroupID`)

`JMSXGroupID` is a message property that enables JMS message grouping. The broker keeps messages with the same group ID associated with one consumer at a time and delivers them in order within that group.

The broker maintains the group-to-consumer association. If a consumer disconnects, the group can be reassigned. This provides consumer affinity for a group, while allowing different groups to be processed by different consumers.

##### Kafka (message key)

The message key determines which partition receives a record. With the same partitioning strategy and partition count, records with the same key are routed to the same partition, where Kafka preserves their order:

```text
AMQ:    JMSXGroupID = player-123  ──► same consumer at a time, in order
Kafka:  key = player-123          ──► same partition, in order
```

Within a Kafka consumer group, each partition is assigned to only one consumer at a time. A rebalance can assign that partition to a different consumer, so Kafka provides partition-level ordering rather than permanent key-to-consumer affinity.

#### Implications

- **Ordering is scoped differently:** AMQ groups messages by consumer affinity; Kafka orders records within a partition.
- **Parallelism is across groups or partitions:** one group is processed sequentially, while separate groups or partitions can be processed concurrently.
- **Hot keys can limit throughput:** a busy scoring subject can become a hotspot. In Kafka, adding consumers does not increase parallelism beyond the number of partitions, and uneven key distribution can create hot partitions.
- **Choose the key from the state being updated:** use the same stable scoring-subject identifier for all events that contribute to that calculation.
- **Preserve order in the handler too:** do not process events for the same scoring subject concurrently. Partition ordering does not prevent application code from completing work out of order.
- **Design retries for stateful processing:** duplicate delivery must not apply a score change twice. A failed event also needs an explicit recovery path; skipping it and processing the next snapshot or outcome could produce the wrong score.

The cutover needs the same care. An older event still being processed from ActiveMQ could finish after a newer event from Kafka. Keeping the same scoring identifier in both systems does not create ordering across the two brokers. Before allowing newer events to update a subject's score, ensure its earlier ActiveMQ work has completed. The same concern applies when switching publishing back during rollback.

If the Kafka message key is not the right fit, alternatives depend on the required guarantee:

- **Use a separate topic** when the message flow needs independent scaling, retention, or failure handling—not just because it has a different message type.
- **Choose an explicit partition** only when fixed placement is required and the coupling to the topic's partition layout is acceptable. A stable key is generally easier to maintain as the topic scales.
- **Handle ordering in the application** with sequence numbers or version checks when strict broker-level ordering is not required. Consumers must then detect and deal with out-of-order or duplicate events.

#### Key takeaway

In both systems, correctness depends on choosing a stable grouping identifier that matches the state being calculated. In our scoring flow, we used `JMSXGroupID` with ActiveMQ and the same scoring-subject value as the Kafka message key, so events for that subject stay ordered.

## The outcome

Moving from ActiveMQ on Amazon MQ to Kafka on Amazon MSK was part of a broader shift in how our services communicate. Kafka gave us a durable event log that multiple consumer groups can read independently, which fits the event-driven architecture we were moving towards.

The two requirements stayed the same throughout: keep processing live traffic and keep score calculations correct. Separating the listeners from the service layer let us change the messaging entry point without rewriting the scoring logic. Keeping the old path available gave us a rollback option, while choosing the scoring subject as the Kafka key carried our grouping requirement into the new system.

The main lesson for me is that a messaging migration is not just a broker change. It is about preserving the behavior the application depends on while changing how messages reach it.
