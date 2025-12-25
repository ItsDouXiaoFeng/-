# Event-Sourced Payment Service: Implementation Plan

## Document Information

| Attribute | Value |
|-----------|-------|
| Author | Software Architecture Team |
| Status | Active |
| Last Updated | December 2024 |
| Target Completion | 6-8 months (5 hours/week) |

---

## 1. Executive Summary

This document outlines the implementation strategy for a production-grade payment processing system built on CQRS and Event Sourcing principles. The architecture prioritizes correctness, auditability, and scalability while maintaining developer productivity through strong typing and functional programming patterns.

### Success Criteria

- Complete payment lifecycle management (create, authorize, capture, refund)
- Full event history with temporal query capability
- Idempotent operations with exactly-once semantics
- Saga-based multi-step transaction coordination
- Observable, testable, and deployable to Kubernetes

---

## 2. Architecture Deep Dive

### 2.1 System Context

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API Gateway                                     │
│                    (Rate Limiting, Auth, Idempotency)                       │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
┌───────────────────┴───────────────┐ ┌────────┴──────────────────────────────┐
│           Command Side            │ │              Query Side               │
│                                   │ │                                       │
│  ┌─────────────────────────────┐  │ │  ┌─────────────────────────────────┐  │
│  │     Command Handlers        │  │ │  │      Read Model Service         │  │
│  │  ┌───────────────────────┐  │  │ │  │   (Optimized Projections)       │  │
│  │  │ Validation Layer      │  │  │ │  └──────────────┬──────────────────┘  │
│  │  │ Business Rules        │  │  │ │                 │                     │
│  │  │ Event Generation      │  │  │ │                 ▼                     │
│  │  └───────────────────────┘  │  │ │  ┌─────────────────────────────────┐  │
│  └──────────┬──────────────────┘  │ │  │         PostgreSQL              │  │
│             │                     │ │  │     (Denormalized Views)        │  │
│             ▼                     │ │  └─────────────────────────────────┘  │
│  ┌─────────────────────────────┐  │ │                 ▲                     │
│  │       Event Store           │  │ │                 │                     │
│  │  ┌───────────────────────┐  │  │ └─────────────────┼─────────────────────┘
│  │  │   Kafka (Event Log)   │──┼──┼─── Projections ───┘
│  │  └───────────────────────┘  │  │    (FS2-Kafka Consumer)
│  │  ┌───────────────────────┐  │  │
│  │  │ Cassandra (Snapshots) │  │  │
│  │  └───────────────────────┘  │  │
│  └─────────────────────────────┘  │
└───────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Saga Orchestrator                                   │
│              (Multi-step Payments, Compensation Logic)                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Request Flow

1. **API Gateway** receives payment request, checks idempotency key
2. **Command Handler** validates request, loads aggregate from event store
3. **Aggregate** applies business rules, generates new events
4. **Event Store** persists events to Kafka (primary) with Cassandra snapshots
5. **Projections** consume events via FS2-Kafka, update PostgreSQL read models
6. **Query Service** serves optimized reads from PostgreSQL
7. **Saga Orchestrator** coordinates multi-step flows with compensation

### 2.3 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Event Store Primary | Kafka | Ordered, partitioned log with replay; exactly-once semantics |
| Event Store Secondary | Cassandra | Efficient aggregate replay by ID; time-range queries |
| Read Store | PostgreSQL | ACID guarantees for projections; familiar query patterns |
| Serialization | Protocol Buffers | Schema evolution support; compact binary format |
| Effect System | Cats Effect 3 | Composable concurrency; resource safety; testability |
| HTTP Framework | http4s | Pure functional; excellent CE3 integration |

---

## 3. Domain Model Specification

### 3.1 Core Value Objects

#### Money

```scala
package payments.domain

import cats.kernel.{Monoid, Order}

/**
 * Represents monetary amounts with currency.
 * Uses minor units (cents) internally to avoid floating-point issues.
 *
 * Design decisions:
 * - Opaque type for zero-cost abstraction
 * - Currency-aware arithmetic prevents accidental mixing
 * - Overflow protection on all operations
 */
opaque type Money = (Long, Currency)

object Money:
  /** Maximum value to prevent overflow: $92 trillion in cents */
  private val MaxMinorUnits: Long = Long.MaxValue / 100

  def apply(amount: BigDecimal, currency: Currency): Either[MoneyError, Money] =
    val scale = currency.minorUnitDigits
    val scaled = amount.setScale(scale, BigDecimal.RoundingMode.UNNECESSARY)
    val minorUnits = (scaled * BigDecimal(Math.pow(10, scale).toLong)).toLongExact

    if minorUnits < 0 then Left(MoneyError.NegativeAmount)
    else if minorUnits > MaxMinorUnits then Left(MoneyError.Overflow)
    else Right((minorUnits, currency))

  def fromMinorUnits(units: Long, currency: Currency): Money = (units, currency)

  def zero(currency: Currency): Money = (0L, currency)

  extension (m: Money)
    def amount: BigDecimal =
      BigDecimal(m._1) / BigDecimal(Math.pow(10, m._2.minorUnitDigits).toLong)

    def minorUnits: Long = m._1
    def currency: Currency = m._2

    def +(other: Money): Either[MoneyError, Money] =
      if m._2 != other._2 then
        Left(MoneyError.CurrencyMismatch(m._2, other._2))
      else
        val sum = m._1 + other._1
        if sum < m._1 then Left(MoneyError.Overflow) // Overflow check
        else Right((sum, m._2))

    def -(other: Money): Either[MoneyError, Money] =
      if m._2 != other._2 then
        Left(MoneyError.CurrencyMismatch(m._2, other._2))
      else if other._1 > m._1 then
        Left(MoneyError.NegativeAmount)
      else
        Right((m._1 - other._1, m._2))

    def isPositive: Boolean = m._1 > 0
    def isZero: Boolean = m._1 == 0
    def negate: Money = (-m._1, m._2)

enum Currency(val code: String, val minorUnitDigits: Int):
  case USD extends Currency("USD", 2)
  case EUR extends Currency("EUR", 2)
  case GBP extends Currency("GBP", 2)
  case JPY extends Currency("JPY", 0)

enum MoneyError:
  case CurrencyMismatch(expected: Currency, actual: Currency)
  case Overflow
  case NegativeAmount
  case InvalidPrecision(maxDigits: Int, actualDigits: Int)
```

#### PaymentId

