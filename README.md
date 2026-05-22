# Sales.Fulfillment
Sales.Fulfillment is the fulfillment service in the Sales backend. It is a .NET 10 ASP.NET Core service responsible for processing orders, orchestrating fulfillment workflows, maintaining fulfillment-related data in the database, and exposing real-time updates via SignalR hubs for connected clients (dashboards, management UIs, and other services).

This README summarizes the project responsibilities and describes the primary components: services, hubs, filters, controllers, data access, configuration, and running/testing guidance.

Overview
- Purpose: Process and manage order fulfillment tasks (pick/pack/ship), update fulfillment state, track shipments and inventory reservations, and provide operators with real-time telemetry and control.
- Surface area: RESTful HTTP APIs for management/automation, SignalR hubs for live telemetry and notifications, background workers for long-running orchestration, and a DbContext for persistence.

Architecture
- ASP.NET Core Web API project targeting .NET 10.
- Layered structure:
  - Controllers: HTTP endpoints for external clients and internal automation.
  - Services: Domain logic for fulfillment workflows, orchestration, and integrations (carrier APIs, warehouse systems).
  - Hubs: SignalR hubs that push fulfillment state and telemetry to connected clients.
  - Filters/Middleware: Cross-cutting concerns such as validation, authorization, exception handling, and request logging.
  - Data: Entity Framework Core DbContext and entities for persistence.
  - Background workers: Hosted services for retry, reconciliation, and integration polling.

Key components

Controllers
- Api controllers expose endpoints for:
  - Creating and updating fulfillment requests
  - Querying fulfillment status and history
  - Triggering manual actions (retry, cancel, re-route)
  - Administrative operations (reconcile inventory, rebuild snapshots)

Examples (routes may vary by implementation):
- POST /api/fulfillment/orders
- GET /api/fulfillment/orders/{id}
- POST /api/fulfillment/orders/{id}/cancel
- POST /api/fulfillment/rebuild

Services
- FulfillmentService: Core orchestration that advances orders through states (Queued, Processing, Picked, Packed, Shipped, Completed, Failed).
- InventoryReservationService: Reserve and release inventory for orders.
- CarrierIntegrationService: Abstracts carrier APIs (create shipment, track status) and handles webhooks or polling.
- RetryAndCompensationService: Handles retry policies and compensating actions for failed steps.

Hubs (SignalR)
- FulfillmentHub: Publishes per-order state changes, aggregated telemetry (active orders, throughput), and operator messages.
  - Clients subscribe to updates to show live dashboards or operator consoles.
  - Typical events: OrderUpdated, FulfillmentTelemetry, OperatorNotification.
- NotificationHub (optional): General-purpose push hub for system-wide alerts and administrative messages.

Filters and Middleware
- Validation filters: Validate incoming DTOs and return standardized validation responses.
- Authorization filters: Enforce roles/claims for management or operator actions.
- Exception handling middleware: Capture unhandled exceptions and map them to proper HTTP responses and structured logs.
- Action filters: Timing and metrics collection around controller actions for observability.

Data Layer
- FulfillmentDbContext: EF Core DbContext containing entities like Orders, OrderItems, FulfillmentSteps, Shipments, InventoryReservations, and AuditRecords.
- Migrations: Store EF Core migrations under the Migrations folder. Use dotnet ef migrations commands for schema changes.
- Persistence strategy: Use SQLite for local development tests or a configured relational provider (SQL Server, PostgreSQL) in production via appsettings.

Background Workers
- Hosted services for:
  - Polling carrier APIs for tracking updates.
  - Reconciliation tasks to detect drift between orders and shipments.
  - Scheduled retries for transient failures.

Configuration
- appsettings.json and environment variables control DB connection, SignalR options, external integrations, and feature toggles.
- Important keys:
  - ConnectionStrings:Fulfillment -> EF Core provider connection string
  - SignalR:HubPath -> Route for SignalR hubs (default: /fulfillmentHub)
  - Integrations:Carrier:{Provider,ApiKey,Endpoint}
  - FeatureToggles:{EnableAutoRetry,EnableAutoAssign}

Running Locally
1. Ensure .NET 10 SDK is installed.
2. Restore and build project:
   dotnet restore
   dotnet build
3. Apply migrations (if using EF migrations locally):
   dotnet ef database update --project Sales.Fulfillment --startup-project Sales.Fulfillment
4. Run the service:
   dotnet run --project Sales.Fulfillment
5. Open the API or connect a client to the SignalR hub at the configured hub path (e.g. /fulfillmentHub).

Testing
- Unit tests should mock external integrations and test orchestration flows in services.
- Integration tests can use an in-memory or SQLite provider to validate DbContext behavior and controller endpoints.
- Manual testing: use the included SignalR-compatible dashboard or a simple SignalR client to observe events.

Observability
- Structured logging via Microsoft.Extensions.Logging.
- Health checks for DB connectivity and integration endpoints (AddHealthChecks in Program).
- Metrics via Prometheus or Application Insights depending on hosting configuration.

Security
- Protect management and operator endpoints with authentication and authorization.
- Sanitize external inputs and validate webhooks using signatures when available.
- Keep integration keys and secrets in a secure store (Key Vault, environment variables, or secret manager).

Contributing
- Follow repository contributing guidelines and code style.
- Run the solution build and unit tests before submitting PRs.
- Document behavioral changes and update README when adding new integration points or public endpoints.

Troubleshooting
- If the service fails to start, check connection string and EF migrations.
- If SignalR clients do not receive updates, ensure the hub path is mapped and CORS is configured for client origins.
- Check logs for exceptions and retry policy behavior in background workers.

Notes
- This document describes the intended responsibilities and common components; specific file and class names may vary. See source files for exact API signatures and types.

License
- Refer to the repository root for license information.
