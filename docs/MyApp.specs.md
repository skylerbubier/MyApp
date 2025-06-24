# {MyApp} Application Specification

## Overview / Summary

Provide a brief summary of what this application/service does, its domain context, and its primary objectives or features.

## Data Models

List the core domain entities and value objects, along with their key properties and any important business rules or relationships.

- EntityName – Description of what this entity represents, its main properties, and any pertinent rules or invariants.
- AnotherEntity – Description of another important model, including its attributes and relationships (e.g. parent/child relations, constraints, etc.).

## Required Infrastructure / Data Providers

Specify the infrastructure components required by this application (such as databases, caches, file storage, etc.), including the chosen technologies or providers.

- Database – The type of database and its purpose (e.g. "SQL Server for storing order and customer data").
- Cache/Storage – Any other infrastructure services (e.g. "Redis cache for session data", "AWS S3 for file storage for user uploads").

## External Integrations

Outline any external systems or third-party APIs that the application will integrate with. Include what each integration is used for.

- External API Name – Brief description of the external API and why it's needed (e.g. "Payment Gateway API – used to process customer payments").
- Messaging Service – Description of any message broker or event streaming integration (e.g. "RabbitMQ – publishes order events for other services to consume").

## Use Cases (Commands & Queries)

Detail the key use cases of the application in terms of commands (actions that change state) and queries (read-only operations). For each use case, provide a name and a brief description.

- UseCaseName – Command: Description of this command use-case and its outcome (e.g. "CreateOrder – Command: validates input and creates a new order in the system").
- UseCaseName – Query: Description of this query use-case and the data it returns (e.g. "GetOrderStatus – Query: retrieves the current status of a given order by order ID").

## Special Testing & Deployment Considerations

Note any special requirements for testing (such as integration test needs, test data setup) and any deployment or runtime constraints.

- Testing: e.g. requirements to test with certain external systems, performance testing considerations, etc.
- Deployment: e.g. environment or hosting constraints ("must run in Linux containers", "requires specific OS or hardware", "needs environment variables for config", etc.).

## Additional Notes / Example Scenarios

Provide any additional information, clarifications, or example scenarios to illustrate how the system should behave.

- Example Scenario: (e.g. "A user submits an order via the API, the service validates it, saves it to the database, and publishes an 'OrderCreated' event for downstream services...").
- Additional Notes: (e.g. any assumptions, non-functional requirements, or future considerations that developers should be aware of).
