+++
title = "Finding multi-record transaction bugs with Antithesis and TLA+ and LLM"
date = "2026-08-10"

[taxonomies]
tags=["aerospike", "distributed-systems", "transactions", "database", "tla", "formal-verification", "chaos-testing", "antithesis"]
+++

**Disclaimer:** This post is written by human, edited and formatted by AI.

# Finding multi-record transaction bugs with Antithesis, TLA+ and LLM

When a distributed database offers **multi-record transactions**, atomic updates across keys that may live on different nodes; the failure modes stop looking like “lost write on one row.” 
They look like **conservation laws breaking**: money appears from nowhere, or vanishes, while every individual RPC returned something plausible.

This post describes how we stress-tested that class of system with [Antithesis](https://antithesis.com/), what we learned when it found real bugs, why reproducing those bugs outside the fuzzer was brutal, and how **TLA+** (and LLM)
helped us reason about gaps before and after we fixed what we could on the client side.

Background on Aerospike’s public architecture: shared-nothing clustering, partition-based sharding, strong consistency is in the VLDB papers 
[*Aerospike: Architecture of a Real-Time Operational DBMS*](https://www.vldb.org/pvldb/vol9/p1389-srinivasan.pdf) and 
[*Techniques and Efficiencies from Building a Real-Time DBMS*](https://www.vldb.org/pvldb/vol16/p3676-srinivasan.pdf). 
The 2023 paper explicitly calls out **global distributed transaction systems** (instant interbank settlement is the motivating example) as a workload where consistency and availability constraints are unforgiving.

## Test case model in Antithesis

We built an **Antithesis workload** around a deliberately simple domain: a **parallel bank**.

- A fixed set of customer accounts holds integer balances.
- Many client processes issue **random transfers** between accounts concurrently.
- Each transfer is implemented as a **multi-record transaction**: debit one account, credit another, then **commit**.
- A global assertion checks **currency conservation** — the sum of all balances must stay constant.

That assertion is the canary. Single-record tests might miss subtle tears; a ledger invariant fails loudly when any participant applies the wrong half of a transfer.

The environment exercises realistic distributed stress:

- A **multi-node Aerospike cluster** with rack-aware placement (as described in the public strong-consistency material).
- **Network partitions, node restarts, and clock-adjacent timing** injected by Antithesis’s deterministic hypervisor.
- **Multiple client languages** driving the same workload (so driver semantics matter, not just server logic).
- **Parallel clients** hammering the cluster for long windows.

## Findings from tests

Antithesis found **currency violations** : runs where the invariant checker reported the wrong total balance even though individual operations often looked fine in isolation.
This is essentially saying that Transactions not honoring ACID properties.

Triaging those failures surfaced **several distinct bug classes**, not one root cause:

| Class | What it is                                                                                    | Typical symptom |
|-------|-----------------------------------------------------------------------------------------------|-----------------|
| **Partition divergence** | Different parts of the cluster disagree about whether a transaction is committing or aborting | One leg of a transfer finalizes forward while the sibling leg rolls back |
| **Client vs server timing** | The client gives up on commit while the server is still driving commit on the backend         | A one-sided ledger adjustment; total off by exactly one transfer amount |
| **Monitor / coordinator loss** | Metadata needed to finish all legs disappears or is incomplete                                | A leg never completes; totals drift |
| **Independent per-key completion** | Each key’s finalize step is a separate action with no atomic “all or nothing” barrier         | One account updates; the paired account does not |
| **Harness bugs** | Test code mishandles error objects (e.g., typos in exception handling)                        | Broken retry paths that mask or amplify real failures |

The most instructive failures were **not** simple “returned error to client” stories. They were **ordering and visibility** stories:

- A client observes **abandon** or **not in doubt** and proceeds to **rollback**, while the server’s coordination layer still treats the transaction as **committing**.
- After a **network partition**, one side’s view of commit intent can **diverge** from another’s; each side then applies **roll-forward on one key** and **roll-back on another** - perfectly locally consistent, globally wrong.
- The public literature already warns that split-brain and partition scenarios require careful merge semantics ([multi-site clustering](https://www.vldb.org/pvldb/vol16/p3676-srinivasan.pdf)).
Our failures were the micro-scale ledger analogue: two keys, one transaction, two different outcomes.

Antithesis gave us **witness logs**: a time-ordered narrative showing client abandon, later server commit activity, and the exact audit delta. That was invaluable indeed, and Antithesis definitively hit the same class of bug in every run. 
Still, it was extremely hard to pin-point the exact states and sequences when this weirdness surfaced. 

## Why it was hard to replicate

**Schedule sensitivity.** The bad interleaving often required a partition at a narrow window, a client timeout on commit, *and* a background coordination tick on the server, plus enough concurrent traffic to queue work in a particular shape. 
Change any variable and the bug vanishes for days.

**Observation bias.** Logs showed *what* happened (abandon at T₁, commit activity at T₂) but not a minimal causal diagram. Many subsystems touch a multi-record transaction: client retry policy, server-side coordination, per-partition masters, replication, and recovery after membership changes ([cluster management in the 2016 paper](https://www.vldb.org/pvldb/vol9/p1389-srinivasan.pdf)).

**No single “repro button.”** We could not boil most witnesses down to a five-line integration test that failed reliably in CI. 
Deterministic replay in Antithesis is powerful, but porting that to a developer laptop cluster, with different timing, hardware, and client versions—often failed to reproduce the same ledger tear.

**Blame diffusion.** A currency failure could be:

- server coordination applying opposite directions on sibling keys,
- client aborting too aggressively after an ambiguous commit,
- or test harness error handling breaking the in-doubt path.

I personally tried a few things to reproduce it locally: created a proxy server that
understood Aerospike wire protocol. And then I injected tailor made faults, partition failures, transaction retries and
so on. But, everytime, the currency was correct, ACID properties honored. No lost or gained money.

Without a separate reasoning tool, teams spin in circles arguing which layer “must have” been at fault.

That gap is what pushed us toward formal modeling.

## Simulation with TLA+

This is where LLM helped the most. LLMs are awesome in tracing through large codebases and identifying the state and flow.
Converting specific state transitions to TLA+ models was little bit of hand holding and lots of prompting.

We wrote a family of **bank models**, not the full Aerospike codebase, but distilled state machines capturing the commit protocol’s skeleton:

**Phase 1: independent legs.** After a transaction is marked committing, each account’s finalize is a **separate step**.
TLC can roll one account and stall the other forever. Invariant: *total balance unchanged.* **Violated.** This matches the structural risk of per-key roll messages with no cross-key guard.

**Phase 1.5: coordinator survival.** Add a boolean “coordinator alive” and adversarial actions: kill the coordinator mid-flight, or corrupt the list of participating keys. Invariant: *if totals are wrong, at least one corrective action remains enabled.* Sometimes **violated**; a permanent tear with no remaining fix in the model.

**Phase 2: client vs server commit intent.** Split variables: server believes committing; client believes it safely abandoned. 
Allow optional client-driven rollback while the server still commits. Invariants:

- *Naive implication:* client abandon ⇒ server not committing : **false** in reachable states.
- *Currency* : **false** after certain orderings.

This phase directly abstracted the Antithesis witness shape: abandon first, commit activity later, audit off by one transfer unit.

**Phase 3 : partition divergence.** Model two cluster “views” of the same transaction’s commit flag after a split. 
One side finalizes forward on account A; the other rolls back account B. Invariants:

- *Currency* : **false** (credit sticks, debit reverts).
- *No opposite finalize directions on paired legs* : **false**.

Each model is tiny—handfuls of variables—but TLC enumerates **all** interleavings allowed by the spec, not the ones you thought to try in a unit test.

We kept an explicit **assumptions box**: the TLA modules under-approximate some real behaviors (client-initiated rolls without the coordinator, 
detailed RPC loss, full replication lag). A TLC counterexample is a bug in the **spec relative to your intent**, which you then map carefully to code.

## How TLA helped find the gaps

TLA did three jobs logs alone could not:

**1. Separated bug classes.** Partition divergence (Phase 3) and client/server timing (Phase 2) produce similar **symptoms** (wrong total) but different **state shapes**. TLC traces labeled which adversary enabled the tear. That stopped us from “fixing” a client retry bug while the partition story remained.

**2. Made implicit assumptions explicit.** We had been acting as if:

> If the client is not *in doubt*, the server cannot still be committing.

TLC produced a **10-step counterexample** where both flags hold. The invariant failed in one click.

**3. Guided what *not* to over-fix.** When the model said “client rollback while server commits causes currency tear,” we knew a client-only mitigation 
had to **avoid aborting on ambiguous commit outcomes**; not merely retry once and give up. That narrowed the search in real code dramatically.

TLC did **not** replace Antithesis. It complemented it: fuzzing finds unbelievable schedules in real binaries; TLA explains **whether those schedules violate properties of a simplified protocol** and gives vocabulary for cross-team review.

## How we fixed it

Fixes landed at different layers:

### Client / harness (confirmed, shipped in tests)

The highest-confidence, client-side bug matched **Phase 2** directly. 
The workload called **abort** when **commit returned an ambiguous error**: timeout, abandoned mark, lost reply, even though the server might already be in the commit path. That is exactly the one-sided rollback the model flagged.

**Fix:**

- After a successful read/write phase, treat commit as a **retry loop**, not a single attempt.
- **Only abort** when commit fails for **verify/read-phase** reasons (the transaction’s preconditions failed).
- On **ambiguous** commit failures, **retry commit** without calling abort; if retries exhaust, **log and continue** without rolling back—accept temporary uncertainty rather than invent a one-sided ledger move.
- Classify typed commit outcomes (`Ok`, `AlreadyCommitted`, abandoned variants) explicitly instead of lumping errors together.

This change turned Antithesis green for the client-abandon class in our Rust parallel-bank workload. It aligns with the public strong-consistency discussion of **in-doubt writes** that must be resolved rather than guessed ([§4.3 in the 2023 paper](https://www.vldb.org/pvldb/vol16/p3676-srinivasan.pdf)): clients should not pretend uncertainty is certainty.

### Driver / test hygiene

Separate from protocol logic, we fixed harness bugs—exception handling typos that broke **in-doubt detection** on commit errors. Those are embarrassing but real; they can turn a recoverable retry into a crash or wrong branch.

## Lessons learnt

1. **Pick an invariant humans care about.** “Sum of balances constant” beats “no errors returned” for multi-record transfers.

2. **Fuzzing finds; formal methods explain.** Antithesis produced witnesses we could not stably replay locally. TLA+ produced **minimal** schedules and named the broken implication.

3. **Ambiguous commit is not failed commit.** In geo-distributed and partitioned deployments, the client’s view of commit completion can lag or diverge from the server’s. Aborting aggressively is a correctness bug, not a resilience feature.

4. **Model adversaries explicitly.** Kill the coordinator. Corrupt metadata. Split the cluster. Good specs read like attack scripts.
5. **Keep the assumptions box visible.** A TLC trace proves something about your spec. The mapping to production code is where engineering judgment lives.
6. **Leverage LLMs for TLA+.** LLMs are surprisingly good at this. Crawling through the code, and antithesis logs, and generating a model for the relevant path.

## References

- V. Srinivasan et al., [*Aerospike: Architecture of a Real-Time Operational DBMS*](https://www.vldb.org/pvldb/vol9/p1389-srinivasan.pdf), PVLDB 9(13), 2016 - clustering, partitioning, client interaction.
- V. Srinivasan et al., [*Techniques and Efficiencies from Building a Real-Time DBMS*](https://www.vldb.org/pvldb/vol16/p3676-srinivasan.pdf), PVLDB 16(12), 2023 - strong consistency, rack awareness, geo-distributed transactions.
- L. Lamport, [TLA+ and TLC](https://lamport.azurewebsites.net/tla/tla.html)
- [Antithesis documentation](https://antithesis.com/docs)
