+++
title =  "Hibernate, Flush Mode, and Broken Invoice Numbers"
date = "2025-12-30"

[taxonomies]
tags=["orm", "hibernate", "sql"]
+++

(This was something I debugged a decade ago, in Seller Platform Services at Flipkart. I don’t exactly remember the 
Hibernate version or detailed configurations, like isolation levels. The below is the best I could do with my memory.)

## The Problem

We discovered that invoice numbers were being generated out of sequence for shipments containing multiple orders.

Instead of clean sequences like:
~~~
inv-1, inv-2, inv-3
~~~

we occasionally saw results such as:

~~~
inv-1, inv-2, inv-125574
~~~

This only happened for shipments with multiple orders, and the pattern was inconsistent across environments, 
sometimes all invoices were wrong, sometimes only one, and sometimes none at all.

## How Invoice Numbers Were Generated

At the time, invoice numbers were generated using a custom sequence table and MySQL’s LAST_INSERT_ID() function.

The flow looked like this:

1. Increment the sequence:

```sql
update invoice_sequence
set next_value = LAST_INSERT_ID(next_value) + 1
where id = :id;
```

2. Fetch the generated value:

```sql
select LAST_INSERT_ID();
```


3. Insert the invoice number into `InvoiceNumber`
4. Insert an outbound message into another table (to notify other consumers of invoice event)

## The Key Assumption

This logic only works if **steps 1 and 2 execute atomically**, i.e., with no inserts happening in between. Otherwise, LAST_INSERT_ID() can change.

## What Went Wrong

### Hibernate Flush Behavior

For shipments with multiple orders, this logic ran **inside a loop**.

Hibernate (JPA), for performance reasons, does not immediately execute insert statements. Instead, it queues them and 
flushes later. Either:

* at transaction commit, or
* automatically before executing certain queries.

This led to a subtle but fatal interaction.

### Why LAST_INSERT_ID() Broke

In MySQL:

* If an insert happens on a table with an AUTO_INCREMENT column, it overwrites LAST_INSERT_ID().

* If the table has no auto-increment, LAST_INSERT_ID() becomes 0.

Hibernate sometimes flushed pending inserts between:

* the sequence update, and
* the `select LAST_INSERT_ID()` call.

When that happened, the invoice number suddenly came from an entirely different table.

## Observed Behavior in Different Environments

Because Hibernate’s default flush mode is AUTO, the timing was unpredictable.

We observed several patterns:

### a) Local Environment

For a shipment with 3 orders:
* The first invoice was correct
* The next two were wrong

Hibernate flushed inserts right before the next iteration’s `select LAST_INSERT_ID()`, corrupting the value.

### b) Production – Flush Only at Commit

Some shipments flushed only when the transaction closed.
No errors occurred

### c) Flush at the Last Order

Hibernate flushed inserts only during the final iteration.
Only the last invoice was corrupted: in 10 orders, only the last one was corrupt

### d) Flush Mid-Loop

Hibernate flushed somewhere in the middle. 
* One random order was corrupted
* Example: 4 orders, 2nd one corrupted

There was no fixed pattern.

## Why This Happened

Hibernate always flushes before executing a **native query**.

Since `select LAST_INSERT_ID()` is a native query, Hibernate triggered a flush—but only if there were pending inserts.

This behavior is documented and consistent with Hibernate’s implementation:

* Default flush mode: AUTO
* Meaning: Hibernate may flush “sometimes”

## Impact

* Shipments with multiple orders generated invalid invoice numbers
* Invoice sequences jumped unexpectedly
* Downstream systems were affected due to incorrect invoice references

## The Fix

We made two key changes:

### 1. Removed LAST_INSERT_ID() Entirely

* Replaced the update/select logic with a `SELECT … FOR UPDATE` on the sequence row
* Ensured correctness without relying on session-level side effects

### 2. Generate Invoice Numbers in One Shot
Instead of generating numbers inside a loop

* We queried and reserved all required invoice numbers once per shipment

* This eliminated flush-related timing issues completely.

## Why Tests Didn’t Catch It
* Unit tests used mocked databases
* Mock DBs don’t replicate:

    * `LAST_INSERT_ID()` semantics 
    * Hibernate flush timing

* The issue was non-deterministic, making it nearly impossible to catch with small test runs
* Integ tests missing multiple orders in one shipment

## Lessons Learned

* Never rely on LAST_INSERT_ID() across multiple statements unless they are truly atomic
* ORM flush behavior matters, especially with native queries
* Mocked DB tests are insufficient for sequence and concurrency bugs
* Prefer explicit locking (FOR UPDATE) or database-native sequences
* Generate identifiers outside loops whenever possible
* Run integration tests with all possible scenarios