```scala
package payments.domain

import java.util.UUID

/**
 * Strongly-typed payment identifier.
 * Prevents accidental mixing with other ID types.
 */
opaque type PaymentId = UUID

object PaymentId:
  def generate: PaymentId = UUID.randomUUID()
  def fromString(s: String): Either[String, PaymentId] =
    try Right(UUID.fromString(s))
    catch case _: IllegalArgumentException => Left(s"Invalid PaymentId: $s")

  def unsafeFromString(s: String): PaymentId = UUID.fromString(s)

  extension (id: PaymentId)
    def value: UUID = id
    def asString: String = id.toString
```

### 3.2 Aggregate: Payment

```scala
package payments.domain

import java.time.Instant

/**
 * Payment aggregate - the consistency boundary for payment operations.
 *
 * Invariants:
 * - Cannot authorize more than original amount
 * - Cannot capture more than authorized amount
 * - Cannot refund more than captured amount
 * - State transitions follow defined state machine
 */
final case class Payment private (
  id: PaymentId,
  amount: Money,
  status: PaymentStatus,
  authorizedAmount: Money,
  capturedAmount: Money,
  refundedAmount: Money,
  metadata: Map[String, String],
  createdAt: Instant,
  updatedAt: Instant,
  version: Long
)

object Payment:
  def empty(id: PaymentId, currency: Currency): Payment = Payment(
    id = id,
    amount = Money.zero(currency),
    status = PaymentStatus.Initial,
    authorizedAmount = Money.zero(currency),
    capturedAmount = Money.zero(currency),
    refundedAmount = Money.zero(currency),
    metadata = Map.empty,
    createdAt = Instant.EPOCH,
    updatedAt = Instant.EPOCH,
    version = 0
  )

  /**
   * Apply an event to produce new state.
   * This is the core of event sourcing - state is derived from events.
   */
  def apply(state: Payment, event: PaymentEvent): Payment = event match
    case PaymentCreated(id, amount, currency, metadata, timestamp) =>
      Payment(
        id = id,
        amount = Money.fromMinorUnits(amount, currency),
        status = PaymentStatus.Created,
        authorizedAmount = Money.zero(currency),
        capturedAmount = Money.zero(currency),
        refundedAmount = Money.zero(currency),
        metadata = metadata,
        createdAt = timestamp,
        updatedAt = timestamp,
        version = 1
      )

    case PaymentAuthorized(_, authorizedAmount, _, timestamp) =>
      state.copy(
        status = PaymentStatus.Authorized,
        authorizedAmount = Money.fromMinorUnits(authorizedAmount, state.amount.currency),
        updatedAt = timestamp,
        version = state.version + 1
      )

    case PaymentCaptured(_, capturedAmount, _, timestamp) =>
      val newCaptured = state.capturedAmount.minorUnits + capturedAmount
      state.copy(
        status = if newCaptured >= state.authorizedAmount.minorUnits
                 then PaymentStatus.Captured
                 else PaymentStatus.PartiallyCaptured,
        capturedAmount = Money.fromMinorUnits(newCaptured, state.amount.currency),
        updatedAt = timestamp,
        version = state.version + 1
      )

    case PaymentRefunded(_, refundedAmount, _, timestamp) =>
      val newRefunded = state.refundedAmount.minorUnits + refundedAmount
      state.copy(
        status = if newRefunded >= state.capturedAmount.minorUnits
                 then PaymentStatus.Refunded
                 else PaymentStatus.PartiallyRefunded,
        refundedAmount = Money.fromMinorUnits(newRefunded, state.amount.currency),
        updatedAt = timestamp,
        version = state.version + 1
      )

    case PaymentFailed(_, reason, _, timestamp) =>
      state.copy(
        status = PaymentStatus.Failed,
        updatedAt = timestamp,
        version = state.version + 1
      )

    case PaymentCancelled(_, reason, timestamp) =>
      state.copy(
        status = PaymentStatus.Cancelled,
        updatedAt = timestamp,
        version = state.version + 1
      )

  /**
   * Fold a sequence of events to reconstruct state.
   * Used when replaying from event store.
   */
  def fold(id: PaymentId, currency: Currency, events: List[PaymentEvent]): Payment =
    events.foldLeft(empty(id, currency))(apply)
```

### 3.3 Payment State Machine

```scala
package payments.domain

/**
 * Payment lifecycle states.
 * Defines valid transitions - invalid transitions are compile-time errors.
 */
enum PaymentStatus:
  case Initial
  case Created
  case Authorized
  case PartiallyCaptured
  case Captured
  case PartiallyRefunded
  case Refunded
  case Failed
  case Cancelled

object PaymentStatus:
  /**
   * Valid state transitions.
   * Used for validation before generating events.
   */
  def canTransitionTo(from: PaymentStatus, to: PaymentStatus): Boolean =
    (from, to) match
      case (Initial, Created) => true
      case (Created, Authorized) => true
      case (Created, Failed) => true
      case (Created, Cancelled) => true
      case (Authorized, PartiallyCaptured) => true
      case (Authorized, Captured) => true
      case (Authorized, Cancelled) => true
      case (Authorized, Failed) => true
      case (PartiallyCaptured, PartiallyCaptured) => true
      case (PartiallyCaptured, Captured) => true
      case (Captured, PartiallyRefunded) => true
      case (Captured, Refunded) => true
      case (PartiallyRefunded, PartiallyRefunded) => true
      case (PartiallyRefunded, Refunded) => true
      case _ => false

  def isFinal(status: PaymentStatus): Boolean = status match
    case Refunded | Failed | Cancelled => true
    case _ => false
```

### 3.4 Event Definitions

```scala
package payments.domain

import java.time.Instant

/**
 * Sealed trait hierarchy for payment events.
 * Each event represents a fact that happened - immutable and append-only.
 *
 * Event Versioning Strategy:
 * - Use Protocol Buffers with explicit field numbers
 * - Never remove fields, only deprecate
 * - Add new optional fields for evolution
 * - Schema registry enforces compatibility
 */
sealed trait PaymentEvent:
  def paymentId: PaymentId
  def timestamp: Instant
  def eventVersion: Int = 1

final case class PaymentCreated(
  paymentId: PaymentId,
  amount: Long,           // Minor units
  currency: Currency,
  metadata: Map[String, String],
  timestamp: Instant
) extends PaymentEvent

final case class PaymentAuthorized(
  paymentId: PaymentId,
  authorizedAmount: Long, // Minor units
  authorizationCode: String,
  timestamp: Instant
) extends PaymentEvent

final case class PaymentCaptured(
  paymentId: PaymentId,
  capturedAmount: Long,   // Minor units
  captureReference: String,
  timestamp: Instant
) extends PaymentEvent

final case class PaymentRefunded(
  paymentId: PaymentId,
  refundedAmount: Long,   // Minor units
  refundReference: String,
  timestamp: Instant
) extends PaymentEvent

final case class PaymentFailed(
  paymentId: PaymentId,
  reason: String,
  errorCode: Option[String],
  timestamp: Instant
) extends PaymentEvent

final case class PaymentCancelled(
  paymentId: PaymentId,
  reason: String,
  timestamp: Instant
) extends PaymentEvent
```

