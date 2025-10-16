# Persistent Replica Runtime

The persistent replica runtime is a deterministic incremental computation
environment.

Background: https://github.com/the-draupnir-project/Draupnir/issues/979

The matrix-protection-suite has evolved to around a "revision issuers" concept
to provide deterministic, incremental computation and data-flow in order to
manage complex interactions and resource intensive operations.

This abstraction has not been generalized and is ad-hoc. And the
matrix-protection-suite had specific and constrained design goals in its early
development in order to build a new core for the draupnir project:

- Revision issuers are entirely in-memory structures. There are no asynchronous
  methods to provide data for persistent storage.
- Attempts to persist specific revision issuers has required for backing stores
  to be written that are in reality a downstream consumer of the deltas produced
  by revisions outside of startup time.
- The backing stores are not generalized either and have virtually no support
  for rollbacks in failed updates etc.
- Transactions between revision issuers are not formalized and it is possible
  for revision issuers to become unsynchronised, if a delta fails to process or
  is dropped, that effect is permanent until the system is restarted.
- Further the lack of "transaction" complicates persistence of revisions issuers
  that are not sources of non-deterministic input e.g. the PolicyRoomRevisions
  were persisted, we'd want to be confident that all deltas from the source
  revision (room state) were applied in order.

Most importantly, the matrix-protection-suite was designed to be used entirely
within one JS process. Using workers was never a consideration. This holds the
Draupnir project back from being able to use more centralized service
architectures. Which would be required to provide a popular service.

## Proposal

We propose that all computation in Draupnir to be expressed through _replica
actors_ using a new framework called the persistent replica runtime. This
transition would happen incrementally, rather than as a focussed effort like the
transition to the matrix-protection-suite and Draupnir 2.0.

This would give us the following advantages:

- Restart can be immediate and instantaneous.
- Workload can be balanced across worker threads using actor migration.
- All core computation is deterministic and reproducible, easing development and
  testing.
- All data-flow interactions are explicit and auditable.
- Effects and effect handlers are first-class, providing not only pre-viewable
  and auditable consequences for protections, but for all side-effects.
- The system is fudamentally fault tolerant: all computation is incremental,
  transactional, and deterministic. And can be restarted at the last delta.

_Revision issuers_ become _persistent replicas_. The live in-memory instance of
a _revision issuer_ becomes a _replica actor_. A meta-object is introduced to
describe the schema of the revisions and the logic of the actor called a
_replica description_.

Persistence is implemented by a storage backend that takes the _replica
description_ and creates tables or documents from them.

A scheduler is introduced that schedules _replica actors_ across worker threads
in Node.JS.

## Concepts

### Record

The first primitive of the runtime is records. These are synonymous with
documents from a document store or a database record. Examples include Matrix
events, policies, room membership.

Records are a concept because they provide us with automatic persistence
management when the schema provided to a replica description alongside the delta
schema.

### Revision

revisions are immutable snapshots that accurately reflect some state at the
point the revision was created. These are composed of a collection of records.

Revisions are produced by replica actors, and both are described by a replica
description.

### Delta

Deltas represent the outcome of a computation related to a revision that provide
a reproducible transformation of the revision state.

Delta's have their own identity, a delta ID and backlink to a previous delta.

#### Provenance data

Each delta needs to include information about upstream inputs it depends on:

- A revision ID for the revision used to derive the delta.
- The revision ID's of any revisions that are transitive dependencies, along
  with the identities of the replicas that issued them.

### Revision reducer

A revision reducer is stateless code that takes a revision and an external delta
or data input and produces a new delta specific to the revision.

### Replica

A replica is the storage layer representation of a specific partitioned
revision, its records, and its deltas.

These are ensured to exist by the replica description.

### Replica actor

A replica actor is responsible for handling input data, calling revision
reducers, and persisting new revisions and deltas.

A revision actor is produced by a replica description and is instantiated with
partition data.

### Replica description

