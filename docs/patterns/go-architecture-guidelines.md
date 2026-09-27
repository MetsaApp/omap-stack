<!--
Vendored reference: a synthesis of Three Dots Labs (threedots.tech) guidance
for Go projects, kept in-repo so the standards survive link rot. Apply these
patterns proportionally to each package's complexity; the document itself says
when NOT to apply them.
-->

# Three Dots Labs Guidelines for Greenfield Go Projects

## TL;DR
- **Three Dots Labs (threedots.tech, by Robert Laszczak and Miłosz Smółka) advocates a pragmatic, idiomatic-Go blend of DDD Lite, CQRS, Clean/Hexagonal Architecture, and the Repository pattern**, proven across production systems and taught through their open-source Wild Workouts example and the free *Go With The Domain* e-book (60,000+ downloads). The core promise, in their words, is "the ability to maintain constant development speed. Without destroying or touching existing code too much."
- **The single most important rule is loose coupling through separated layers (domain, app, ports, adapters) and separate models per responsibility**: never a single struct carrying JSON + DB + validation tags. Keep business logic in the domain, define interfaces where they're consumed, and inject dependencies by hand.
- **Apply these patterns proportionally to complexity.** For simple CRUD/data-oriented services, skip them ("it's probably enough to put everything in one `main` package"); for complex business domains that will live for years, adopt them from the start. Start with a modular monolith, not microservices.

## Key Findings

1. **Philosophy: go slow to go fast, and quality is not traded against speed in the long run.** The patterns are an investment that keeps development velocity constant as code ages. They are grounded in the book *Accelerate* (loosely coupled architecture is among the strongest predictors of high-performing teams) rather than dogma.
2. **Four layers with strict dependency inversion:** domain (pure business logic, knows nothing else), application (thin orchestration / use cases), ports (entry points: HTTP/gRPC/CLI/Pub-Sub handlers), adapters (exits: DB, external clients). Outer layers may import inner; never the reverse.
3. **DDD Lite = behavior-rich domain types with private fields, constructors that validate, and no getters/setters**, always keep a valid state in memory, reflect business language literally, keep the domain database-agnostic.
4. **CQRS = split application service into command handlers (change state, return no data) and query handlers (return data, change nothing).** Read models are decoupled from write/domain models.
5. **Repository pattern with the `UpdateFn` closure** cleanly handles transactions and optimistic locking without leaking DB details into logic. One repository per aggregate, not per table.
6. **Event-driven architecture with Watermill**, using the message Router, middlewares, and the Forwarder (outbox pattern) for reliable at-least-once delivery; handlers must be idempotent.
7. **Test pyramid: unit (domain), integration (adapters + real infra via Docker), component (single service, mocked externals), end-to-end (all services).** Aim to avoid heavy reliance on E2E.
8. **Idiomatic-Go conventions:** no framework, hand-written dependency injection, port-agnostic slug errors, generated boilerplate (OpenAPI/gRPC/SQL) over `reflect`, explicit struct tags, `go-cleanarch` linter in CI.

## Details

### 1. The Three Dots Labs philosophy and when it applies

Three Dots Labs is the Go-focused blog and training company founded by **Robert Laszczak** and **Miłosz Smółka**, the creators of the **Watermill** event-driven library and authors of the free e-book **Go With The Domain** (60,000+ downloads). Per their own site: "We're pioneers of Domain-Driven Design in Go, publishing our first articles in 2020 and releasing first version of this e-book in 2021. Many patterns we presented have since become the standard way of building DDD services in Go."

Their central thesis: most web/business applications don't solve hard *technical* problems, the real challenge is "not becoming an unmaintainable Big Ball of Mud." They combine DDD Lite, CQRS, Clean Architecture, and the Repository pattern because, used together, they give you "the ability to maintain constant development speed. Without destroying or touching existing code too much" as requirements change.

Key framing principles:
- **"You need to go slow, to go fast."** Taking shortcuts saves time only in the short term. "There is no quality vs. speed tradeoff in the long run. If you want to go fast in the long term, you need to maintain high quality."
- **Evidence-based, not dogmatic.** They lean on *Accelerate* (Forsgren, Humble, Kim), whose research, based on ~23,000 survey responses across 2,000+ organizations, found loosely coupled architecture to be among the strongest predictors of deployment frequency and team performance. They explicitly warn against "authority bias" and trusting patterns just because they're popular.
- **Pragmatism over purity.** "It took me three years to connect all the dots." They deliberately deviate from textbook Hexagonal Architecture where Go idioms suggest a simpler path.