### 3.5 Command Definitions

```scala
package payments.domain

/**
 * Commands represent intent to change state.
 * Commands can fail; events are facts.
 */
sealed trait PaymentCommand:
  def paymentId: PaymentId

final case class CreatePayment(
  paymentId: PaymentId,
  amount: Money,
  metadata: Map[String, String],
  idempotencyKey: String
) extends PaymentCommand

final case class AuthorizePayment(
  paymentId: PaymentId,
  amount: Option[Money], // None = full amount
  idempotencyKey: String
) extends PaymentCommand

final case class CapturePayment(
  paymentId: PaymentId,
  amount: Option[Money], // None = full authorized amount
  idempotencyKey: String
) extends PaymentCommand

final case class RefundPayment(
  paymentId: PaymentId,
  amount: Option[Money], // None = full captured amount
  reason: Option[String],
  idempotencyKey: String
) extends PaymentCommand

final case class CancelPayment(
  paymentId: PaymentId,
  reason: String
) extends PaymentCommand
```

---

## 4. Implementation Phases

### Phase 1: Domain Model and Pure Core

**Duration**: Weeks 1-3 (15 hours)

**Objective**: Build complete domain model with zero infrastructure dependencies.

#### Deliverables

| Deliverable | Description | Acceptance Criteria |
|-------------|-------------|---------------------|
| `Money.scala` | Currency-aware monetary arithmetic | All arithmetic operations correct; overflow protection |
| `Currency.scala` | Supported currency definitions | USD, EUR, GBP, JPY with correct minor units |
| `PaymentId.scala` | Type-safe identifier | Generation, parsing, validation |
| `PaymentStatus.scala` | State machine definition | All valid transitions defined |
| `PaymentEvent.scala` | Event type hierarchy | Complete event set for payment lifecycle |
| `Payment.scala` | Aggregate with event application | Correct state reconstruction from events |
| `PaymentCommand.scala` | Command definitions | All commands with validation rules |
| `MoneySpec.scala` | Property-based tests for Money | Commutativity, associativity, overflow |
| `PaymentSpec.scala` | State machine tests | All transitions, invalid paths rejected |

#### Architecture Decision Records

- **ADR-001**: Aggregate Boundaries
- **ADR-002**: Event Versioning Strategy
- **ADR-003**: Currency Handling Approach

#### Key Code: Command Handler (Pure)

```scala
package payments.core.commands

import cats.data.{EitherT, NonEmptyList}
import cats.effect.Sync
import payments.domain.*

/**
 * Pure command handler - no IO, just business logic.
 * Returns Either[CommandError, NonEmptyList[PaymentEvent]]
 */
object PaymentCommandHandler:

  def handle(
    command: PaymentCommand,
    currentState: Payment
  ): Either[CommandError, NonEmptyList[PaymentEvent]] = command match

    case CreatePayment(id, amount, metadata, _) =>
      if currentState.status != PaymentStatus.Initial then
        Left(CommandError.PaymentAlreadyExists(id))
      else
        Right(NonEmptyList.one(
          PaymentCreated(
            paymentId = id,
            amount = amount.minorUnits,
            currency = amount.currency,
            metadata = metadata,
            timestamp = java.time.Instant.now()
          )
        ))

    case AuthorizePayment(id, requestedAmount, _) =>
      val amount = requestedAmount.getOrElse(currentState.amount)

      for
        _ <- validateStatus(currentState, PaymentStatus.Created)
        _ <- validateAmountNotExceeded(amount, currentState.amount)
      yield NonEmptyList.one(
        PaymentAuthorized(
          paymentId = id,
          authorizedAmount = amount.minorUnits,
          authorizationCode = generateAuthCode(),
          timestamp = java.time.Instant.now()
        )
      )

    case CapturePayment(id, requestedAmount, _) =>
      val amount = requestedAmount.getOrElse(
        Money.fromMinorUnits(
          currentState.authorizedAmount.minorUnits - currentState.capturedAmount.minorUnits,
          currentState.amount.currency
        )
      )

      for
        _ <- validateCapturable(currentState)
        _ <- validateCaptureAmount(amount, currentState)
      yield NonEmptyList.one(
        PaymentCaptured(
          paymentId = id,
          capturedAmount = amount.minorUnits,
          captureReference = generateCaptureRef(),
          timestamp = java.time.Instant.now()
        )
      )

    case RefundPayment(id, requestedAmount, reason, _) =>
      val amount = requestedAmount.getOrElse(
        Money.fromMinorUnits(
          currentState.capturedAmount.minorUnits - currentState.refundedAmount.minorUnits,
          currentState.amount.currency
        )
      )

      for
        _ <- validateRefundable(currentState)
        _ <- validateRefundAmount(amount, currentState)
      yield NonEmptyList.one(
        PaymentRefunded(
          paymentId = id,
          refundedAmount = amount.minorUnits,
          refundReference = generateRefundRef(),
          timestamp = java.time.Instant.now()
        )
      )

    case CancelPayment(id, reason) =>
      for
        _ <- validateCancellable(currentState)
      yield NonEmptyList.one(
        PaymentCancelled(
          paymentId = id,
          reason = reason,
          timestamp = java.time.Instant.now()
        )
      )

  private def validateStatus(
    state: Payment,
    expected: PaymentStatus
  ): Either[CommandError, Unit] =
    if state.status == expected then Right(())
    else Left(CommandError.InvalidStatus(state.status, expected))

  private def validateCapturable(state: Payment): Either[CommandError, Unit] =
    state.status match
      case PaymentStatus.Authorized | PaymentStatus.PartiallyCaptured => Right(())
      case other => Left(CommandError.NotCapturable(other))

  private def validateRefundable(state: Payment): Either[CommandError, Unit] =
    state.status match
      case PaymentStatus.Captured | PaymentStatus.PartiallyRefunded => Right(())
      case other => Left(CommandError.NotRefundable(other))

  private def validateCancellable(state: Payment): Either[CommandError, Unit] =
    state.status match
      case PaymentStatus.Created | PaymentStatus.Authorized => Right(())
      case other => Left(CommandError.NotCancellable(other))

  private def validateAmountNotExceeded(
    requested: Money,
    limit: Money
  ): Either[CommandError, Unit] =
    if requested.minorUnits <= limit.minorUnits then Right(())
    else Left(CommandError.AmountExceedsLimit(requested, limit))

  private def validateCaptureAmount(
    amount: Money,
    state: Payment
  ): Either[CommandError, Unit] =
    val remaining = state.authorizedAmount.minorUnits - state.capturedAmount.minorUnits
    if amount.minorUnits <= remaining then Right(())
    else Left(CommandError.CaptureExceedsAuthorized(amount.minorUnits, remaining))

  private def validateRefundAmount(
    amount: Money,
    state: Payment
  ): Either[CommandError, Unit] =
    val remaining = state.capturedAmount.minorUnits - state.refundedAmount.minorUnits
    if amount.minorUnits <= remaining then Right(())
    else Left(CommandError.RefundExceedsCaptured(amount.minorUnits, remaining))

enum CommandError:
  case PaymentAlreadyExists(id: PaymentId)
  case PaymentNotFound(id: PaymentId)
  case InvalidStatus(current: PaymentStatus, expected: PaymentStatus)
  case NotCapturable(status: PaymentStatus)
  case NotRefundable(status: PaymentStatus)
  case NotCancellable(status: PaymentStatus)
  case AmountExceedsLimit(requested: Money, limit: Money)
  case CaptureExceedsAuthorized(requested: Long, available: Long)
  case RefundExceedsCaptured(requested: Long, available: Long)
  case CurrencyMismatch(expected: Currency, actual: Currency)
```

