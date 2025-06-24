# Implementation Guide

This guide outlines a step-by-step approach to building the application according to the established architecture. By partitioning the work into sequential stages, GitHub Copilot (or a developer) can focus on one layer at a time and avoid attempting to generate too much at once. The steps below are generic and can be applied to any service following this architecture.

## 1. Set Up Solution Structure

Initialize a new .NET 9 solution and create the baseline projects.

- Create a solution (e.g. MyApp.sln) with a src directory and a test directory.
- Within src, create class library projects for MyApp.Shared, MyApp.Domain, MyApp.Application, MyApp.Infrastructure.Data, MyApp.Infrastructure.External, and an ASP.NET Core MyApp.Api project (using Minimal API). Each project should target .NET 9 and use the naming conventions from the architecture.
- Establish the project references so that each project only depends on the lower-level projects as defined in the architecture (e.g. MyApp.Api references all other layers, Infrastructure depends on Application/Domain/Shared, etc.). Ensure no reverse dependencies (dependency flow must be inward-only).
- Add corresponding test projects under test/ (e.g. MyApp.UnitTests and MyApp.IntegrationTests), and reference the appropriate src projects from these test projects.

## 2. Implement Shared Layer

Populate the MyApp.Shared project with cross-cutting primitives and utilities.

- Add guard clauses or validation helper classes (for example, an ArgumentGuard static class to validate inputs).
- Define common value object types if needed (e.g. a `Result<T>` or `Maybe<T>` struct to represent operation outcomes).
- Add basic abstractions used across layers, such as interfaces for common services or types (IEntity, IAggregateRoot, IDateTimeProvider, etc.).
- Keep this layer free of any domain-specific logic.

## 3. Implement Domain Layer

Using the specifications in `docs/{MyApp}.specs.md`, create the domain model in MyApp.Domain.

- Define the main entity classes and value objects for the core domain (e.g. the key business entities with their properties and invariants). Ensure that constructors or methods enforce business rules (invariants) where appropriate.
- If any domain logic requires external information (for example, currency conversion or time-based calculations), define domain-level interfaces for those services (e.g. ICurrencyConverter) so the domain layer stays infrastructure-agnostic.
- Create domain event classes if the domain uses event-driven concepts (optional, depending on the domain requirements).
- The Domain project should not depend on Application or Infrastructure; it only uses the Shared kernel and its own types.

## 4. Implement Application Layer

In MyApp.Application, build the use-case logic and contracts.

- Define data transfer objects and service contracts under an appropriate namespace (e.g. MyApp.Application.Contracts). These include input/output models for operations. Separate internal DTOs from external API contracts if needed.
- For each use case (command or query) described in `docs/{MyApp}.specs.md`, create a class to represent that request (e.g. CreateOrderCommand, GetOrderQuery) along with a handler class (e.g. CreateOrderHandler) that contains the orchestration logic. Use MediatR’s interfaces (IRequest, IRequestHandler, etc.) to structure these handlers.
- Implement the business logic in the handlers by coordinating domain entities and calling necessary interfaces. Each handler should focus on a single use-case. If using CQRS via MediatR, add package references for MediatR in this project (to define requests and handlers) but do not configure MediatR here (that will be done in the API layer).
- Define any interfaces in the Application layer that represent infrastructure dependencies required by the use-cases (e.g. IOrderRepository, IEmailSender, INotificationService). These interfaces abstract the persistence or external calls needed, to be implemented in the Infrastructure layer.
- Optionally, implement request validation logic for commands/queries (e.g. using FluentValidation classes) to ensure incoming data meets business criteria.

## 5. Implement Infrastructure.Data Layer

In MyApp.Infrastructure.Data, provide concrete data-access implementations.

- Add a reference to a suitable EF Core database provider (for example, Microsoft.EntityFrameworkCore.SqlServer if using SQL Server) and define a DbContext class (e.g. MyAppDbContext) representing the database. Include `DbSet<TEntity>` properties for each aggregate or entity that will be persisted.
- Implement repository classes or data access services corresponding to the interfaces defined in the Application layer (e.g. OrderRepository implementing IOrderRepository). Use the chosen data access technology (EF Core, Dapper, etc.) to interact with the database. For example, the repository methods will use MyAppDbContext to query or save data.
- If using EF Core (if none specified in `docs/{MyApp}.specs.md`, default to using EF Core), you can also set up entity type configurations or migrations in this project. Keep all database-related logic confined to the Infrastructure.Data layer.
Ensure that Infrastructure.Data classes rely only on the abstractions from Application and do not contain any business logic themselves. This layer’s code should strictly handle persistence concerns.

