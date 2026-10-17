---
slug: share-a-kafka-topic-only-when-order-matters
title: Share a Kafka Topic Only When Order Matters
authors: [joung]
tags: [kafka]
---

We merged Kafka topics because the same services consumed the messages together. It felt simpler. Two incidents showed what we had really done: one shared topic means one shared queue, one failure domain, and one on-call problem—not a tidy folder in a repo.

My default rule: **share a topic only when order matters** for a clear stream (same partition key, same lifecycle). If urgency, retries, batch vs live traffic, or ownership differ, split the topics. Below I explain why we merged, what broke in production, when one topic is still right, and defaults that help teams avoid the same mistakes.

{/* truncate */}

## Why we merged topics

We had several topics. The same services read the events. Dashboards grouped them. We used the same domain words (“orders,” “billing,” “notifications”). Merging felt reasonable:

- Fewer topic names and fewer ACL/schema subjects to manage
- One obvious place per business area
- One mental pipeline for producers and consumers

We also followed a familiar rule of thumb: **combine topics that are mostly consumed together.** If the same consumers normally subscribe to all of them, one topic can be simpler. Consumers can skip message types they do not handle—that is acceptable. Fetching from Kafka is cheap; ignoring even a large share of messages on a topic is often not a big cost. You only need a separate topic when consumers would ignore the *vast majority* of messages.

We put **message type in the header** so each record could be parsed and routed to the right handler. That made “one topic, many processes” feel safe: separate code paths without separate topic names.

That made merging feel low-risk. The incidents below showed we had still missed something.

## What that advice does not cover

Co-consumption answers *who reads the stream*. It does not spell out *what guarantees the log must provide*—ordering, retention, retries, and what happens when one message type misbehaves.

You can join streams in application code, materialized views, or analytics. A Kafka topic is not a join key. It is an append-only log with contracts for ordering and retention. Headers that distinguish message type help you **dispatch** inside the consumer; they do not give you separate partitions, consumer lag, or retry boundaries. When message types need different contracts, one topic still ties them together. You often only notice that under load or during failure—not when consumers happily skip or route messages in code.

## Incident 1: Bulk traffic blocked important work

**Setup.** Topic `billing.events`, partition key `accountId`, already handled live `payment-captured` and `invoice-issued`. Later we added `ledger-export-row` to the same topic so the export job could reuse the same pipeline and `accountId` keying. One consumer group still processed every message type in partition order.

**What happened.** After `ledger-export-row` joined `billing.events`, a heavy export run—or backlog after an outage—filled partitions for busy `accountId` values. `payment-captured` and `invoice-issued` kept flowing as before, but new records queued behind export rows on the same partitions. The consumer could not prefer the live types that had been fine on their own; offset order now applied to `ledger-export-row` too. Lag on `payment-captured` and `invoice-issued` climbed while the group chewed through export rows at a fixed rate.

**Symptoms.**

- End-to-end latency alerts on `payment-captured` / `invoice-issued` flows while `ledger-export-row` lag was high on `billing.events`
- Support reports: payments or invoices “stuck for hours” with no export job visibly “broken”
- Healthy brokers, growing consumer lag on the shared group consuming `billing.events`

**Mitigation.** We stopped the `ledger-export-row` producer, watched lag on `billing.events`, and waited until the consumer drained enough backlog that `payment-captured` and `invoice-issued` caught up. It worked, but only as a manual lever—identify the bulk source, turn it off, hope you chose the right one.

**Lesson.** Live types on a topic do not stay safe when you add bulk later. Do not park `ledger-export-row`-class export on the same partition-ordered topic as `payment-captured` and `invoice-issued` unless you accept that slow export blocks fast billing for that key. Move export back to its own topic (for example `billing.export-replay`) or isolate it with a separate consumer group.

## Incident 2: One broken process blocked everything on the topic

**Setup.** Topic `order.events`, partition key `orderId`. A **message-type** header routed each record—for example `order-placed`, `payment-authorized`, and `return-initiated`—to its own handler with different validation and idempotency rules. Separate code paths, but still **one topic** and **one consumer group** per partition, processed in offset order.

**What happened.** A `return-initiated` message was built from bad data in the database (invalid refund amount on the return record). The returns handler rejected it, the message retried, and the consumer could not commit past that offset on the partition. Later records on the same partition—including `order-placed` and `payment-authorized` for other orders that hashed to that partition—waited behind the failing `return-initiated`. Headers picked the right handler; they did not unblock the shared queue.

**Symptoms.**

- Log spikes for `return-initiated` validation failures on `order.events`
- Stalled processing for `order-placed` / `payment-authorized` on the same partition despite healthy payloads
- Retries on the bad `return-initiated` record amplifying load; on-call debating skip, seek, or redeploy on `order.events`

