# Event sourcing and streaming: the pattern

> Vendored reference, pattern only. It describes six mechanisms proven in an
> earlier Go system (Postgres log of record, Watermill transport, optional NATS
> JetStream). It is **input, not a decision**: which omap-stack state is
> event-sourced, and on which of these mechanisms, is decided on the wayfinder
> map (ENG-322) and recorded in `docs/adr/`. Mechanics defer to this repo's
> conventions (`AGENTS.md`); event semantics follow this document.
>
> **omap-stack is Postgres only.** Where this document relies on NATS JetStream
> (broker dedup via `Nats-Msg-Id`, `MaxAckPending(1)` ordering, generated broker
> grants in section 5), it does not apply here. Consumers read the log from
> Postgres in id order with a stored cursor, and the catalog conformance test is
> kept without broker enforcement.
>
> Companion: `docs/patterns/go-architecture-guidelines.md`.

---

## 0. TL;DR for the impatient

The pattern is **six mechanisms that only work together**:

| # | Mechanism | One-line invariant it buys |
|---|-----------|---------------------------|
| 1 | **Per-module append-only `events` table** | The log of record is Postgres, not the broker. |
| 2 | **`Record(ctx, tx, …)` takes the caller's tx** | "Publish outside the transaction" is *unwritable*. |
| 3 | **Relay** (LISTEN/NOTIFY + poll + advisory lock) | Committed rows reach the bus, in order, exactly once per crash-window. |
| 4 | **`processed_events` claim-insert** | Consumer idempotency where the *unique violation is the check*. |
| 5 | **Typed catalog (`Def[T]`) + conformance test** | An event's shape and its authorization can't drift. |
| 6 | **Broker-enforced grants** (generated NATS conf) | Default-deny; a "firewall" subject is provable at test time. |

Take fewer than all six and you get a system that looks event-driven and
silently loses events.

---

## 2. The write path, mechanism by mechanism

### 2.1 The log table (one per module schema)

```sql
CREATE TABLE events (
  id              uuid        PRIMARY KEY,   -- UUIDv7, generated in Go
  event_type      text        NOT NULL,      -- "lineup_locked", past tense
  payload_version int         NOT NULL,      -- schema evolution
  payload         jsonb       NOT NULL,      -- references, never aggregates
  subject_key     text        NOT NULL,      -- the aggregate this is about
  correlation_id  text        NOT NULL,      -- causal chain across modules
  recorded_at     timestamptz NOT NULL DEFAULT now(),
  published_at    timestamptz                -- NULL = relay hasn't sent it
);

CREATE INDEX events_unpublished_idx ON events (id) WHERE published_at IS NULL;

-- Append-only, enforced by the grant system rather than by convention.
REVOKE UPDATE, DELETE ON events FROM <module_role>;
GRANT  UPDATE (published_at) ON events TO <module_role>;
```

Four decisions worth stealing verbatim:

- **UUIDv7 generated in Go, not `DEFAULT uuidv7()`.** The relay needs the id
  *before* the insert returns, because the id doubles as the broker dedup key.
  It also makes `ORDER BY id` identical to `ORDER BY recorded_at`, which is why
  the partial index above *is* the ordering guarantee, not just an optimisation.
- **Column-level grants make append-only real.** `REVOKE UPDATE, DELETE` +
  `GRANT UPDATE (published_at)` means the module's own role physically cannot
  rewrite history, and the relay still marks progress. No second table, no
  trigger, no discipline required.
- **`subject_key` is the ordering domain.** Not the module, not the topic, the
  aggregate. "Ordering guarantees are per subject key."
- **Payloads carry references, never embedded aggregates and never large
  artifacts.** the source system's `RunnerGeneratedPayload` deliberately carries a name but
  *not* a rating, so a consumer wanting the rating "has to go and design the
  lossy projection properly rather than reaching for a number that happened to
  be in the payload." This is the single highest-leverage discipline in the
  whole pattern: it is what stops events becoming a distributed shared database.

### 2.2 `Record`: the API shape that makes misuse impossible

```go
func Record(ctx context.Context, tx pgx.Tx, schema string, e Event) (uuid.UUID, error)
```

Note what is *absent*: there is no `Publisher` to obtain, no `Flush`, no
post-commit hook. It takes the caller's `pgx.Tx`. Atomicity is then "whatever
the caller's commit does", nothing to get wrong.