---

### Phase 2: In-Memory Event Store and HTTP API

**Duration**: Weeks 4-6 (15 hours)

**Objective**: Working end-to-end system with in-memory storage.

#### Deliverables

| Deliverable | Description |
|-------------|-------------|
| `EventStore.scala` | Event store algebra (trait) |
| `InMemoryEventStore.scala` | Ref-based implementation |
| `IdempotencyStore.scala` | Idempotency key storage |
| `PaymentRoutes.scala` | http4s routes for payment API |
| `PaymentService.scala` | Service orchestrating command handling |
| `Main.scala` | Application entry point |
| `docker-compose.yml` | Local development setup |

#### Key Code: Event Store Algebra

```scala
package payments.core.eventstore

import cats.effect.kernel.Ref
import cats.effect.{IO, Sync}
import cats.syntax.all.*
import payments.domain.*

/**
 * Event store algebra - abstraction over event persistence.
 * Implementations: InMemory (Phase 2), Kafka+Cassandra (Phase 3)
 */
trait EventStore[F[_]]:
  /**
   * Append events for an aggregate.
   * Returns the new version number.
   * Fails if expectedVersion doesn't match (optimistic locking).
   */
  def append(
    aggregateId: PaymentId,
    expectedVersion: Long,
    events: List[PaymentEvent]
  ): F[Long]

  /**
   * Load all events for an aggregate.
   */
  def load(aggregateId: PaymentId): F[List[PaymentEvent]]

  /**
   * Load events from a specific version.
   */
  def loadFrom(aggregateId: PaymentId, fromVersion: Long): F[List[PaymentEvent]]

object EventStore:
  /**
   * In-memory implementation using Cats Effect Ref.
   * Thread-safe, suitable for testing and Phase 2.
   */
  def inMemory[F[_]: Sync]: F[EventStore[F]] =
    Ref.of[F, Map[PaymentId, List[(Long, PaymentEvent)]]](Map.empty).map { ref =>
      new EventStore[F]:
        def append(
          aggregateId: PaymentId,
          expectedVersion: Long,
          events: List[PaymentEvent]
        ): F[Long] =
          ref.modify { store =>
            val existing = store.getOrElse(aggregateId, List.empty)
            val currentVersion = existing.lastOption.map(_._1).getOrElse(0L)

            if currentVersion != expectedVersion then
              (store, Left(OptimisticLockException(expectedVersion, currentVersion)))
            else
              val numbered = events.zipWithIndex.map { (e, i) =>
                (currentVersion + i + 1, e)
              }
              val newStore = store.updated(aggregateId, existing ++ numbered)
              val newVersion = currentVersion + events.length
              (newStore, Right(newVersion))
          }.flatMap {
            case Left(err) => Sync[F].raiseError(err)
            case Right(v) => Sync[F].pure(v)
          }

        def load(aggregateId: PaymentId): F[List[PaymentEvent]] =
          ref.get.map(_.getOrElse(aggregateId, List.empty).map(_._2))

        def loadFrom(aggregateId: PaymentId, fromVersion: Long): F[List[PaymentEvent]] =
          ref.get.map(
            _.getOrElse(aggregateId, List.empty)
              .filter(_._1 > fromVersion)
              .map(_._2)
          )
    }

case class OptimisticLockException(
  expected: Long,
  actual: Long
) extends Exception(s"Expected version $expected but was $actual")
```

#### Key Code: HTTP Routes

```scala
package payments.api

import cats.effect.*
import cats.syntax.all.*
import io.circe.generic.auto.*
import io.circe.syntax.*
import org.http4s.*
import org.http4s.circe.*
import org.http4s.dsl.io.*
import payments.core.PaymentService
import payments.domain.*

object PaymentRoutes:

  def routes(service: PaymentService[IO]): HttpRoutes[IO] = HttpRoutes.of[IO] {

    case req @ POST -> Root / "payments" =>
      for
        cmd <- req.as[CreatePaymentRequest]
        result <- service.create(cmd.toCommand)
        response <- result match
          case Right(payment) => Created(payment.asJson)
          case Left(err) => BadRequest(err.asJson)
      yield response

    case req @ POST -> Root / "payments" / UUIDVar(id) / "authorize" =>
      for
        cmd <- req.as[AuthorizeRequest]
        paymentId = PaymentId.unsafeFromString(id.toString)
        result <- service.authorize(paymentId, cmd.amount, cmd.idempotencyKey)
        response <- result match
          case Right(payment) => Ok(payment.asJson)
          case Left(err) => handleError(err)
      yield response

    case req @ POST -> Root / "payments" / UUIDVar(id) / "capture" =>
      for
        cmd <- req.as[CaptureRequest]
        paymentId = PaymentId.unsafeFromString(id.toString)
        result <- service.capture(paymentId, cmd.amount, cmd.idempotencyKey)
        response <- result match
          case Right(payment) => Ok(payment.asJson)
          case Left(err) => handleError(err)
      yield response

    case req @ POST -> Root / "payments" / UUIDVar(id) / "refund" =>
      for
        cmd <- req.as[RefundRequest]
        paymentId = PaymentId.unsafeFromString(id.toString)
        result <- service.refund(paymentId, cmd.amount, cmd.reason, cmd.idempotencyKey)
        response <- result match
          case Right(payment) => Ok(payment.asJson)
          case Left(err) => handleError(err)
      yield response

    case GET -> Root / "payments" / UUIDVar(id) =>
      val paymentId = PaymentId.unsafeFromString(id.toString)
      service.get(paymentId).flatMap {
        case Some(payment) => Ok(payment.asJson)
        case None => NotFound()
      }

    case GET -> Root / "payments" / UUIDVar(id) / "events" =>
      val paymentId = PaymentId.unsafeFromString(id.toString)
      service.getEvents(paymentId).flatMap { events =>
        Ok(events.asJson)
      }
  }

  private def handleError(err: CommandError): IO[Response[IO]] = err match
    case CommandError.PaymentNotFound(_) => NotFound()
    case CommandError.PaymentAlreadyExists(_) => Conflict(err.asJson)
    case _ => BadRequest(err.asJson)
```