**Mitigation.** We fixed the invalid data in the database—the source of the bad refund amount—then replayed or retried until the `return-initiated` handler could succeed and the partition offset could move. `order-placed` and `payment-authorized` traffic resumed on that partition. Patching DB rows to unblock a Kafka partition was slow, risky, and coordination-heavy; it should not be the routine way to recover `order.events`.

**Lesson.** Routing `return-initiated` by header is not the same as isolating it. One topic still shares retry policy and poison-message risk across types—and a bad row in the database can stall unrelated message types on the same partition. A dedicated topic (for example `order.returns`) or an isolated consumer + DLQ for `return-initiated` shrinks blast radius so returns work does not hold up fulfillment or payment events.

## The rule: share a topic only when order matters

Share a topic when all of this is true:

1. Events form one logical sequence for one id (aggregate, entity, or workflow instance).
2. Consumers must process that sequence in order to stay correct—not just for easier debugging.
3. You can name the partition key for that sequence (`orderId`, `accountId`, `shardId`, …).

Kafka orders messages within a partition. It does not order across keys. It does not let you prioritize message type A over type B on the same partition.

### When one topic is the right model

Example: account lifecycle for one account.

- **Topic:** `account.events` (or your environment prefix)
- **Key:** `accountId`
- **Events:** `account-opened` → `limit-set` → `debit-applied` → `account-closed`

If you rebuild account state from the log, you need order per account. One topic, stable keys, and documented schema changes fit well.

### When the same domain still needs multiple topics

You may still say “account” or “order” in conversation—but need different guarantees:

| Concern | Split because |
| --- | --- |
| Live lifecycle vs bulk backfill | Different urgency and lag tolerance |
| Commands vs analytics fan-out | Strict order for state; best-effort for metrics/search |
| Process A vs process B handlers | Different failures, retries, and owners |
| Notifications vs ledger | Different SLOs and idempotency rules |

Example split:

- `order.lifecycle` — key = `orderId`, state machine, strict order
- `order.backfill` — bulk replay, isolated consumers, not on the live path
- `order.notifications` — key = `orderId` or `userId`, own retry/DLQ

**For experienced teams:** Draw topic boundaries around SLOs and failure domains. DDD boxes can guide naming; the log contract is what you must get right.

## Recommendations

### Do not share a topic when…

- Interactive and batch/export/replay traffic need different latency.
- One message type can poison the queue or retry forever while others are fine.
- Different teams ship on different cycles—you do not want one bad deploy to block another team’s events.
- One event type can spike 10× and define lag for everything else.
- “We read them together” is only true in a UI or report—join in the app, not in the broker.
- A **message-type header** (or similar) routes work in code—it does not replace separate topics when lag, retries, or failure domains must differ.

### Creating a new topic per feature is usually correct

Teams ask: “Will we drown in topics?” Creating topics is cheap. Coupled outages are expensive. A feature with its own producers, consumers, retention, and on-call owner deserves its own topic name and a clear ordering story—not a slot on a crowded legacy topic because the noun matches.

What grows with topic count is governance: ACLs, Schema Registry subjects, dashboards, naming rules—not a cliff where the cluster stops accepting topics. Invest there instead of minimizing topic count for its own sake.

### Defaults worth putting in team standards

- Document what the partition key means for “order.”
- One consumer group per service that owns processing; do not share groups to save machines.
- DLQ and retry policy per pipeline—not one policy for unrelated handlers on a mega-topic.
- Backfill on dedicated topics or controlled replay; do not mix with live traffic without a design review.
- Runbooks: owner, retention, key, ordering guarantee, and what to do when lag spikes—without “patch the database to unblock the partition” as step one.

### Request/reply and channel separation

In many microservice setups (including NestJS/Kafka-style patterns), request and reply use two topics. That is on purpose: different consumers, correlation rules, and scaling. Same idea—order may matter per correlation id on each channel, not “one topic for all RPC-shaped traffic.”

## Closing

We merged topics because consumption looked shared. Incident 1: bulk and urgent traffic must not share an ordered queue if recovery means stopping producers and waiting. Incident 2: unrelated processes on one topic share retries and poison messages—and fixing bad rows in the database to unblock the log is a bad habit, not a strategy.

**Default:** Share a topic only when order matters for a defined key and lifecycle. Split on urgency, failure isolation, and ownership before the next 2 a.m. drain.

This week, pick one busy shared topic and ask: mixed urgency? mixed handlers? one retry policy for everything? If yes, plan a split—the incidents above are what that review prevents.