Inside, after validation and `json.Marshal`, it does two writes:

```go
INSERT INTO <schema>.events (id, event_type, payload_version, payload,
                             subject_key, correlation_id) VALUES (…)
NOTIFY "the source system_events_<schema>"
```

`NOTIFY` inside the transaction is **itself transactional**: delivered on
commit, dropped on rollback. A caller that rolls back has announced nothing -
without any code path checking for it.


### 2.3 Correlation IDs

`ensureCorrelationID(ctx)` inherits the ambient correlation id or mints a
UUIDv7. Every recorded event is correlated, inherited when the causing work was
correlated, fresh when it wasn't. UUIDv7 again, because "sorting a log by
correlation id groups causal chains in the order they began."

Watermill's `middleware.CorrelationID` propagates it across the bus, so one
originating HTTP request is greppable through every downstream module.

### 2.4 The relay: the half that turns rows into messages

One relay per module (about 250 lines in the source system). Its loop:

```
acquire pooled conn → pg_try_advisory_lock(module) → LISTEN channel
loop:
    drain()                         # SELECT … WHERE published_at IS NULL ORDER BY id LIMIT 256
    if published == batchSize: continue      # more waiting, go straight back
    waitForWork()                   # WaitForNotification OR poll timeout (2s)
```

Five load-bearing details:

1. **Advisory lock for the whole life of the relay.** Two processes cannot
   publish the same module's events concurrently and interleave them out of
   order. This is what keeps "single-instance" honest across a deploy overlap
   instead of silently double-publishing.
2. **NOTIFY is latency; the poll is correctness.** A `NOTIFY` delivered while
   nobody was listening is simply gone. The 2s poll guarantees progress after a
   crash, a bus outage, or a restart mid-backlog. Do not drop the poll.
3. **Publish first, mark second.** A crash in between republishes on restart and
   the broker dedups on message id. Marking first would *lose* the event. The
   asymmetry is deliberate: at-least-once by construction.
4. **Never skip a failing event.** `drain` returns the index it failed at; the
   failing row stays unpublished and is retried. Publishing the events behind it
   would break ordering for the whole subject.
5. **Exponential backoff, 100ms → 30s**, reset on any successful drain.

Message metadata set on publish (this is the wire contract):

```go
msg.Metadata.Set(MetadataEventID,       e.ID.String())
msg.Metadata.Set(MetadataEventType,     e.Type)
msg.Metadata.Set(MetadataVersion,       strconv.Itoa(e.Version))
msg.Metadata.Set(MetadataSubjectKey,    e.SubjectKey)
msg.Metadata.Set(MetadataModule,        module)
msg.Metadata.Set(MetadataCorrelationID, e.CorrelationID)
msg.Metadata.Set(nats.MsgIdHdr,         e.ID.String())   // ← broker dedup key
```

That last line is why the source system sets Watermill's `TrackMsgID: false`: the option
would derive the dedup key from the *watermill* UUID and overwrite the header.
The key must be the event id, because the event id is the event's identity.

---

## 3. The read path

### 3.1 Replay from Postgres, never from the broker

```go
func ReadAll(ctx, q Querier, schema string) ([]Recorded, error)
func ReadAfter(ctx, q Querier, schema string, after uuid.UUID) ([]Recorded, error)
```

`Querier` is satisfied by both `*pgxpool.Pool` and `pgx.Tx`, so a rebuild reads
outside a transaction and a test reads inside one. `ReadAfter` works because
UUIDv7 sorts by creation: "after" is both id order and time order, so a
projection stores one cursor (`last_event_id`) and resumes.

**This is the function that makes it event sourcing.** JetStream/watermill-sql
is transport with a retention limit; the log of record is here and outlives it.

### 3.2 Consumer idempotency: the claim-insert

```go
func Idempotent(pool *pgxpool.Pool, schema, consumerName string, h Handler) message.NoPublishHandlerFunc {
    // 1. decode Recorded from message metadata
    // 2. BEGIN
    // 3. INSERT INTO <schema>.processed_events (consumer, event_id) VALUES ($1, $2)
    //       → unique violation? already handled: return nil (ack, no effect)
    // 4. h(ctx, tx, e)          ← handler receives THE SAME tx
    // 5. COMMIT
}
```

