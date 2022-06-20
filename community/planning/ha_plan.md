----
"mermaid = true"
----

<!--
  Copyright 2022 Cargill Incorporated
  Licensed under Creative Commons Attribution 4.0 International License
  https://creativecommons.org/licenses/by/4.0/
-->

# Grid High-availability Plan

* Context - what are we trying to do and why? Multiple instances, redundancy
  and availability
* How are we going to do this? Database coordination and claims
*   Why the database?
*   Claims and expiry
*   Two-phase expiry (fast path/slow path)
* Delay and fail paths








### PASS/DELAY/FAIL

* Each event represents an interaction with an outside component that changes
the outside component's state.
* Consider DELAY and FAIL between each event
* DELAY means that the claim expires but then work continues at an unknown
  point in time - outside components don't know if this one is delayed or has
  failed


```
--- request claim on service_id from DATABASE
 |
 |  get claim on service_id
 |
 |  get batch from database
 |
 |  queue batch
 |
-+- begin batch submission & notify OBSERVER
 |
-+- begin post request to DLT
 *
-+- receive http response from DLT
 |
-+- update batch status via OBSERVER
 |
-+- request renewal on service_id
```

### Why use a claim/expiry mechanism?

Really, why do we need instances to "claim" `service_id`s anyway?? Why???

In the situation where we have a lot of services, claims allow instances to
narrow the scope of work they need to do. They don't need to queue all of the
batches, just the ones for which they have a claim.

An alternative would be a batch-based claim, but that would require instances
to work on the entire list of pending batches, resulting in duplicate work and
excessive memory use.

Another alternative would be coordination amongst instances, but that would
be significantly more complex to implement and have trivial additional benefit.

We must be able to invalidate a claim in case instances fail -> Failed
instances can't invalidate their own claims; I don't think we want a
centralized down-detection mechanism; intances can't check the health of other
instances -> therefore, the claim must be time-based.

Claims need an expiry, because instances may fail or hang. Without a way to
invalidate claims, the ability to work on a `service_id` would die with the
instance that claimed it.

How many should an instance try to claim? -> All of them; it doesn't know if
it's the only instance out there. Instances are ignorant of one another.

### Do we need two-phase expiry?

First phase: If there are no batches mid-submission, you can grab this
service_id. If there are batches mid-submission, leave it alone.
Second phase: Up for grabs.

#### What does this accomplish?

It allows instances that are waiting for a slow submission to hold onto a
claim. If an instance isn't mid-submission and hasn't renewed the claim,
that's a good indicator something is wrong with it. It's sort of a fast path/
slow path for determining if there's something wrong with the instance.

We expect a healthy service to be actively submitting batches and/or
maintaining an active claim on a service_id. If it's doing neither, there's
something wrong, and another instance should take its work.

If a service appears to be submitting batches but not maintaining an active
claim, we don't know if it is hung up or has failed. Past a certian amount of
time (based on the maximum time it can spend submitting a batch), it's probable
that the instance is hung or has failed, and another instance should take its
work.

#### What's the tradeoff?

The tradeoff largely comes down to the timing of when an instance is considered
to have failed. This would be configurable, so the tradeoff is flexible.

graph TD
    A[Within claim window?] --> |yes| B[Probably HEALTHY]
    A --> |no| C[In which expiry window?]
    C -->|first| D[Appear to be submitting?]
    D -->|yes| E[Assume HEALTHY]
    D -->|no| F[Probably FAILED]
    C -->|second| G[Probably FAILED]