## 6. Implement Infrastructure.External Layer

In MyApp.Infrastructure.External, implement outbound integration services needed by the application.

- For each external dependency (see `docs/{MyApp}.specs.md` External Integrations section), create a service/client class that implements the corresponding Application layer interface. For example, if Application defines IEmailSender, implement SmtpEmailSender or SendGridEmailSender here.
- Utilize HttpClientFactory for calling external HTTP APIs. Configure resilient HTTP clients if needed (using policies like retries and circuit breakers) – for instance, use the Microsoft.Extensions.Http.Resilience package to add resilience policies. The actual HttpClient configuration (base address, policies) can be defined here or in API startup as appropriate.
- Implement other external connectivity (e.g. message bus publishers, file storage clients) using relevant SDKs or libraries in this layer. For example, if the application needs to publish events, create a MessageBusPublisher that implements an IMessagePublisher interface from Application and uses a library (like a RabbitMQ client) to send messages.
- Like the data layer, keep external integration code free of any domain decisions. It should translate between Application layer calls and external system interactions, nothing more.

## 7. Implement API Layer

Set up the MyApp.Api project to expose the functionality via HTTP endpoints.

- In Program.cs, configure the WebApplication (Minimal API). Map HTTP endpoints to the Application layer’s commands and queries using Minimal API routes. For example, `app.MapPost("/orders", async (CreateOrderCommand cmd, IMediator mediator) => await mediator.Send(cmd));` defines a POST endpoint that sends a CreateOrderCommand to MediatR. Each endpoint should translate HTTP input (e.g. JSON body or query params) into a command/query object and send it via MediatR.
- Configure the dependency injection container with all services. Register Application and Domain layers (e.g. `builder.Services.AddMediatR(typeof(MyApp.Application.AssemblyMarker))` to add MediatR handlers). Register FluentValidation validators (e.g. using `AddValidatorsFromAssemblyContaining<SomeCommandValidator>()`). Register the Infrastructure implementations for the interfaces (e.g. IOrderRepository → OrderRepository) so that handlers can get their dependencies.
- Register any HTTP clients or other external services in the API startup. For instance, if IExternalApiClient is implemented by ExternalApiClient in Infrastructure.External, use `builder.Services.AddHttpClient<IExternalApiClient, ExternalApiClient>(...)` to configure it. The resilience policies defined in Infrastructure.External should be applied here when adding the HttpClient.
- Apply cross-cutting middleware and configurations as needed: global exception handling, request logging, and correlation ID middleware (for traceability) as noted in the architecture. In Minimal API, this could involve adding custom middleware or using built-in features (e.g. app.UseExceptionHandler(...) for centralized error handling).
- Enable OpenAPI description generation if required. Since we are using Minimal API, ensure builder.Services.AddEndpointsApiExplorer() is called, and optionally use builder.Services.AddSwaggerGen() or Microsoft.AspNetCore.OpenApi to produce an OpenAPI specification (even if no Swagger UI). This helps in documenting the API endpoints.
- Finally, build and run the application to ensure that all pieces are wired together correctly. The API project should reference all other projects (Domain, Application, Shared, Infrastructure.Data, Infrastructure.External) and the application should start up without errors.

## 8. Testing and Verification

Implement tests to validate the application’s behavior according to requirements.

- Write unit tests in MyApp.UnitTests for the Domain and Application layer logic. Use a testing framework like xUnit and leverage Moq (or similar) for creating dummy implementations of interfaces. For example, test that domain entities enforce rules (e.g. cannot create an order with negative quantity) and that Application handlers invoke the right repository methods or domain operations. Use mocking to isolate the unit of test.
- Write integration tests in MyApp.IntegrationTests to cover end-to-end scenarios. This may involve spinning up real infrastructure components or realistic substitutes: for example, use Testcontainers to start a temporary database for testing repository classes, and use an HTTP stub (like WireMock) to simulate external API responses. You can also use the actual MyApp.Api with an in-memory test server (via WebApplicationFactory in ASP.NET Core) to test the HTTP endpoints and the wiring of the whole system.
- Ensure tests cover both successful cases and failure cases (e.g. validating error responses for bad input). Integration tests should clean up or isolate side effects (for instance, using an in-memory database or resetting state between runs).
- Run the full test suite to verify that all layers work together as expected. If any test fails, debug and fix issues while maintaining the prescribed layer boundaries. The goal is a green test suite indicating the application meets the specified use-cases and architecture constraints.