```sql
CREATE TABLE processed_events (
  consumer   text        NOT NULL,
  event_id   uuid        NOT NULL,
  handled_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (consumer, event_id)
);
```

Three things here that are easy to get subtly wrong:

- **The insert IS the check.** Checking first and then handling lets two
  concurrent deliveries both see "not handled" and both apply effects. Claiming
  by insert makes the race a unique-constraint violation instead.
- **The handler gets the transaction.** The record of having handled the event
  commits *with* its effects. "The effects happened" and "we handled this event"
  cannot disagree. And a handler that records its own event gets all three in
  one commit, the outbox chains.
- **`consumer` is in the primary key.** Two consumers of one event are
  independent; neither suppresses the other. Corollary: **the consumer name is a
  production identifier.** Renaming it replays that consumer's entire history.


### 3.3 The router: what the Watermill dependency actually buys

```go
r.AddMiddleware(
    middleware.CorrelationID,  // first: everything downstream logs under it
    poison,                    // outside Retry: set aside only after retries spent
    retry.Middleware,          // 5 attempts, 100ms → 10s, ×2
    middleware.Recoverer,      // inside Retry: a panic becomes a retryable error
)
```

The comment in the source is worth reproducing because middleware order is the
kind of thing that is silently wrong for a year:

> *PoisonQueue outside Retry, so an event is only set aside after its retries
> are spent, not on the first failure. Recoverer inside Retry, so a panic becomes
> an error that retries like any other rather than killing the process.*

**Retries are bounded on purpose.** With `MaxAckPending(1)` (below), an event
that can never be handled would block everything behind it forever. Bounded
retries + dead-letter is the only safe combination.

### 3.4 Ordering, end to end

Per-subject order survives *handling*, not just delivery, via one option:

```go
nats.MaxAckPending(1)   // server won't deliver #2 until #1 is acked
SubscribersCount: 1     // more would defeat MaxAckPending(1)
```

The full chain: UUIDv7 keys sort by creation → one relay goroutine publishes in
id order under an advisory lock → broker preserves publish order per stream →
`MaxAckPending(1)` serialises handling. Break any link and ordering is folklore.