---

### Phase 3: Kafka Integration

**Duration**: Weeks 7-9 (15-20 hours)

**Objective**: Replace in-memory storage with Kafka for durable event storage.

#### Deliverables

| Deliverable | Description |
|-------------|-------------|
| `KafkaEventStore.scala` | Kafka-backed event store |
| `ProtobufCodec.scala` | Event serialization with Protobuf |
| `payment_events.proto` | Protobuf schema definitions |
| `CassandraSnapshotStore.scala` | Snapshot storage for replay |
| `EventConsumer.scala` | FS2-Kafka consumer base |
| Schema Registry setup | Confluent Schema Registry config |

#### Key Code: Kafka Event Store

```scala
package payments.infra.kafka

import cats.effect.*
import cats.syntax.all.*
import fs2.kafka.*
import payments.core.eventstore.EventStore
import payments.domain.*

class KafkaEventStore[F[_]: Async](
  producer: KafkaProducer[F, PaymentId, PaymentEvent],
  consumer: KafkaConsumer[F, PaymentId, PaymentEvent],
  snapshotStore: SnapshotStore[F],
  topic: String
) extends EventStore[F]:

  def append(
    aggregateId: PaymentId,
    expectedVersion: Long,
    events: List[PaymentEvent]
  ): F[Long] =
    for
      // Verify version (read current state)
      current <- loadCurrentVersion(aggregateId)
      _ <- if current != expectedVersion
           then Async[F].raiseError(OptimisticLockException(expectedVersion, current))
           else Async[F].unit

      // Produce events transactionally
      records = events.map { event =>
        ProducerRecord(topic, aggregateId, event)
      }
      _ <- producer.produce(ProducerRecords(records)).flatten

      // Update snapshot periodically
      newVersion = expectedVersion + events.length
      _ <- maybeSnapshot(aggregateId, newVersion, events)
    yield newVersion

  def load(aggregateId: PaymentId): F[List[PaymentEvent]] =
    for
      snapshot <- snapshotStore.get(aggregateId)
      fromVersion = snapshot.map(_.version).getOrElse(0L)
      recentEvents <- loadEventsFromKafka(aggregateId, fromVersion)
    yield snapshot.map(_.events).getOrElse(List.empty) ++ recentEvents

  private def loadEventsFromKafka(
    aggregateId: PaymentId,
    fromVersion: Long
  ): F[List[PaymentEvent]] =
    // Implementation uses consumer to read from specific partition
    // based on aggregateId hash
    ???

  private def maybeSnapshot(
    aggregateId: PaymentId,
    version: Long,
    events: List[PaymentEvent]
  ): F[Unit] =
    // Snapshot every 100 events
    if version % 100 == 0 then
      for
        allEvents <- load(aggregateId)
        _ <- snapshotStore.save(aggregateId, Snapshot(version, allEvents))
      yield ()
    else Async[F].unit
```

#### Protobuf Schema

```protobuf
// payment_events.proto
syntax = "proto3";

package payments.events;

option java_package = "payments.infra.proto";

message PaymentEventEnvelope {
  string payment_id = 1;
  int64 version = 2;
  int64 timestamp_millis = 3;

  oneof event {
    PaymentCreated created = 10;
    PaymentAuthorized authorized = 11;
    PaymentCaptured captured = 12;
    PaymentRefunded refunded = 13;
    PaymentFailed failed = 14;
    PaymentCancelled cancelled = 15;
  }
}

message PaymentCreated {
  int64 amount_minor_units = 1;
  string currency = 2;
  map<string, string> metadata = 3;
}

message PaymentAuthorized {
  int64 authorized_amount_minor_units = 1;
  string authorization_code = 2;
}

message PaymentCaptured {
  int64 captured_amount_minor_units = 1;
  string capture_reference = 2;
}

message PaymentRefunded {
  int64 refunded_amount_minor_units = 1;
  string refund_reference = 2;
}

message PaymentFailed {
  string reason = 1;
  optional string error_code = 2;
}

message PaymentCancelled {
  string reason = 1;
}
```

---

### Phase 4: CQRS Projections and Read Models

**Duration**: Weeks 10-12 (15 hours)

**Objective**: Separate read path with optimized PostgreSQL projections.

#### Deliverables

| Deliverable | Description |
|-------------|-------------|
| `ProjectionConsumer.scala` | FS2-Kafka projection consumer |
| `PaymentProjection.scala` | Payment summary projection |
| `AccountBalanceProjection.scala` | Account balance projection |
| `AuditLogProjection.scala` | Audit trail projection |
| `ProjectionManager.scala` | Projection lifecycle management |
| PostgreSQL schema migrations | Flyway migrations for read models |
| `ReadModelRepository.scala` | Query interface for projections |

#### Key Code: Projection Consumer