A replica description describes how to build replicas from the partition data,
and also how to build revision actors.

### Rebuild delta

A rebuild delta is a specific kind of delta that is the direct result of a
change in implementation of a replica. For instance it may be important to issue
a rebuild delta when bugs are fixed in replica descriptions that meant replica's
were producing faulty data.

### Partition

replicas are partitioned by a key. This key is usually something like a Matrix
room ID, or the user identifier of a Draupnir instance.

### Actor ID

These prevent multiple live in-memory actors providing for the same replica. The
scheduler makes sure that there is only ever one live actor per replica
partition. This does require the scheduler to rebuild all actors if the
scheduler crashes, but i'm fine with that.

### Revision ID

A revision identifier is a specific ULID that is associated with a revision.

### Intent replica (effect)

A intent replica is a special kind of replica that produces an intention to see
some outcome enacted. These intents are then carried out by outcome issuers
(effect handler).

Enabled protections should be intent replicas. These protections are partitioned
by the capability provider set they are using and the draupnir they are
associated with. They are not associated with the in-memory instance but the
activation of the protection.

### Outcome replica (effect handler)

An outcome replica is a special kind of replica that records whether actions
have been issued for intents. The semantic meaning of the outcome is not
relevant, e.g. failure to ban a user is still an outcome that needs to be
recorded and is the outcome of the intent.

outcome replicas for protections are partitioned in the same way as the intent
replicas, it's just the capability providers are responsible for producing the
actions.

## Semantics

### Actor migration

Actors need to be transportable across node.js worker threads in order to
balance CPU.

### Preemption

replica actor client code will be interruptive in order to gracefully handle
shutdown and migration of actors.

This will be achieved by abusing I/O operations to signal interrupts. CPU bound
actor code will still have to cooperatively check for an interrupt signal.

### Garbage collection

Replica dependents will formally be tracked. Transitive dependencies will also
need to be considered for rollbacks to be successful. replicas will only be able
to discard an old revision when all transitive dependents have successfully
migrated to a superseding revision.

### Scheduling

The scheduler is responsible for starting replica actors and migrating them when
CPU becomes exhausted in worker threads. The scheduler gives each actor a
controller that signals interrupts. The store is expected to also receive the
controller so that it can preempt on I/O.

The scheduler will have to be communicated `os.loadavg()` from each worker
thread and use this to trigger actor migration. Realistically we can probably
only move actors from high utilisation to low utilisation and have some strategy
that tries to identify and isolate the cpu intensive actors for reporting.

### Crashing

What happens when an actor crashes?

#### Poisoned delta

If the revision reducer crashes when processing a delta, and continues to crash
when re-attempted, the delta will be skipped with an explicit acknowledgement.
This signifies a bug in the implementation of the revision reducer and is really
a critical issue. It's important not to bring down the whole system, but quite
honestly the existence of a reducer bug in critical areas can be as bad as that.
Protections obviously can have faulty reducers and don't effect other
functionality.

#### Other issues

The actor should be restarted with exponential backoff.

### Messaging

How do deltas get communicated to dependencies? especially if the in-memory
actors can crash and be unreliable for message delivery.

The deltas get written to the persistent store and a message is sent to the
other threads about the write so that the delta can be read back from the store
and propagated to dependencies.

## Non-deterministic inputs

The PRR needs non-deterministic inputs in order to do anything useful, such as
matrix events, commands, and time scheduling.

These are provided by _source replicas_.

### Source replica

A source replica is a system or client provided replica with no dependencies
that produces deltas. Source replicas provide no data in their records, only the
deltas are significant.

### Time source replica

Some protections in Draupnir have time-scheduled background tasks, such as
DraupnirNews.

These should be implemented by having a revision reducer that accepts a tick
from a time service.

The time source replica is a system provided source replica that persists
records about when to send ticks to other replicas. Each replica that requires
time scheduling is given a adaptor replica to bridge them to the time source. So
that they are not informed about every single tick.