[![](https://mermaid.ink/img/pako:eNpl0M1OwzAMB_BXsXzeXqAHUFk7NmkHJCYh1O6QNt5q0SRVPlSqZe9Ou24gwCcffvbf8hlrIwkTPFnRNbDPSg1jpcUb-4Y11K1gBT1rafrHAyyXDxAHchGeihdrKlG1A2zydLffvB9uozPSJsKq2GroG64boM-O7fC9aKaricYjW-cjZEXadSQseAMVgQuVYu9Zn-46u-preF6kzgVFf6JnMSWvf65bp9tdnv1KdFQbLSM8_1O4QEVWCZbjS87TTIm-IUUlJmMrhf0osdSX0YVOCk-5ZG8sJkfROlqgCN68DrrGxNtAd5SxGN-rburyBSeEeTw)](https://mermaid.live/edit#pako:eNpl0M1OwzAMB_BXsXzeXqAHUFk7NmkHJCYh1O6QNt5q0SRVPlSqZe9Ou24gwCcffvbf8hlrIwkTPFnRNbDPSg1jpcUb-4Y11K1gBT1rafrHAyyXDxAHchGeihdrKlG1A2zydLffvB9uozPSJsKq2GroG64boM-O7fC9aKaricYjW-cjZEXadSQseAMVgQuVYu9Zn-46u-preF6kzgVFf6JnMSWvf65bp9tdnv1KdFQbLSM8_1O4QEVWCZbjS87TTIm-IUUlJmMrhf0osdSX0YVOCk-5ZG8sJkfROlqgCN68DrrGxNtAd5SxGN-rburyBSeEeTw)

#### What is actually going to manage this claim expiry?

What is going to go through the database and use this decision tree?

Probably the instance, specifically the queuer.
Does it do it as part of the claim process - this could make the claim process
slow and increase the odds of collision.
Does it run a maintenance activity before the claim process? We might be able
to make this lock-free... and the claim process would stay simple and fast.
Or if it's on the serializable isolation level and it fails, that means another
instance cleaned it up and this instance doesn't need to.

So the queuer would:

1. Perform maintenance activity (clean up expired `service_id` claims)
2. Renew existing claims
3. Attempt new claims
4. Get batches of claimed `service_id`s
5. Queue batches

Would any of these queuries conflict with one another (assuming different
instances are running them)?
1 & 2: How does an instance know that its claims are still valid?? What if it
was delayed and it thinks it still has a claim but it was cleaned up? Maybe
check and update the claims in one transaction? I.e. rely on the database for
what claims the instance has rather than its internal state. 1 and 2 should be
serializable.
1 & 3: These are looking for different entries, so they won't overlap. 3 will
see the work of 1 once the results are committed.
1 & 4: 1 reads data from the same table as 4 to determine expiry. 4 is read
only, so this should be ok.
2 & 3: These would look for different entries, so they won't overlap. 3 will
exclude results from 2 in its query.
2 & 4: Work on different tables, so ok.
3 & 4: Work on different tables, so ok.

NOT SURE ABOUT 4: It could use the service_id_claims table instead of instance
state!!!

It appears all queries that work on the service_id_claims table must be
serializable, but queries on the main batches table don't need to be. This is
good, because the service_id_claims table should be small; the cost of the
serializable isolation level should be minimal and serialization failures
relatively less common. The batches table, which is much bigger, won't need
serializable and won't incur that cost.

Maybe we can set that table's default transaction level to serializable?
PROBABLY NOT because of 4



## Paths

__request claim on service_id from DATABASE__

DELAY consequence:

1. Claim exprires and another instance submits batch (fast path)
2. This instance then submits batch for a second time
3. The first batch submission is confirmed (other results don't matter to in
  this scenario)
4. Another instance submits the next batch for this service_id
?? Would we poll for the resubmission if the first came back ok?

FAIL consequence: claim expires and another instance submits batch (fast path)

__begin batch submission & notify OBSERVER__

DELAY consequence:

FAIL consequence: claim expires and another instance submits batch (slow path)
Is this batch "unknown" if the claim expires? It could have been submitted
What happens if it is not submitted and marked unknown? -> It gets polled for
(time unknown), then requeued by some new instance. Meanwhile, the service_id
is blocked until it's successfully submitted. This would probably take quite a
while.
What happens if it is not marked unknown?

__begin POST request to DLT__

DELAY consequence:

FAIL consequence: Same as step above

__receive http response from DLT__

DELAY consequence:

FAIL consequence: Same as step above

__update batch status via OBSERVER__

DELAY consequence:

FAIL consequence: claim expires and another instance gets claim (fast path)

__request claim on service_id from DATABASE__

DELAY consequence:

FAIL consequence: claim expires and another instance get claim (fast path)
















## Instance Actions

Template table

||Resources|Processes|
|---|---|---|
|__Individual__|Private, instance state|Each instance must complete (blocking)|
|__Shared__|Public, centralized state|Some instance must complete (try)|

Centralized state must be visited to be safe. Can be copied, but work on copied
state is unsafe.

### Queuer - Claim maintenance

```sql
WITH questionable_instance_statuses (cliamant, batches_in_submission) AS (
  -- Get instances that appear to be in process of submitting batches but
  -- have passed the first expiry
  SELECT
    s.claimant AS claimant,
    COUNT(b.batch_id) AS batches_in_submission
  FROM service_id_claims s
  LEFT JOIN batches b
  ON service_id
  LEFT JOIN batch_statuses bs
    ON service_id, batch_id
  -- In process of being submitted?
  WHERE b.submitted
  -- Exclude already successfully submitted batches
  AND (
    bs.dlt_status IS NULL -- First time being submitted
    OR bs.dlt_status = 'UNKNOWN') -- Submitted previously but with an issue
  -- Past the first expiry
  AND EXTRACT(EPOCH FROM now()) - EXTRACT(EPOCH FROM s.expiry_1) > 0
  GROUP BY claimant
  ),
-- Void claims of instances where ...
UPDATE service_id_claims
SET claimant = NULL
-- Instance has been inactive for an unreasonable amount of time
WHERE EXTRACT(EPOCH FROM now()) - EXTRACT(EPOCH FROM expiry_2) > 0
-- Or instance is no longer either actively submitting or managing its claims
OR claimant IN (
  SELECT claimant
  FROM questionable_instance_statuses
  WHERE batches_in_submission = 0
)
```

### Queuer - Renew claims

```sql
UPDATE service_id_claims
SET
  expiry_1 = (now() + INTERVAL '30 seconds'),
  expiry_2 = (now() + INTERVAL '3 minutes')
WHERE claimant = {instance_id};
```

### Queuer - Get new claims

```sql
UPDATE service_id_claims
SET
  claimant = {instance_id},
  expiry_1 = (now() + INTERVAL '30 seconds'),
  expiry_2 = (now() + INTERVAL '3 minutes')
WHERE claimant IS NULL;
```

### Queuer - Get batches of claims
```sql
SELECT FOR UPDATE 
  s.service_id,
  b.batch_id,
  b.submitted,
  b.created_at,
  bs.dlt_status AS status,
  bs.updated_at AS last_status_update
FROM service_id_claims s
LEFT JOIN batches b
ON service_id, batch_id
LEFT JOIN batch_statuses bs
ON service_id, batch_id
WHERE bs.dlt_status NOT IN (
  'PENDING', 'INVALID', 'VALID', 'COMMITTED'
)
```

## NEW QUERIES START HERE

### Queuer - Renew claims

```sql
UPDATE service_id_claims
SET
  expiry = (now() + INTERVAL '30 seconds'),
WHERE claimant = {instance_id};
```

### Queuer - Get new and expired claims

```sql
UPDATE service_id_claims
SET
  claimant = {instance_id},
  expiry = (now() + INTERVAL '30 seconds'),
WHERE claimant IS NULL
OR EXTRACT(EPOCH FROM now()) - EXTRACT(EPOCH FROM expiry) > 0;
```

### Queuer - Get batches of claims

This might incorporate the queuing logic.
If the queuer gives the submitter a `batch_id` instead of a batch, this query
would not include `b.serialized_batch`.

```sql
SELECT FOR UPDATE 
  s.service_id,
  b.batch_id,
  b.submitted,
  b.created_at,
  b.serialized_batch,
  bs.dlt_status AS status,
  bs.updated_at AS last_status_update
FROM service_id_claims s
LEFT JOIN batches b
ON service_id, batch_id
LEFT JOIN batch_statuses bs
ON service_id, batch_id
WHERE s.claimant == {instance_id}
AND bs.dlt_status NOT IN (
  'PENDING', 'INVALID', 'VALID', 'COMMITTED'
)
```