```scala
package payments.infra.projections

import cats.effect.*
import cats.syntax.all.*
import doobie.*
import doobie.implicits.*
import fs2.*
import fs2.kafka.*
import payments.domain.*

/**
 * Consumes events from Kafka and updates PostgreSQL read models.
 * Ensures exactly-once semantics through idempotent writes.
 */
class ProjectionConsumer[F[_]: Async](
  consumer: KafkaConsumer[F, PaymentId, PaymentEvent],
  xa: Transactor[F],
  projections: List[Projection[F]]
):

  def run: Stream[F, Unit] =
    consumer.stream
      .mapAsync(16) { record =>
        val event = record.record.value
        val offset = record.offset

        for
          // Check if already processed (idempotency)
          processed <- isProcessed(event.paymentId, offset)
          _ <- if processed then Async[F].unit
               else processEvent(event, offset)
        yield ()
      }
      .through(commitBatchWithin(100, 1.second))

  private def processEvent(event: PaymentEvent, offset: CommittableOffset[F]): F[Unit] =
    val updates = projections.traverse_(_.apply(event))
    val markProcessed = recordOffset(event.paymentId, offset)

    // Transactional update
    (updates *> markProcessed).transact(xa)

  private def isProcessed(id: PaymentId, offset: CommittableOffset[F]): F[Boolean] =
    sql"""
      SELECT EXISTS(
        SELECT 1 FROM projection_offsets
        WHERE payment_id = ${id.asString}
        AND kafka_offset >= ${offset.offset.offset}
      )
    """.query[Boolean].unique.transact(xa)

  private def recordOffset(id: PaymentId, offset: CommittableOffset[F]): ConnectionIO[Unit] =
    sql"""
      INSERT INTO projection_offsets (payment_id, kafka_offset, processed_at)
      VALUES (${id.asString}, ${offset.offset.offset}, NOW())
      ON CONFLICT (payment_id) DO UPDATE SET
        kafka_offset = EXCLUDED.kafka_offset,
        processed_at = EXCLUDED.processed_at
    """.update.run.void

trait Projection[F[_]]:
  def apply(event: PaymentEvent): ConnectionIO[Unit]

/**
 * Payment summary projection - denormalized view for queries.
 */
class PaymentSummaryProjection extends Projection[IO]:
  def apply(event: PaymentEvent): ConnectionIO[Unit] = event match
    case PaymentCreated(id, amount, currency, metadata, ts) =>
      sql"""
        INSERT INTO payment_summary (
          payment_id, amount_minor_units, currency, status,
          created_at, updated_at
        ) VALUES (
          ${id.asString}, $amount, ${currency.code}, 'created',
          $ts, $ts
        )
      """.update.run.void

    case PaymentAuthorized(id, amount, _, ts) =>
      sql"""
        UPDATE payment_summary
        SET status = 'authorized',
            authorized_amount = $amount,
            updated_at = $ts
        WHERE payment_id = ${id.asString}
      """.update.run.void

    case PaymentCaptured(id, amount, _, ts) =>
      sql"""
        UPDATE payment_summary
        SET captured_amount = captured_amount + $amount,
            status = CASE
              WHEN captured_amount + $amount >= authorized_amount THEN 'captured'
              ELSE 'partially_captured'
            END,
            updated_at = $ts
        WHERE payment_id = ${id.asString}
      """.update.run.void

    case PaymentRefunded(id, amount, _, ts) =>
      sql"""
        UPDATE payment_summary
        SET refunded_amount = refunded_amount + $amount,
            status = CASE
              WHEN refunded_amount + $amount >= captured_amount THEN 'refunded'
              ELSE 'partially_refunded'
            END,
            updated_at = $ts
        WHERE payment_id = ${id.asString}
      """.update.run.void

    case PaymentFailed(id, reason, _, ts) =>
      sql"""
        UPDATE payment_summary
        SET status = 'failed', failure_reason = $reason, updated_at = $ts
        WHERE payment_id = ${id.asString}
      """.update.run.void

    case PaymentCancelled(id, reason, ts) =>
      sql"""
        UPDATE payment_summary
        SET status = 'cancelled', cancellation_reason = $reason, updated_at = $ts
        WHERE payment_id = ${id.asString}
      """.update.run.void
```

#### PostgreSQL Schema

```sql
-- V1__create_read_models.sql

CREATE TABLE payment_summary (
    payment_id UUID PRIMARY KEY,
    amount_minor_units BIGINT NOT NULL,
    currency VARCHAR(3) NOT NULL,
    status VARCHAR(32) NOT NULL,
    authorized_amount BIGINT DEFAULT 0,
    captured_amount BIGINT DEFAULT 0,
    refunded_amount BIGINT DEFAULT 0,
    failure_reason TEXT,
    cancellation_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_payment_summary_status ON payment_summary(status);
CREATE INDEX idx_payment_summary_created ON payment_summary(created_at);

CREATE TABLE projection_offsets (
    payment_id UUID PRIMARY KEY,
    kafka_offset BIGINT NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    payment_id UUID NOT NULL,
    event_type VARCHAR(64) NOT NULL,
    event_data JSONB NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL,
    FOREIGN KEY (payment_id) REFERENCES payment_summary(payment_id)
);

CREATE INDEX idx_audit_log_payment ON audit_log(payment_id);
CREATE INDEX idx_audit_log_occurred ON audit_log(occurred_at);
```

---

### Phase 5: Saga Orchestration

**Duration**: Weeks 13-16 (15-20 hours)

**Objective**: Multi-step payment flows with compensation logic.

#### Deliverables

| Deliverable | Description |
|-------------|-------------|
| `Saga.scala` | Type-safe saga state machine |
| `SagaStep.scala` | Individual saga step definition |
| `SagaExecutor.scala` | Saga execution engine |
| `SagaStore.scala` | Saga persistence |
| `PaymentSaga.scala` | Payment-specific saga implementation |
| `CompensationHandler.scala` | Rollback/compensation logic |
| `DeadLetterQueue.scala` | Failed saga handling |

#### Key Code: Saga Framework

