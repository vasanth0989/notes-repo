# Wire Rail DB: transaction table schema (proposal)

Status: draft for review. Nothing here is approved yet.

## 1. The short version

- One table, one item per wire, keyed by UETR.
- The item has three parts: what we received, what we sent, and where the wire stands.
- The attributes in section 4 are needed only if the table also acts as the queue. If SQS or Streams is chosen, leave section 4 out. Everything else stays the same.
- No clear account number is ever written to the table, in any attribute.
- Names carry no "OPF", so a later submission service for another channel can use the same table.

## 2. Table settings

| Setting | Value | Note |
|---|---|---|
| Table name | `bank-wire-rail-ii-transaction` | The provisioning form allows letters, numbers, underscore, dash and dot. Spaces are not allowed. |
| Partition key | `uetr`, String | The UETR as received, lower case, 36 characters. |
| Sort key | None | One item per wire. |
| Capacity | On demand | About 1,500 wires a day. Nothing to size. |
| Encryption | Platform default | As in the provisioning story. |
| Point-in-time recovery | On | |
| Time to live | Attribute `expiresAt` | Retention period is a product and records decision. See section 8. |
| Replication | Both regions | Needed for any design that survives a region loss. To confirm with the platform. |
| Streams | Off for now | Turn on with new and old images only if it is used as a wake-up or to feed reporting. |

## 3. Core attributes (needed whatever the queuing mechanism is)

### Identity

| Attribute | Type | Written by | Meaning |
|---|---|---|---|
| `uetr` | S | API | Partition key. Also the idempotency key sent to the channel. |
| `paymentId` | S | API | Payment Exchange's own id for the payment. Lets support go from a PayX screen to this record. Source field name to confirm. |
| `wireType` | S | API | `CREDIT_TRANSFER`, `CUSTOMER_DRAWDOWN`, `BANK_DRAWDOWN`. |
| `fulfillmentChannel` | S | API | `OPF` today. Later: the correspondent bank, Fed or SWIFT channel. |
| `schemaVersion` | N | API | Starts at 1. Lets readers handle old items after a change. |

### What we received

| Attribute | Type | Written by | Meaning |
|---|---|---|---|
| `paymentObject` | S | API | The payment object as received, as JSON, account numbers tokenised. Written once, never changed. |
| `paymentObjectVersion` | S | API | Version of the payment object library, for example `2.16`. |
| `paymentObjectHash` | S | API | SHA-256 of the received payload. A replay with the same UETR and a different hash is a conflict, not a retry. |

### What we sent

| Attribute | Type | Written by | Meaning |
|---|---|---|---|
| `submissionRequest` | S | Submission service | The converted request (pain.001 V11 JSON), account numbers still tokenised. Written once at the first conversion and reused on every retry, so a retry sends the same bytes. |
| `submissionFormat` | S | Submission service | `pain.001.001.11+json`. |
| `mapperVersion` | S | Submission service | Version of the mapper library that produced the request. |

### Where the wire stands

| Attribute | Type | Written by | Meaning |
|---|---|---|---|
| `status` | S | Both | See section 5. |
| `statusReason` | S | Submission service | Short code for a rejection or expiry. |
| `channelHttpStatus` | N | Submission service | Last HTTP status from the channel: 202, 400, 422, 500, 503. |
| `channelResponse` | S | Submission service | Last response body from the channel, as received. |
| `submittedAt` | S | Submission service | When the channel accepted the wire. |
| `callbackStatus` | S | Submission service | `NOT_DUE`, `PENDING`, `DELIVERED`, `FAILED`. |
| `callbackDeliveredAt` | S | Submission service | When Payment Exchange acknowledged the callback. |

### Housekeeping

| Attribute | Type | Written by | Meaning |
|---|---|---|---|
| `createdAt` | S | API | When the record was accepted. |
| `updatedAt` | S | Both | Last change. |
| `version` | N | Both | Raised by one on every write. Used for optimistic locking. |
| `acceptedRegion` | S | API | Region that accepted the wire. |
| `expiresAt` | N | API | Time to live, in epoch seconds. The only timestamp that is a number, since DynamoDB requires it. |

All other timestamps are ISO-8601 strings in UTC with milliseconds, for example `2026-11-12T18:04:05.123Z`. Fixed width, so they sort correctly as strings.

## 4. Work attributes (only if the table is the queue)

| Attribute | Type | Meaning |
|---|---|---|
| `workQueue` | S | `SUBMIT` or `CALLBACK`. Present only while work is pending. Removed when the work is done. |
| `workDueAt` | S | When the next attempt is due. Present and removed together with `workQueue`. |
| `attemptCount` | N | Submission attempts so far. |
| `callbackAttemptCount` | N | Callback attempts so far. |
| `leaseOwner` | S | Id of the worker holding the record. |
| `leaseExpiresAt` | S | When that hold lapses. A crashed worker's record frees itself. |
| `deadlineAt` | S | After this the wire is expired and an alert is raised. |
| `lastError` | S | Short text of the last failure. No payload, no account data. |

### The index

| Index | Partition key | Sort key | Projection |
|---|---|---|---|
| `work-index` | `workQueue` | `workDueAt` | Keys plus `leaseExpiresAt`, `acceptedRegion`, `attemptCount` |

- The index is sparse. An item is in it only while `workQueue` and `workDueAt` exist. A finished wire is not in it.
- On a normal day it holds a handful of items.
- A worker asks: `workQueue = SUBMIT` and `workDueAt <= now`.
- A wire needs one kind of work at a time, first submit and then callback, so one pair of attributes and one index cover both.
- If volume grows, the key becomes `SUBMIT#0` to `SUBMIT#n`. That is a configuration change, not a redesign.

