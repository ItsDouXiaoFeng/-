# Event-Sourced Payment Service

A production-grade payment processing system built with CQRS (Command Query Responsibility Segregation) and Event Sourcing patterns. This project demonstrates distributed systems design, type-safe functional programming in Scala 3, and enterprise-grade payment infrastructure.

## Overview

This service implements a complete payment lifecycle management system using event sourcing as the primary data persistence pattern. Every state change is captured as an immutable event, providing full auditability, temporal queries, and the ability to reconstruct system state at any point in time.

### Key Features

- **Event Sourcing**: All state changes persisted as immutable events
- **CQRS Architecture**: Separated command and query responsibilities for optimal read/write patterns
- **Saga Orchestration**: Multi-step payment flows with compensation logic for failure recovery
- **Idempotency**: Safe retry semantics for all operations
- **Type-Safe Domain Model**: Leveraging Scala 3 ADTs and opaque types for compile-time correctness

## Architecture

```
                              API Gateway
                    (Rate Limiting, Auth, Idempotency)
                                  |
                    +-------------+-------------+
                    |                           |
              Command Side                 Query Side
                    |                           |
             Command Handlers          Read Model Service
             (Validation, Events)     (Query Projections)
                    |                           |
                    v                           ^
              Event Store                 PostgreSQL
           (Kafka + Cassandra)          (Read Models)
                    |                           |
                    +---> Projections ----------+
                              (FS2-Kafka)
```

## Technology Stack

| Component | Technology |
|-----------|------------|
| Language | Scala 3 |
| Effect System | Cats Effect 3 |
| HTTP | http4s |
| Streaming | FS2-Kafka |
| Event Log | Apache Kafka |
| Event Store | Cassandra |
| Read Models | PostgreSQL |
| Serialization | Protocol Buffers |
| Testing | Weaver + Testcontainers |
| Deployment | Docker / Kubernetes |

## Project Structure

```
payment-service/
├── domain/          # Pure domain model, zero dependencies
├── core/            # Business logic, command handlers
├── infra/           # Kafka, Cassandra, PostgreSQL implementations
├── api/             # HTTP routes and middleware
├── app/             # Application wiring and entry point
└── docs/            # Architecture documentation
```

## Getting Started

### Prerequisites

- JDK 17+
- sbt 1.9+
- Docker and Docker Compose

### Running Locally

```bash
# Start infrastructure
docker-compose up -d

# Run the service
sbt "project app" run

# Run tests
sbt test
```

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/payments` | Create payment intent |
| POST | `/payments/:id/authorize` | Authorize payment |
| POST | `/payments/:id/capture` | Capture authorized payment |
| POST | `/payments/:id/refund` | Refund captured payment |
| GET | `/payments/:id` | Query payment status |
| GET | `/payments/:id/events` | Get payment event history |

## Documentation

Detailed documentation is available in the [docs](./docs) folder:

- [Implementation Plan](./docs/IMPLEMENTATION_PLAN.md)

## License

MIT License

## Contributing

This project is currently in active development. See the GitHub Issues for planned features and implementation progress.