**The slow-handler trap** (learned the hard way, per the source system's comments):
watermill-nats has a *client-side* `AckWaitTimeout` (default 30s) distinct from
the broker's `AckWait`. When the client deadline passes it cancels the handler's
context, so a legitimately long handler is killed mid-flight with nothing
logged, while the broker sits on the unacked delivery for the full `AckWait`.
**The two deadlines must move together or the longer one is a lie.** Hence
`HandleSlow`/`ConsumeSlow`, which set both from one `ackWait` argument.

---

## 4. The catalog: the part most ports forget

`Def[T]` binds a subject to a payload type and a schema version:

```go
type Def[T any] struct {
    Module  string   // producing module: its schema, and the subject's middle token
    Type    string   // "lineup_locked", the subject's last token
    Version int
}

func (d Def[T]) Subject() string { return fmt.Sprintf("events.%s.%s", d.Module, d.Type) }
```

Each producing module declares its whole catalog in one package
(`internal/<module>/api/events/`), importable by anyone:

```go
package lifecycleevents

const module = "lifecycle"

type RunnerGeneratedPayload struct {
    RunnerID    uuid.UUID `json:"runner_id"`
    Name        string    `json:"name"`
    Nationality string    `json:"nationality"`
}

var RunnerGenerated = events.Def[RunnerGeneratedPayload]{
    Module: module, Type: "runner_generated", Version: 1}

// Defs is the module's catalog entry, for the conformance test.
var Defs = []events.Meta{RunnerGenerated.Meta(), /* … */}
```

Producing and consuming then go through the def, so **subscribing to an event
that does not exist is a compile error**:

```go
// produce, inside the caller's tx
events.Emit(ctx, tx, lifecycleevents.RunnerGenerated, runnerID.String(), payload)

// consume, typed handler; decoding is generated for you
events.Consume(router, lifecycleevents.RunnerGenerated, pool, schema,
    func(ctx context.Context, tx pgx.Tx, p RunnerGeneratedPayload, e events.Recorded) error {
        return s.store.UpsertListing(ctx, tx, Listing{RunnerID: p.RunnerID, Name: p.Name})
    })
```

Consumer names follow one convention, `<consuming module>_<event type>`, which
keys the idempotency records *and* names the durable consumer.

**Versioning is deliberately permissive on the read side.** Consumers are not
filtered by version: an older row decodes into the same Go type with missing
fields zero. Additive change = bump `Version`, keep one type. That is the whole
evolution story, and it is enough as long as payloads stay reference-only.


## 5. Authorization: grants as a firewall, tested

One table (`modules.go` in the source system) declaring every module and what it
may publish/subscribe to on the bus:

```go
type Module struct {
    Name       string
    Persistent bool       // owns a Postgres schema and role
    Publish    []string   // explicit enumeration: never "events.>"
    Subscribe  []string
}
```

From that single table, three things are derived:

1. **The NATS server config is generated** (`//go:generate`). The *broker*, not application code, refuses an
   unauthorized subscribe. JetStream API grants are scoped to the module's own
   stream name: not the whole `$JS.API` space, which would let a module delete
   another's stream and route around subject permissions entirely.
2. **Startup iterates it** : migrate → pool → connect → ensure
   stream → relays → module runners, each step a precondition for the next.
3. **Tests assert against it**, including `TestFirewall`, which asserts a
   named subject appears in *exactly one* module's Subscribe list.

Wildcards are banned in Subscribe lists on purpose: "a wildcard would grant
every future subject to whoever holds it and quietly break the default-deny
property the firewall depends on."

### The conformance test: catalog ↔ grants

A `catalog_test.go` holds the two sources of truth together. Three assertions:

- `TestEveryDefIsPublishable`, every typed def has a Publish grant. Without it,
  the broker refuses the event on first emit, in production, on a Sunday.
- `TestNoDefIsDeclaredTwice`: two defs on one subject = two payload shapes
  claiming one wire contract.
- `TestEverySubscribeGrantHasADef`, drift in the other direction: a stale or
  typo'd grant, or a def missing from its `Defs` slice.

This test is ~110 lines and is the single highest value-per-line artifact in the
pattern. **Port it early, before there is drift to clean up.**


## 6. Testability: the memory bus

An in-memory bus gives the same two interfaces (`message.Publisher`,
`SubscriberSource`) over `gochannel.GoChannel`, **enforcing the same grants**:

```go
func (m *ModuleBus) SubscriberFor(string, time.Duration) (message.Subscriber, error)
var _ SubscriberSource = (*ModuleBus)(nil)   // compile-time proof it slots in
```

Two config choices carry the whole fidelity claim:

- `Persistent: true`, a subscriber arriving after a publish still receives
  everything, matching JetStream's `DeliverAll`.
- `BlockPublishUntilSubscriberAck: true`, **the only mode in which gochannel
  preserves order.** Without it each message gets its own goroutine and twenty
  sequential publishes race onto the subscriber's channel. A blocked publish is
  a relay waiting on a consumer, exactly as a full ack window would make it wait.

And critically, the grant check is *duplicated* into the test rather than
exported from production, "so the test cannot accidentally loosen what
production checks."

Because the router cannot tell the difference, an in-process harness run means
the shipped consumers work.


## 7. Anti-patterns the source system explicitly guards against

Worth stating because each is a real bug someone shipped once:

- **Fat payloads.** Events carry references. A payload that embeds an aggregate
  turns the bus into a distributed shared database and every consumer into a
  coupled reader.
- **Wildcard subscribe grants.** Silently grants every future subject.
- **Checking-then-handling for idempotency.** Loses the race. Claim by insert.
- **Marking published before publishing.** Loses the event outright.
- **Skipping a failing event to make progress.** Breaks ordering for everything
  behind it on that subject.
- **Unbounded retries** with a serialised consumer. One poison message stops the
  world.
- **Renaming a consumer casually.** Replays its entire history.
- **Auto-provisioning streams.** Wrong names, no retention control.
- **Deriving the dedup key from the transport's message UUID.** A republish
  after a crash then fails to dedup, because the transport UUID is fresh.
- **A GC that can see the log of record.** It silently destroys the history being kept.
- **Degrading quietly when the bus is unavailable.** the source system refuses to start:
  "a half-started event backbone is worse than a process that refuses to run: it
  silently stops announcing state changes that did commit."