### The vendor-outage marker

The shared "the channel is down" marker is not a wire, so it does not belong in a table keyed by UETR. Put it in a small second table, or in a configuration store both regions read. Decide this only once the proposal is approved.

## 5. Status values

| Status | Meaning | Final | Next |
|---|---|---|---|
| `ACCEPTED` | Stored. Not yet accepted by the channel. | No | `SUBMITTED`, `REJECTED`, `EXPIRED` |
| `SUBMITTED` | The channel returned 202. | Yes | |
| `REJECTED` | The channel refused the request as invalid. No retry. | Yes | |
| `EXPIRED` | The deadline passed before the channel accepted it. Alert raised. | Yes | |

There is no "in progress" status. A live lease says a worker is on it. That keeps status to what the business cares about.

Only these four exist until the channel's later status feed is consumed. Settlement and bank acknowledgement are not tracked here.

## 6. The three writes that matter

**Accept (API).** Put the item only if the UETR is new.

```
ConditionExpression: attribute_not_exists(uetr)
```

If the condition fails, read the item. Same `paymentObjectHash`: return its current status. Different hash: return a conflict.

**Claim (submission service).** Take the record only if it is still waiting and nobody holds it.

```
ConditionExpression: #status = :accepted
                 AND (attribute_not_exists(leaseExpiresAt) OR leaseExpiresAt < :now)
UpdateExpression:    SET leaseOwner = :me, leaseExpiresAt = :until,
                         attemptCount = attemptCount + :one, version = version + :one
```

**Record the outcome (submission service).** Only the lease holder may write it.

```
ConditionExpression: leaseOwner = :me
Accepted:   SET #status = :submitted, submittedAt = :now, channelHttpStatus = :code,
                channelResponse = :body, callbackStatus = :pending,
                workQueue = :callback, workDueAt = :now
            REMOVE leaseOwner, leaseExpiresAt
Retryable:  SET workDueAt = :nextAttempt, lastError = :text, channelHttpStatus = :code
            REMOVE leaseOwner, leaseExpiresAt
```

When the callback is delivered, `workQueue` and `workDueAt` are removed and the item leaves the index.

## 7. Example item

```json
{
  "uetr": "7a1f0c3e-5b2d-4e8a-9c41-0d6f2b7e8a15",
  "paymentId": "PX-000123456",
  "wireType": "CREDIT_TRANSFER",
  "fulfillmentChannel": "OPF",
  "schemaVersion": 1,
  "paymentObject": "{ ...payment object, account numbers tokenised... }",
  "paymentObjectVersion": "2.16",
  "paymentObjectHash": "9f2c...e1",
  "submissionRequest": "{ ...pain.001 V11 JSON, account numbers tokenised... }",
  "submissionFormat": "pain.001.001.11+json",
  "mapperVersion": "0.2.0",
  "status": "SUBMITTED",
  "channelHttpStatus": 202,
  "channelResponse": "{ ...as received... }",
  "submittedAt": "2026-11-12T18:04:06.412Z",
  "callbackStatus": "PENDING",
  "workQueue": "CALLBACK",
  "workDueAt": "2026-11-12T18:04:06.412Z",
  "attemptCount": 1,
  "callbackAttemptCount": 0,
  "deadlineAt": "2026-11-12T22:04:05.123Z",
  "acceptedRegion": "us-east-1",
  "createdAt": "2026-11-12T18:04:05.123Z",
  "updatedAt": "2026-11-12T18:04:06.412Z",
  "version": 3,
  "expiresAt": 1857600000
}
```

The ids and values are made up.

## 8. Choices made, and why

| Choice | Why |
|---|---|
| Store the payment object as received | It is the source of truth. A later submission service for another channel converts from it. This is the direction from the developer connect. |
| Store the converted request too | A retry must send the same bytes. The converted request carries a creation time, so it cannot be regenerated on each attempt. |
| Do not store the pain.001 XML | The JSON is what goes on the wire. The XML is an intermediate step. Add `submissionRequestXml` only if audit or support asks for it. |
| Payloads as JSON strings, not DynamoDB maps | The exact bytes are preserved. A map would reorder fields and change number formats. |
| No sort key | One item per wire. Attempt history goes to logs, not to the table. |
| Latest outcome only | Matches the earlier decision that the table keeps the latest status, not a retry log. |
| Generic names | The component is generic. The channel is a value, not part of a name. |

## 9. Open points

1. **Retention.** How long a record is kept. The inbound API of the other pod keeps two years. Sets `expiresAt`.
2. **Payment object field names.** Which fields give `paymentId` and `wireType`. To read from the payment object class, not guess.
3. **What the channel returns on 202.** If it carries a reference of its own, it earns a top-level attribute. Waiting on the vendor samples.
4. **Does the channel's response echo account numbers?** If yes, `channelResponse` must be masked or tokenised before it is stored.
5. **Token and the same-bytes rule.** Detokenisation must give the same value every time for a token. To confirm with the tokenisation team.
6. **Reporting.** If a daily report reads this table, add `amount`, `currency` and a second index on business date. If it reads a stream or the data lake, add nothing.
7. **Lookup by PayX payment id.** If the status path needs it, add an index on `paymentId`. Not needed if every caller has the UETR.
8. **Item size.** The limit is 400 KB. One wire with both payloads is a few KB. Remittance-heavy wires should be checked once real samples exist.
9. **Replication across regions**, and with which consistency. To confirm with the platform.
