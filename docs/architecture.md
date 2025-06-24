# Architecture and Stack Patterns

## 1. Purpose

This document provides a repeatable, opinionated blueprint for building service-oriented .NET 9 applications. This architecture fixes layer boundaries, project naming conventions, and dependency flow so every new service starts from the same solid foundation—yet remains free to plug in domain-specific code.

## 2. Solution Layout

In the structure below (and throughout this document), **MyApp** is used as a placeholder for the solution name. Replace **MyApp** with the actual project name when applying this template.

### 2.1 Diagram

- MyApp.sln
  - docs/
    - Architectural, implementation, and specification documents (architecture.md, implementation.md, {MyApp}.solution.md, and supporting documentation.)
  - src/
    - MyApp.Shared/
      - Shared primitives (lowest layer)
    - MyApp.Domain/
      - Domain model and business rules
    - MyApp.Application/
      - Use-cases, commands, queries, and DTO/contracts
    - MyApp.Infrastructure.Data/
      - Persistence adapters (EF Core, Dapper, etc.)
    - MyApp.Infrastructure.External/
      - Outbound integrations (HTTP, messaging, etc.)
    - MyApp.API/
      - Hosting layer - *Minimal API only*
  - test/
    - MyApp.UnitTests/
      - Pure unit tests (Moq). Folders mirror src folder structure.
    - MyApp.IntegrationTests/
      - Integration/contract tests

### 2.2 Project-to-Project Dependencies

Dependencies flow inward only. Higher-level project may depend on lower-level ones, never the opposite, keeping the core pure and testable.

- MyApp.Shared
  - References: (none)
- MyApp.Domain
  - References: MyApp.Shared
- MyApp.Application
  - References: MyApp.Domain, MyApp.Shared
- MyApp.Infrastructure.Data
  - References: MyApp.Application, MyApp.Domain, MyApp.Shared
- MyApp.Infrastructure.External
  - References: MyApp.Application, MyApp.Domain, MyApp.Shared
- MyApp.Api
  - References: MyApp.Application, MyApp.Domain, MyApp.Shared, MyApp.Infrastructure.*

## 3. Layer Responsibilities

### 3.1 Shared *(MyApp.Shared)*

Pure, reusable building-blocks with no domain knowledge.

- Guard/validation helpers
- Primitive value objects (e.g. Result, Maybe, PagedResult)
- Cross-cutting abstractions: IEntity, IAggregateRoot, IDateTimeProvider, etc.

### 3.2 Domain *(MyApp.Domain)*

- Entities, value objects, domain events, invariants
- Domain services (encapsulate business logic that doesn’t fit in an entity)
- Interfaces for domain-required services when those abstractions logically belong in the model (e.g. ICurrencyConverter)

### 3.3 Application *(MyApp.Application)*

- Use-case orchestration (command/query handlers, typically via MediatR)
- **DTO & Contract definitions** (in MyApp.Application.Contracts namespace):
  - *Internal DTOs* – command/query shapes internal to the service
  - *Public contracts* – request/response types exposed to other services
  - (These remain lightweight and reference only MyApp.Shared.)
- Interfaces that describe infrastructure needs (e.g. INoteClient, IUnitOfWork, IEmailSender)

### 3.4 Infrastructure *(MyApp.Infrastructure.)*

- **Infrastructure.Data** – persistence technologies (EF Core, migrations, Dapper, Cosmos DB, etc.)
- **Infrastructure.External** – outbound HTTP clients, message-bus publishers, file-storage adapters
- Implements interfaces declared in Application. No domain logic.

### 3.5 MyApp.API *(MyApp.API)*

- Hosting bootstrap (Program.cs) using Minimal API exclusively
- Endpoint definitions map HTTP requests → Application layer (via MediatR)
- Middleware includes exception handling, logging, correlation IDs
- Registers MediatR, FluentValidation validators, and OpenAPI generator for this service

## 4. Testing Strategy

- Unit tests (MyApp.UnitTests) — focus on Domain and Application layers, using Moq for test doubles.
- Integration tests (MyApp.IntegrationTests) — spin up real Infrastructure adapters (e.g. Testcontainers for databases, WireMock for HTTP) to verify end-to-end behavior.
- All tests live under the single test/ directory for straightforward discovery.

## 5. Package Reference Checklist

- **Resilience & Retries**: Use `Microsoft.Extensions.Http.Resilience` for robust HTTP calls.
- **Mediator Pattern**: Use MediatR (plus `MediatR.Extensions.Microsoft.DependencyInjection`) for internal request/response dispatching (registered only in MyApp.Api).
- **Validation**: Use FluentValidation (with `FluentValidation.DependencyInjectionExtensions`) for input validation logic.
- **OpenAPI** (no UI): Use `Microsoft.AspNetCore.OpenApi` to generate OpenAPI descriptions for the Minimal API.
- **Database Provider**: Include `Microsoft.EntityFrameworkCore.<Provider>` (e.g. SqlServer, PostgreSQL) as appropriate for data access.
- **Testing Mocks**: Use Moq for creating mock implementations in unit tests.

## 6. Cross-cutting Configuration

- **MediatR Registration**: Register MediatR in the API layer during startup; the Application (and Domain) layer should only contain handler classes, not the registration logic.
- **HTTP Clients with Resilience**: Define HttpClientFactory clients with resilience policies (using the resilience library above) in the Infrastructure projects, but register these HTTP clients in the API startup.
- **Date/Time Provider**: Use an abstraction (e.g. IDateTimeProvider in Shared) for obtaining current time; provide an implementation in Infrastructure (e.g. SystemDateTimeProvider) and inject it wherever date/time is needed.

## 7. Glossary

- **DTO (Data Transfer Object)**: A structure for transferring data with no business logic (often used for external communication or between layers).
- **Shared**: The most stable, dependency-free primitives consumed by every other project.
- **Domain**: The rich core model expressing business rules and invariants.
- **Application**: The orchestration layer handling commands and queries, enforcing transactional boundaries and coordinating work across domain and infrastructure.