**When to apply:** Complex business domains expected to live longer than ~a month/be maintained by a team. **When NOT to apply:** "If you are creating a project that is not complex and won't be touched any time soon after one month of development, it's probably enough to put everything in one `main` package." Simple, data-oriented CRUD services (like the Wild Workouts `users` service) deliberately skip Clean Architecture and CQRS. Their more recent guidance (AMA, 2026) reiterates: "Start simple and add layers only when you feel the pain of complexity, not because someone said you should."

### 2. Recommended project structure and layer responsibilities

Three Dots Labs use **four layers**, physically separated into packages. Using their own words from the Clean Architecture article:

- **Adapter**, "how your application talks to the external world. You have to *adapt* your internal structures to what the external API expects. Think SQL queries, HTTP or gRPC clients, file readers and writers, Pub/Sub message publishers." (These are Hexagonal's *Secondary Adapters*.)
- **Port**, "an input to your application, and the only way the external world can reach it. It could be an HTTP or gRPC server, a CLI command, or a Pub/Sub message subscriber." (These are Hexagonal's *Primary Adapters*.)
- **Application**, "a thin layer that 'glues together' other layers. It's also known as 'use cases'. If you read this code and can't tell what database it uses or what URL it calls, it's a good sign… Think about it as an orchestrator." It also owns cross-cutting concerns (logging, transactions, instrumentation) and read models.
- **Domain**, "holds just the business logic." Knows nothing about other layers.

**The Dependency Inversion Principle governs imports:**
- Domain knows nothing about other layers (pure business logic).
- Application can import domain, but "has no idea whether it's being called by an HTTP request, a Pub/Sub handler, or a CLI command."
- Ports can import inner layers, but can't directly access Adapters.
- Adapters can import inner layers and operate on domain/application types.

Because Go has implicit interfaces, they **do not keep a separate "ports" layer for interfaces**, they define interfaces next to the code that uses them: the application service says "I need a way to cancel a training with a given UUID… I trust you to do it right if you implement this interface." Dave Cheney's point that "Go interfaces are a perfect match" for dependency inversion is cited.

**Directory layout** (from Wild Workouts): each service (bounded context) under `internal/<service>/` has `domain/`, `app/` (often split into `app/command/` and `app/query/`), `ports/`, and `adapters/`. As the project grows, add a second level (e.g., `adapters/hour/mysql_repository.go`, `ports/http/hour_handler.go`). The repo-level layout is: `api/` (OpenAPI + gRPC/protobuf definitions), `docker/`, `internal/` (application code), `scripts/`, `terraform/`, `web/` (frontend).

**Crucial caveat on structure:** In the anti-patterns article they warn *against* overthinking the directory structure. "If types couple your application, no directory structure will change it." A single model with combined tags couples packages no matter how nicely they're named. The important part "isn't the directory structure but how packages and structures reference each other." Don't start a project by separating directories, "You're unlikely to do it right before you've written any code."

To enforce the layering automatically, use Robert's **`go-cleanarch`** linter locally and in CI.

**The Application struct** collects all handlers as an explicit "menu" of use cases:
```go
type Application struct {
    Commands Commands
    Queries  Queries
}
type Commands struct {
    ApproveTrainingReschedule command.ApproveTrainingRescheduleHandler
    CancelTraining            command.CancelTrainingHandler
    // ...
}
```

### 3. DDD Lite guidance

Three "rules" from the DDD Lite article:

**Rule 1, Reflect business logic literally.** Stop thinking of structs as dumb data with getters/setters; think "types with behavior." Business says "I'm scheduling training at 13:00," not "I'm setting the attribute state to 'training scheduled'." So model behavior:
```go
func (h *Hour) ScheduleTraining() error {
    if !h.IsAvailable() {
        return ErrHourNotAvailable
    }
    h.availability = TrainingScheduled
    return nil
}
```
The litmus test: "Will business stakeholders understand my code without any translation of technical terms?"

**Rule 2, Always keep a valid state in memory.** All fields private, validation in the constructor, the only public API is behavior methods. This achieves DRY validation (one place) and guarantees the object can't be put into an invalid state. Value objects (like `UserID`) use the same encapsulation so that "if you create a new `UserID` and receive no error, you're sure it's valid."

**Rule 3, The domain must be database-agnostic.** Domain types "should be shaped only by business rules," not by the DB. Keep DB structs (`mysqlHour`) entirely separate from domain types (`hour.Hour`), don't add `db:` tags to domain types. This enables optimal storage, easier testing, no duplicated validation, and avoids ORM "magic."

**Entities vs. value objects vs. aggregates:** Entities (like `Training`) have identity and private fields validated by a constructor (`NewTraining`). Value objects are always-valid encapsulated types. **Aggregates** are "a set of data that must always be consistent", they define transaction and repository boundaries (see §5). They caution that guessing entities from nouns is not rigorous; proper discovery uses Strategic DDD and Event Storming (a topic they treat separately).

**Avoiding anemic models:** If you see `if`s related to business logic in the application layer, move them to the domain. Domain logic "is fairly stable after initial development and can live unchanged for a long time." Small helper *functions* in the domain package are fine in Go, you don't need a Java-style "domain service" object for every calculation (e.g., `CancelBalanceDelta(tr Training, ...) int`).

**Domain-First approach:** For complex projects, spend "2-4 weeks working on the domain layer with just an in-memory database implementation," all driven by unit tests, deferring the database decision. Timebox it; it requires business trust.

**Testing helpers:** Provide `MustNewX` constructors (panic on invalid input, fine for tests), custom asserts (`assertTrainingsEquals`), and use `github.com/google/go-cmp` with `cmp.AllowUnexported` to compare domain types with private fields.

### 4. CQRS guidance

CQRS (Greg Young) means "instead of having one big model for reads and writes, you should have two separate models. One for writes and one for reads." Three Dots Labs stress it is *simple*, not enterprise-heavy: "In practice, CQRS is a simple pattern that doesn't require a lot of investment."

**Command vs. Query rule:** "A Query should not modify anything, just return data. A command is the opposite: it should make changes in the system but not return any data." (Side effects like logs/metrics don't count; commands returning an error is fine.)

**Command structure:**
```go
type ApproveTrainingReschedule struct {
    TrainingUUID string
    User         training.User   // use domain-defined types for type safety
}
```
Then a dedicated handler type (`ApproveTrainingRescheduleHandler`) with a `Handle(ctx, cmd)` method that orchestrates: load aggregate via repository, call domain behavior, call external services, return the updated aggregate to be saved. The domain decides *whether* an operation is allowed; the handler only orchestrates.

Even trivial commands get their own type: "writing code is always much cheaper than maintenance. Adding this simple type takes 3 minutes."

**Queries** use a `ReadModel` interface returning query-specific types (e.g., `query.Date`), which can be more complex than domain types and evolve independently. You don't need a read model interface for every query, it's fine to use the domain repository for simple reads. Queries are a great place for cross-cutting concerns (logging/instrumentation) so they're measured consistently regardless of port.

**Naming:** Use ubiquitous language, "Schedule training" / "Cancel training," not "Create/Delete training." "Think twice about whether any of your command names really need to start with Create/Delete/Update."

**When to use CQRS:** Complex business services. **When NOT to:** authentication, simple CRUD that receives and returns the same data (Wild Workouts `users` service). But watch such services, if logic grows, refactor.

**Returning created entities via REST with CQRS:** Preferred approach, generate the UUID client-side/in the port, pass it in the command, and return `204 No Content` with a `content-location` header pointing to the new resource, rather than returning the entity from the command.

**Future extensions CQRS unlocks** (only when needed): async commands (via an async command bus / Watermill); a separate read database (polyglot persistence, e.g., Elasticsearch) synced via events, accepting eventual consistency; and event sourcing (natural fit for audit-heavy/financial domains). "It's good to defer key decisions like this one."

**Splitting into microservices is often wrong for cohesive command sets:** "Many operations here need to be transactionally consistent. Splitting into separate services would involve several distributed transactions (Sagas)… It's not a good tradeoff. Complexity scales horizontally here."

### 5. Repository pattern, transactions, and updating aggregates

The repository "abstracts our database implementation by defining the interaction with it through an interface" that is "free of any implementation details of any specific database." Interfaces are defined in the domain/app package (like `io.Writer`); implementations live in `adapters/`. Success test: "If you can swap the database, it's a sign that you implemented the repository pattern correctly." They demonstrate in-memory, MySQL (`sqlx`), and Firestore implementations behind one interface.

**Reads** use context-aware methods (e.g., `GetOrCreateHour(ctx, ...)`) and map DB "transport types" to domain types.

**The `UpdateFn` closure pattern is their go-to for transactions and updating aggregates:**
```go
type Repository interface {
    AddTraining(ctx context.Context, tr *Training) error
    GetTraining(ctx context.Context, trainingUUID string, user User) (*Training, error)
    UpdateTraining(
        ctx context.Context,
        trainingUUID string,
        user User,
        updateFn func(ctx context.Context, tr *Training) (*Training, error),
    ) error
}
```
Inside one transaction the repository: gets the entity, runs the closure (which applies domain behavior), saves the returned value, and rolls back on error. This keeps logic in the command handler and transaction handling in the repository. They explicitly reject alternatives they tried (passing `tx` via `context.Context`, transaction middleware at the HTTP/gRPC level) as "a bit magical, not explicit, and slow."

**Optimistic locking / concurrency:** the MySQL implementation uses `SELECT ... FOR UPDATE` to lock the row so parallel transactions can't corrupt state; a `finishTransaction` helper commits on success and rolls back on error (combining errors with `go.uber.org/multierr`), driven by a named return `(err error)` and `defer`.

**Transactions in layered architecture** (dedicated article) ranks the approaches:
- ❌ **Skipping transactions**, "If something can theoretically happen, it will likely happen in production." Use transactions for interdependent queries; add `FOR UPDATE` for concurrent safety.
- ⚠️ **Transactions in the logic layer** (passing `tx` around), works but leaks implementation details into logic, complicates flow, and forces integration tests where unit tests should suffice. Avoid if you can.
- ✅ **Transactions inside the repository, one repository per aggregate**, "Don't create a repository for each database table. Instead, think of the data that needs to be transactionally stored together." Data that must be consistent belongs in one aggregate with one repository.
- ✅ **The `UpdateFn` pattern**, their recommended default; logic in the closure, transaction in the repository.
- ✅ **The Transaction Provider** (for edge cases like audit logs spanning repositories), a `Transact(func(adapters Adapters) error)` method injecting transaction-bound adapters. "Be careful not to over-use it… Stick to the `UpdateFn` pattern instead." Warning: combined with `FOR UPDATE`, every `SELECT` must include the clause; consider `REPEATABLE READ` instead.

On premature optimization of the `Update` method updating all fields: "Unless you're saving huge datasets or dealing with massive scale, it shouldn't really matter… Be pragmatic and avoid premature optimization. If in doubt, run stress tests."

### 6. Event-driven architecture with Watermill

**Watermill** is Three Dots Labs' open-source library for message/event-driven Go applications, "Think of it like an HTTP router but for messages. It's a library, not a framework, so you don't need to change your architecture to use it." Watermill 1.0 shipped in 2019 with a stable public API; by the v1.4 release it had grown to roughly 7.5k GitHub stars and 55 contributors, and supports 12 Pub/Sub implementations including Kafka, Redis Streams, NATS, Google Cloud Pub/Sub, Amazon SQS/SNS, SQL (Postgres/MySQL), and RabbitMQ/AMQP (plus Firestore, Bolt, Go channels, HTTP). Its core interface: `func(*Message) ([]*Message, error)`.

**Router and HandlerFunc.** The Router is the high-level API providing correlation, metrics, poison queue, retry, throttling, etc. The handler signature:
```go
type HandlerFunc func(msg *Message) ([]*Message, error)
```
`msg.Ack()` is called automatically when the handler returns no error; `msg.Nack()` when it returns an error. `AddHandler(name, subscribeTopic, subscriber, publishTopic, publisher, handlerFunc)` requires a unique handler name; `AddNoPublisherHandler`/`AddConsumerHandler` are for handlers that don't publish.

**Middlewares** (decorators of type `func(HandlerFunc) HandlerFunc`, added router-wide or per-handler):
- **Retry**, retries the handler with exponential backoff; after `MaxRetries` the message is Nacked and the Pub/Sub redelivers.
- **PoisonQueue**, "salvages unprocessable messages and publishes them on a separate topic. The main middleware chain then continues on."
- **CircuitBreaker**, "fail fast if the handler keeps returning errors… useful for preventing cascading failures" (backed by `gobreaker`).
- **Throttle**, limits messages processed per unit of time.
- **Timeout**, cancels the message context after a set time.
- **CorrelationID**, propagates a correlation ID to all produced messages (essential for tracing async flows).
- **Recoverer**, recovers from handler panics, appending the stacktrace to the error.
- **InstantAck**, instantly acks regardless of errors (trades exactly-once/ordering for throughput).
- Others: **Deduplicator**, **Duplicator** (deliberately processes twice to force idempotency), **IgnoreErrors**, **DelayOnError**.

**Delivery guarantees and idempotency.** Watermill is built with **at-least-once delivery**: "if an error occurs while processing a message and an Ack cannot be sent, the message will be redelivered. You need to keep it in mind and build your application to be idempotent or implement a deduplication mechanism." Watermill deliberately does not ship a universal dedup middleware ("it's not possible to create a universal middleware for deduplication, so we encourage you to build your own"). `Message.Ack()` and `Nack()` are both non-blocking and idempotent. GoChannel is the one exactly-once exception; real brokers are at-least-once.

**The Outbox pattern via the Forwarder.** The classic problem, in Watermill's words: "you'd want to persist the application state and publish the message in a transaction, as not doing so might get you easily into troubles with data consistency." The answer is to publish the event **into the same database (same transaction) as your business data**, then have the **Forwarder**, "a background running daemon which awaits messages that are published to a database, and makes sure they eventually reach a message broker", move them to the real broker (e.g., SQL → Kafka/Google Cloud Pub/Sub). The article contrasts three orderings:
1. Publish event, then store data, risk: event emitted but no data persisted (leaks failure outside the component).
2. Store data, then publish event, risk: data persisted but no event (contained failure, but manual recovery needed).
3. **Store data and publish event in one transaction**, the recommended outbox approach; as the docs put it, "any of them can't succeed having the other failed."

Mechanically: the command uses an SQL publisher bound to the transaction (`sql.NewPublisher(sql.TxFromStdSQL(tx), ...)`), decorated with `forwarder.NewPublisher(...)`; a separate Forwarder (`forwarder.NewForwarder(sqlSubscriber, brokerPublisher, logger, forwarder.Config{ForwarderTopic: ...})`) runs as a daemon. `watermill-sql` provides default schemas (`DefaultMySQLSchema`, `DefaultPostgreSQLSchema`).

**CQRS component.** A high-level API letting you "work with Go structs instead of messages," built on Pub/Sub and Router. Building blocks: **CommandBus** (`Send`), **EventBus** (`Publish`), **CommandProcessor**, **EventProcessor**, and **EventGroupProcessor** (multiple handlers sharing one subscriber to preserve event ordering). Every command has exactly one handler; an event can have multiple handlers, and handlers must be thread-safe. Since v1.3, generic handlers exist: `cqrs.NewCommandHandler[Command]` and `cqrs.NewEventHandler[T]`. Marshalers: `cqrs.JSONMarshaler{}` and `cqrs.ProtoMarshaler{}`. "You don't need to implement the entire CQRS. It's very common to use just the event part."

**Guidance on going async (No Silver Bullet / EDA "Hard Parts"):** "Start with synchronous architecture by default, it's simpler to understand, debug, and maintain." Async improves scalability/resilience but "require[s] more experienced teams and better tooling." "It's easier to migrate from a well-structured synchronous system to async later than the other way around." Other hard-parts lessons: observability (tracing, logs, correlation IDs) is essential; use the outbox pattern to avoid losing events; design events carefully, "large, generic events can lead to tight coupling and painful refactors"; avoid over-engineering.

### 7. Testing strategy

Their "test architecture" maps test types to the layers, following Simon Stewart's "Test Sizes" advice: create a table so the team agrees on what each test type means:

| Feature | Unit | Integration | Component | End-to-End |
|---|---|---|---|---|
| Docker database | No | Yes | Yes | Yes |
| Use external systems | No | No | No | Yes |
| Focused on business cases | Depends | No | Yes | Yes |
| Used mocks | Most deps | Usually none | External systems | None |
| Tested API | Go package | Go package | HTTP and gRPC | HTTP |

- **Unit tests**, the domain layer is pure logic, "the tests here should be some of the simplest to write and run super fast." Aim for high domain coverage, test only exported code (black-box via `_test` package suffix), use table-driven tests. Application-layer commands with real orchestration get unit tests with hand-written mocks (Dependency Inversion makes this trivial). Don't test code that just glues layers, "you don't want to end up testing the mocks."
- **Integration tests**, "checks if an adapter works correctly with external infrastructure," mostly DB repositories via docker-compose. "There's no reason for integration tests to be slow and flaky… practices like automatic retries and increasing sleep times should be absolutely out of the question." Run in parallel with `t.Parallel()`; design tests not to interfere (don't assert list lengths; scope each test to a unique user/context). Pin the Docker image to the production version.
- **Component tests**, "check the completeness of a single service in isolation, with all infrastructure it needs." Call real ports (HTTP/gRPC) but mock adapters to external services (via a second constructor like `NewComponentTestApplication` injecting mocks). Started once via `TestMain`; use a `WaitForPort` helper, "Don't replace it with sleeps." Test the happy path only; corner cases belong in unit/integration tests.
- **End-to-end tests**, spin up all services in docker-compose, test a few critical paths over public HTTP. "They should test whether services connect together correctly, not the logic inside them… They shouldn't fail most of the time, and if they do, it usually means someone broke the contract." Keep them short and few.

The whole point is that a well-architected system lets you "do most of our testing without requiring an integrated environment" (an *Accelerate* marker of elite teams). A Go 1.21-and-earlier gotcha is highlighted: loop-variable capture in parallel table tests (fixed in Go 1.22).

### 8. Practical conventions and Go anti-patterns

**No framework.** "One of the worst things you can do in Go is follow an approach from other programming languages." Frameworks save time at bootstrap but you "hit the framework's wall of conventions and limitations." Use targeted libraries instead. Their vetted list is the "22 libraries that never failed us" article: e.g., **chi** (HTTP router), **oapi-codegen** (OpenAPI), **sqlx**/**sqlc** for SQL, **Watermill** for messaging, plus observability and testing libs.

**Dependency injection by hand.** Start by wiring dependencies in `main.go` with plain constructors that validate inputs (panic on missing dependency). They tried Google's **wire** (code-generation, their recommendation *if* you need a DI tool because "you can just go to the generated file and see how everything is injected") but "found out that it's just better to inject by hand because you see what's happening there." If hand-wiring becomes very hard, "it may be a problem of your code… maybe your dependency graph is too complicated", DI difficulty is a smell, not a reason to reach for reflection-based frameworks.

**Error handling.** Embrace Go's explicit `if err != nil { return err }`, "It makes you repeat yourself, but it's not the duplication DRY tells you to avoid." Use **port-agnostic slug errors** in the application layer so both HTTP and gRPC ports can translate them (e.g., `errors.NewIncorrectInputError("date-from-after-date-to", "...")` → `400` + a slug the frontend can localize). Their "unpopular opinions" stance: "Go's error handling is fine for most projects."

**Configuration & databases.** Their 2026 default database recommendation is **just use PostgreSQL** ("easier to operate and simpler to migrate between cloud providers"). For SQL code generation, **sqlc** is now their default for new projects (SQLBoiler is in maintenance mode).

**The single-model anti-pattern (the big one).** "Don't give a single model more than one responsibility. Don't use more than one tag per structure field." A struct carrying `json` + `gorm` + `validate` tags "gives you one of the worst issues: strong coupling between the API, storage, and logic." Instead keep separate HTTP (request/response), DB, and domain models and write "plain and obvious functions to convert between them." Duplication here is cheaper than coupling: "In contrast to tightly-coupled code, fixing duplicated code is trivial."

**Other anti-patterns they warn about:**
- **DRY taken too far** introduces premature coupling. "When you make two things use a common abstraction, you introduce coupling."
- **Prefer generated code over `reflect`** for boilerplate (OpenAPI/gRPC/SQL), strong types, compile-time safety. But "don't overuse libraries", don't let generated models become a single coupled model; watch out for "magic" in struct tags (gorm permissions, validator cross-field rules) that trades away compile-time checks.
- **Always fill struct tags explicitly**, even if names match, to avoid silent breakage when fields are renamed.
- **Don't start from the database schema**, "you will end up exposing implementation details" (e.g., leaking a "membership" table concept nobody in the business uses). Model from the domain; storage methods should follow product behavior.
- **Your web app is not a CRUD.** "Don't design your application around the idea of four CRUD operations… It's the special rules and weird details that make your application different."
- **The Distributed Monolith.** "Don't split your application into microservices before you know the boundaries." Microservices don't lower coupling, "it's not important how many times you split the application. What matters is how you connect the pieces."

### 9. Microservices vs. modular monolith, and trade-offs

Their long-standing position: **whether an app is a monolith or microservices should be an implementation detail** of the ports/adapters layers. If you build a "Clean Monolith" with proper bounded-context separation, "the decision to move to microservices architecture will be not a problem." The domain and application layers stay identical between the two; only interfaces/infrastructure differ (e.g., an in-process function call vs. an HTTP/gRPC/AMQP call).

They cite Martin Fowler's "MonolithFirst" (martinfowler.com, June 2015): "Almost all the successful microservice stories have started with a monolith that got too big and was broken up. Almost all the cases where I've heard of a system that was built as a microservice system from scratch, it has ended up in serious trouble."

Recent distilled takeaways (2025–2026):
- **Start with a monolith**, microservices overhead isn't worth it until you have real pain points.
- **Microservices solve human/organizational problems, not just technical ones**, they let teams work independently.
- **Tight coupling over HTTP is still tight coupling**, separating services doesn't automatically give isolation.
- **Watch for signals of wrong boundaries**, lots of cross-service calls, or frequent changes spanning multiple services.
- **Sometimes joining services back together is the right move.**

**gRPC and OpenAPI for boundaries:** For internal service-to-service communication they favor **gRPC** for robust contracts, "The way gRPC generates servers and clients is much stricter than OpenAPI… gRPC works great for synchronous communication, but not every process is synchronous by nature." For public HTTP APIs they use **OpenAPI** with `oapi-codegen` to generate the Go server and JavaScript client from a single spec, keeping generated HTTP models separate from domain and DB models. Not everything should be synchronous, apply gRPC where synchronous calls fit, and Watermill/events where they don't.

**Distributed transactions / sagas:** "It's a solved problem. Sometimes, there are good reasons to use them. But more often, it's overkill." "If you feel like you need distributed transactions between three services, maybe you should just merge them into one." Enforce bounded-context boundaries with a CI tool (`go-cleanarch`); discover them with **Event Storming**.

**When to break the rules:** They are explicit that these are guidelines, not laws. CQRS's command/query separation can be broken "as long as you understand why they were introduced and what tradeoffs you're making." Clean Architecture "is most beneficial for complex projects with larger teams, for small teams or simple projects, it can become overengineering." "It's easy to go too far, using too many interfaces or too many layers without a clear reason creates unnecessary complexity."

### 10. How to adopt this on a new project (learning order)

Their recommended learning/adoption order (from the monolith article and AMA): **start with Clean Architecture, then basic CQRS, then DDD.** "You don't need to learn all of these techniques at the same time." Kick off with planning/Event Storming, "There is no 'lack of design', it is only good design and bad design."

For refactoring an existing mess, they recommend **pair or mob programming** with strict timeboxes and daily integration/deployment so stakeholders keep trust.

**References for deeper reading (primary sources):**
- *Go With The Domain* e-book and the 14-article "Modern Business Software in Go" series (threedots.tech/go-with-the-domain, /series/modern-business-software-in-go)
- Introduction to DDD Lite (`/post/ddd-lite-in-go-introduction/`)
- How to implement Clean Architecture in Go (`/post/introducing-clean-architecture/`)
- How to use basic CQRS in Go (`/post/basic-cqrs-in-go/`)
- Combining DDD, CQRS, and Clean Architecture in Go (`/post/ddd-cqrs-clean-architecture-combined/`)
- The Repository pattern in Go (`/post/repository-pattern-in-go/`)
- Database Transactions in Go with Layered Architecture (`/post/database-transactions-in-go/`) and Distributed Transactions in Go (`/post/distributed-transactions-in-go/`)
- Microservices test architecture (`/post/microservices-test-architecture/`)
- When using Microservices or Modular Monolith in Go can be just a detail (`/post/microservices-or-monolith-its-detail/`)
- Common Anti-Patterns in Go Web Applications (`/post/common-anti-patterns-in-go-web-applications/`) + Safer Enums in Go
- The Best Go framework: no framework? (`/post/best-go-framework/`) and The Go libraries that never failed us (`/post/list-of-recommended-libraries/`)
- Robust gRPC communication on Google Cloud Run (`/post/robust-grpc-google-cloud-run/`)
- Watermill docs (watermill.io), Router, Middlewares, Pub/Sub, Forwarder (outbox), CQRS component
- Repos: `github.com/ThreeDotsLabs/wild-workouts-go-ddd-example`, `.../go-web-app-antipatterns`, `.../monolith-microservice-shop`, `.../watermill`; linter `github.com/roblaszczak/go-cleanarch`

## Recommendations

**Stage 0, Decide whether you even need this.** If the service is a genuine CRUD, or a throwaway/PoC, keep it in one package and skip the layers. Adopt the full stack only for complex, long-lived business domains or multi-person teams. *Threshold to escalate:* you start seeing business `if`s spread across HTTP handlers, fear of touching code, or slowing feature delivery.

**Stage 1, Establish the skeleton.**
- Create `internal/<boundedcontext>/{domain,app,ports,adapters}`; put `app/command/` and `app/query/` if using CQRS.
- Choose no framework; add chi (HTTP), your OpenAPI/gRPC codegen, PostgreSQL + sqlc, and Watermill only if you need messaging.
- Wire dependencies by hand in `main.go`. Add the `go-cleanarch` linter to CI.
- Generate HTTP/gRPC/DB boilerplate; keep separate HTTP, DB, and domain models with hand-written mapping functions. One tag per field.

**Stage 2, Model the domain first.** Timebox 2–4 weeks (for complex domains) building behavior-rich domain types with private fields, validating constructors, and an in-memory repository, all unit-tested. Defer the database decision.

**Stage 3: Add persistence and orchestration.** Implement the repository with the `UpdateFn` closure pattern and one repository per aggregate; use transactions + `FOR UPDATE` (or `REPEATABLE READ`). Keep command handlers thin.

**Stage 4, Build the test pyramid.** High-coverage unit tests on the domain; integration tests for adapters (docker-compose, parallel, pinned image versions); component tests per service (mock external adapters); a few E2E happy-path tests. Run all in CI.

**Stage 5, Go event-driven only when needed.** Default to synchronous. When you add async, use Watermill's Router + middlewares (Retry, PoisonQueue, CorrelationID, Recoverer), design idempotent handlers, and use the Forwarder/outbox pattern for atomic persist-and-publish. Add tracing/observability from day one of async.

**Stage 6, Stay a modular monolith until you have a concrete reason to split.** Split only along well-understood bounded contexts; treat monolith-vs-microservices as a deployment detail. Avoid distributed transactions/sagas, prefer merging tightly coupled services.

*Benchmarks that should change your approach:* if hand-wiring DI becomes painful → simplify the dependency graph (don't reach for a DI framework); if E2E tests become slow/flaky/numerous → push coverage down to component/integration; if cross-service calls proliferate → your service boundaries are wrong (consider merging).

## Caveats

- **These are opinionated, experience-based guidelines, not universal laws.** The authors repeatedly stress "it depends" and warn that all of this is overengineering for simple projects. Take positions deliberately and document the trade-offs.
- **Wild Workouts intentionally contains anti-patterns** in its early commits, the 14-article series refactors them over time. When reading the repo, follow the article/tag versions (e.g., `v2.4`, `v2.5`) rather than assuming any given commit is exemplary.
- **Some Three Dots Labs specifics have been superseded by their own later advice:** they now recommend PostgreSQL over Firestore, sqlc over SQLBoiler (now in maintenance mode), and Go 1.22 removes the parallel-test loop-variable gotcha. The original Wild Workouts used Firestore/Google Cloud Run; treat those as implementation details.
- **Strategic DDD (Event Storming, bounded-context discovery, ubiquitous language) is essential but under-covered** in the tactical articles, the authors note that without strategic patterns "you'll only get 30% of the advantages that DDD can offer." A greenfield team should invest in Event Storming up front.
- **Content dates:** the core "Modern Business Software in Go" series was published 2020–2021 and last updated through 2024–2026; the transactions, anti-patterns, distributed-transactions, and library articles are newer (2021–2024). Watermill is at a stable v1.x API.
- This report synthesizes primary Three Dots Labs sources (blog articles, Wild Workouts and go-web-app-antipatterns repos, Watermill docs). A few corroborating details on Watermill delivery semantics come from the official watermill.io docs, which embed the library's own source-code comments.