```scala
package payments.core.saga

import cats.effect.*
import cats.syntax.all.*
import payments.domain.*

/**
 * Type-safe saga definition.
 * A saga is a sequence of steps with compensation logic.
 */
sealed trait SagaState
object SagaState:
  case object Pending extends SagaState
  case object Running extends SagaState
  case object Completed extends SagaState
  case object Compensating extends SagaState
  case object Failed extends SagaState

case class Saga[F[_], A](
  id: SagaId,
  name: String,
  steps: List[SagaStep[F, ?]],
  currentStep: Int,
  state: SagaState,
  result: Option[Either[SagaError, A]],
  compensatedSteps: List[Int]
)

case class SagaStep[F[_], A](
  name: String,
  execute: F[A],
  compensate: A => F[Unit],
  retryPolicy: RetryPolicy
)

case class RetryPolicy(
  maxAttempts: Int,
  initialDelay: FiniteDuration,
  maxDelay: FiniteDuration,
  backoffMultiplier: Double
)

/**
 * Saga executor - runs sagas with compensation on failure.
 */
class SagaExecutor[F[_]: Async](
  store: SagaStore[F],
  dlq: DeadLetterQueue[F]
):

  def execute[A](saga: Saga[F, A]): F[Either[SagaError, A]] =
    for
      _ <- store.save(saga.copy(state = SagaState.Running))
      result <- runSteps(saga, 0, List.empty)
      _ <- store.save(saga.copy(
        state = if result.isRight then SagaState.Completed else SagaState.Failed,
        result = Some(result)
      ))
    yield result

  private def runSteps[A](
    saga: Saga[F, A],
    stepIndex: Int,
    completedResults: List[Any]
  ): F[Either[SagaError, A]] =
    if stepIndex >= saga.steps.length then
      // All steps completed
      Async[F].pure(Right(completedResults.last.asInstanceOf[A]))
    else
      val step = saga.steps(stepIndex)
      executeWithRetry(step).attempt.flatMap {
        case Right(result) =>
          // Step succeeded, continue
          store.updateStep(saga.id, stepIndex, StepStatus.Completed) *>
            runSteps(saga, stepIndex + 1, completedResults :+ result)

        case Left(error) =>
          // Step failed, compensate
          store.updateStep(saga.id, stepIndex, StepStatus.Failed) *>
            compensate(saga, stepIndex - 1, completedResults).as(
              Left(SagaError.StepFailed(step.name, error))
            )
      }

  private def compensate[A](
    saga: Saga[F, A],
    fromStep: Int,
    results: List[Any]
  ): F[Unit] =
    if fromStep < 0 then Async[F].unit
    else
      val step = saga.steps(fromStep)
      val result = results(fromStep)

      for
        _ <- store.save(saga.copy(state = SagaState.Compensating))
        _ <- step.compensate(result.asInstanceOf[step.type]).handleErrorWith { err =>
          // Compensation failed - send to DLQ
          dlq.send(FailedCompensation(saga.id, fromStep, err))
        }
        _ <- store.updateStep(saga.id, fromStep, StepStatus.Compensated)
        _ <- compensate(saga, fromStep - 1, results)
      yield ()

  private def executeWithRetry[A](step: SagaStep[F, A]): F[A] =
    // Implement retry with exponential backoff
    ???
```

#### Key Code: Payment Saga

```scala
package payments.core.saga

import cats.effect.*
import payments.domain.*
import payments.core.PaymentService

/**
 * Complete payment saga: Create -> Authorize -> Capture
 * With compensation at each step.
 */
object PaymentSaga:

  def create[F[_]: Async](
    paymentService: PaymentService[F],
    request: PaymentRequest
  ): Saga[F, Payment] =
    Saga(
      id = SagaId.generate,
      name = "payment-saga",
      steps = List(
        // Step 1: Create payment
        SagaStep(
          name = "create-payment",
          execute = paymentService.create(CreatePayment(
            paymentId = PaymentId.generate,
            amount = request.amount,
            metadata = request.metadata,
            idempotencyKey = request.idempotencyKey
          )),
          compensate = (payment: Payment) =>
            paymentService.cancel(payment.id, "Saga compensation"),
          retryPolicy = RetryPolicy(3, 100.millis, 1.second, 2.0)
        ),

        // Step 2: Authorize payment
        SagaStep(
          name = "authorize-payment",
          execute = paymentService.authorize(
            request.paymentId,
            None, // Full amount
            s"${request.idempotencyKey}-auth"
          ),
          compensate = (payment: Payment) =>
            paymentService.voidAuthorization(payment.id),
          retryPolicy = RetryPolicy(3, 100.millis, 1.second, 2.0)
        ),

        // Step 3: Capture payment
        SagaStep(
          name = "capture-payment",
          execute = paymentService.capture(
            request.paymentId,
            None, // Full authorized amount
            s"${request.idempotencyKey}-capture"
          ),
          compensate = (payment: Payment) =>
            paymentService.refund(payment.id, None, Some("Saga compensation"),
              s"${request.idempotencyKey}-refund"),
          retryPolicy = RetryPolicy(5, 200.millis, 5.seconds, 2.0)
        )
      ),
      currentStep = 0,
      state = SagaState.Pending,
      result = None,
      compensatedSteps = List.empty
    )
```

---

### Phase 6: Production Hardening

**Duration**: Weeks 17-20+ (15-20 hours)

**Objective**: Production-ready deployment with observability.

#### Deliverables

| Deliverable | Description |
|-------------|-------------|
| `RateLimiter.scala` | Token bucket rate limiting |
| `CircuitBreaker.scala` | Resilience for external calls |
| Metrics integration | Prometheus/Grafana setup |
| Distributed tracing | OpenTelemetry integration |
| Kubernetes manifests | Deployment configurations |
| Load testing | k6 performance tests |
| Runbook | Operations documentation |

#### Key Code: Rate Limiter

```scala
package payments.api.middleware

import cats.effect.*
import cats.effect.std.Semaphore
import cats.syntax.all.*
import org.http4s.*
import scala.concurrent.duration.*

/**
 * Token bucket rate limiter.
 * Configurable per-client or global limiting.
 */
class RateLimiter[F[_]: Temporal](
  tokensPerSecond: Int,
  bucketSize: Int
) {
  private case class Bucket(tokens: Double, lastRefill: Long)

  def middleware(
    getBucketKey: Request[F] => String
  ): HttpMiddleware[F] = { routes =>
    Ref.of[F, Map[String, Bucket]](Map.empty).flatMap { bucketsRef =>
      HttpRoutes { request =>
        val key = getBucketKey(request)

        OptionT(
          bucketsRef.modify { buckets =>
            val now = System.currentTimeMillis()
            val bucket = buckets.getOrElse(key, Bucket(bucketSize.toDouble, now))

            // Refill tokens based on time elapsed
            val elapsed = (now - bucket.lastRefill) / 1000.0
            val refilled = math.min(
              bucketSize.toDouble,
              bucket.tokens + elapsed * tokensPerSecond
            )

            if refilled >= 1.0 then
              // Allow request, consume token
              val newBucket = Bucket(refilled - 1.0, now)
              (buckets.updated(key, newBucket), Some(routes(request)))
            else
              // Rate limited
              val retryAfter = ((1.0 - refilled) / tokensPerSecond * 1000).toLong
              (buckets.updated(key, bucket.copy(lastRefill = now)),
               Some(OptionT.pure[F](
                 Response[F](Status.TooManyRequests)
                   .withHeaders(Header.Raw(ci"Retry-After", retryAfter.toString))
               )))
          }.flatMap(_.sequence)
        ).flatten
      }
    }
  }
}
```

