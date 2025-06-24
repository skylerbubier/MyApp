# GitHub Copilot Instructions

## Architecture and Implementation Guidelines

- **Use the Documentation as Context**
  - Always consult the design documents (especially docs/architecture.md and docs/{solution}.md) before writing any code. These files describe the required project structure, domain models, and use-cases that you must adhere to.
- **Follow Layered Architecture**
  - Respect the defined layer boundaries at all times. Higher-level layers may depend on lower-level layers, never the reverse. For example, Domain and Application projects must not depend on Infrastructure projects. Place code in the appropriate project based on its responsibility (domain logic in Domain, data access in Infrastructure.Data, etc.), and avoid leaking logic into the wrong layer.
- **Work Step-by-Step**
  - Implement the application in the incremental steps outlined in docs/implementation.md. Tackle one step at a time (e.g. first set up the solution and projects, then implement the Shared layer, then Domain, and so on), and ensure each layer is working correctly before moving to the next. Do not attempt to generate the entire solution in one go.
- **Minimal API for Endpoints**
  - Use the Minimal API style in the API layer (no MVC controllers). Define all HTTP endpoints in the MyApp.Api project’s Program.cs, mapping requests to Application layer commands/queries via MediatR. Adhere to one endpoint per use-case, and follow RESTful conventions for routes and HTTP verbs where possible.
- **Dependency Injection in API**
  - Configure all services and dependencies in the API project (the composition root). For example, register MediatR and FluentValidation in Program.cs (as per the architecture, MediatR should be registered only in the API layer). Also register the concrete Infrastructure implementations for the interfaces defined in Application (e.g. add the repository, external service clients, DbContext, etc. to the DI container in startup).
- **Use Recommended Libraries & Patterns**
  - Follow the established tech stack and patterns from the architecture. For instance, use MediatR for command/query handlers, FluentValidation for input validation, Microsoft.Extensions.Http.Resilience for resilient outbound HTTP calls, etc.. Utilize these libraries in the manner prescribed (e.g. MediatR for CQRS, one handler per command/query; validators for request models; resilient HttpClient policies for external calls).
- **Ensure Correct Solution & Projects**
  - All generated projects should target .NET 9 and follow the naming conventions (MyApp.Shared, MyApp.Domain, etc.). Include every project in the solution file and make sure each project’s .csproj contains the appropriate <ProjectReference> entries to other projects. This guarantees the intended dependency flow is enforced and the solution builds without missing references.
- **Testing Guidelines**
  - Adhere to the testing strategy when generating test code. Create unit tests that focus on Domain and Application logic (using Moq or similar for dependencies) and integration tests that cover end-to-end scenarios with realistic infrastructure (e.g. ephemeral databases via Testcontainers, stubbed external APIs via WireMock). Ensure tests are placed in the correct test projects and use the proper frameworks (such as xUnit for .NET). Aim for tests that validate compliance with business rules and integration points as described in the docs.