---

## 5. Testing Strategy

### 5.1 Testing Pyramid

```
                    ┌───────────────┐
                    │   E2E Tests   │  <- Few, slow, high confidence
                    │ (Testcontainers│
                    │  full stack)  │
                    └───────┬───────┘
                            │
               ┌────────────┴────────────┐
               │   Integration Tests     │  <- Moderate count
               │  (Single service with   │
               │   real dependencies)    │
               └────────────┬────────────┘
                            │
          ┌─────────────────┴─────────────────┐
          │         Property Tests            │  <- Many, fast
          │  (Domain logic, state machines,   │
          │   serialization round-trips)      │
          └─────────────────┬─────────────────┘
                            │
    ┌───────────────────────┴───────────────────────┐
    │              Unit Tests                       │  <- Most, fastest
    │  (Pure functions, command validation,         │
    │   event application, business rules)          │
    └───────────────────────────────────────────────┘
```

### 5.2 Key Test Scenarios

```scala
// Property: Event replay is deterministic
property("event replay produces same state") {
  forAll(genEventSequence) { events =>
    val state1 = events.foldLeft(Payment.empty)(Payment.apply)
    val state2 = events.foldLeft(Payment.empty)(Payment.apply)
    state1 == state2
  }
}

// Property: Serialization round-trips
property("protobuf round-trip") {
  forAll(genPaymentEvent) { event =>
    decode(encode(event)) == Right(event)
  }
}

// Scenario: Duplicate command handling
test("duplicate authorize returns same result") {
  for
    id <- createPayment(amount = 100)
    r1 <- authorize(id, idempotencyKey = "key-1")
    r2 <- authorize(id, idempotencyKey = "key-1")
  yield expect(r1 == r2)
}

// Scenario: Saga compensation
test("failed capture triggers auth reversal") {
  for
    _      <- givenExternalCaptureWillFail()
    id     <- createPayment(100)
    _      <- authorize(id)
    result <- capture(id).attempt
    state  <- getPaymentState(id)
  yield expect(result.isLeft && state == PaymentState.AuthReversed)
}
```

---

## 6. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Kafka operational complexity | High | High | Start single partition; fallback to Pulsar |
| Event schema evolution issues | Medium | High | Protobuf with field numbers; never remove fields |
| Scope creep on production features | High | Medium | Define "portfolio ready" vs "production ready" |
| Time underestimation | High | Medium | Each phase has buffer; cut scope first |
| Cassandra operational burden | Medium | Medium | Start with PostgreSQL for events; migrate later |
| Testing distributed scenarios | Medium | High | Invest in Testcontainers early |

---

## 7. Definition of Done

### Per Phase

- [ ] All deliverables implemented and tested
- [ ] Integration tests passing
- [ ] Code reviewed and documented
- [ ] ADRs written for key decisions
- [ ] Docker Compose runs successfully

### Project Complete

- [ ] Full payment lifecycle works end-to-end
- [ ] Idempotency verified under concurrent load
- [ ] Saga compensation tested with fault injection
- [ ] Performance targets met (100 req/s sustained)
- [ ] Kubernetes deployment successful
- [ ] Runbook documentation complete

---

## 8. Appendix

### A. Project Structure

```
payment-service/
├── build.sbt
├── project/
│   ├── build.properties
│   └── plugins.sbt
├── docker-compose.yml
├── domain/
│   └── src/
│       ├── main/scala/payments/domain/
│       │   ├── Money.scala
│       │   ├── Currency.scala
│       │   ├── PaymentId.scala
│       │   ├── PaymentStatus.scala
│       │   ├── PaymentEvent.scala
│       │   ├── PaymentCommand.scala
│       │   └── Payment.scala
│       └── test/scala/payments/domain/
├── core/
│   └── src/
│       ├── main/scala/payments/core/
│       │   ├── commands/
│       │   ├── eventstore/
│       │   └── projections/
│       └── test/scala/payments/core/
├── infra/
│   └── src/main/scala/payments/infra/
│       ├── kafka/
│       ├── cassandra/
│       └── postgres/
├── api/
│   └── src/main/scala/payments/api/
└── app/
    └── src/main/scala/payments/
```

### B. Dependencies (build.sbt)

```scala
ThisBuild / scalaVersion := "3.3.1"
ThisBuild / organization := "com.payments"

val catsEffectVersion = "3.5.2"
val http4sVersion = "0.23.24"
val fs2KafkaVersion = "3.2.0"
val doobieVersion = "1.0.0-RC4"
val circeVersion = "0.14.6"
val weaverVersion = "0.8.3"

lazy val domain = project
  .settings(
    libraryDependencies ++= Seq(
      "org.typelevel" %% "cats-core" % "2.10.0",
      "com.disneystreaming" %% "weaver-cats" % weaverVersion % Test,
      "com.disneystreaming" %% "weaver-scalacheck" % weaverVersion % Test,
    )
  )

lazy val core = project
  .dependsOn(domain)
  .settings(
    libraryDependencies ++= Seq(
      "org.typelevel" %% "cats-effect" % catsEffectVersion,
    )
  )

lazy val infra = project
  .dependsOn(core)
  .settings(
    libraryDependencies ++= Seq(
      "com.github.fd4s" %% "fs2-kafka" % fs2KafkaVersion,
      "org.tpolecat" %% "doobie-core" % doobieVersion,
      "org.tpolecat" %% "doobie-postgres" % doobieVersion,
      "org.tpolecat" %% "doobie-hikari" % doobieVersion,
    )
  )

lazy val api = project
  .dependsOn(core, infra)
  .settings(
    libraryDependencies ++= Seq(
      "org.http4s" %% "http4s-ember-server" % http4sVersion,
      "org.http4s" %% "http4s-circe" % http4sVersion,
      "org.http4s" %% "http4s-dsl" % http4sVersion,
      "io.circe" %% "circe-generic" % circeVersion,
    )
  )

lazy val app = project
  .dependsOn(api)
  .settings(
    Compile / mainClass := Some("payments.Main")
  )
```

### C. References

- [Event Sourcing Pattern](https://martinfowler.com/eaaDev/EventSourcing.html)
- [CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)
- [Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [Cats Effect Documentation](https://typelevel.org/cats-effect/)
- [http4s Documentation](https://http4s.org/)
- [FS2 Kafka](https://fd4s.github.io/fs2-kafka/)
