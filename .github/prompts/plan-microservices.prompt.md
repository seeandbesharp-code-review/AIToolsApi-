## Plan for Decomposing WebApiShop Monolith into Microservices

**TL;DR**
The existing ASP.NET Core monolith contains users, products, categories, orders, ratings, and password logic. We will carve it into a set of bounded-context services (User/Auth, Product/Catalog, Order, Rating, etc.), choose technology stacks per service (keep .NET where it makes sense but allow polyglot), and pick appropriate storage (relational for transactions, NoSQL/event store where eventual consistency is acceptable). The plan explains domains, service boundaries, data ownership, communication patterns, and migration steps.

---

### 1. Domain Decomposition

1. **User/Authentication Service (`UserSvc`)**
- Manages user profiles, registration, login, and roles.
- Holds `Users` and `Passwords` tables and zxcvbn validation.
- Issues JWTs or integrates with OAuth2.
- Language: C#/.NET (reuses existing code); could be Node/Go if team prefers polyglot.
- DB: Relational (SQL Server or PostgreSQL) for strong consistency and ACID for credentials.

2. **Catalog Service**
- Products and categories.
- Exposes product search, filtering, and category hierarchy.
- Language: C#/.NET initially; can evolve independently.
- DB: Relational or document DB (for example PostgreSQL with JSONB, MongoDB) depending on query needs.

3. **Order Service**
- Handles orders, order items, status, and inventory reservation if added later.
- Owns `Orders` and `OrderItems` entities and orchestrates fulfillment.
- Language: C# or JVM/Go for performance.
- DB: Relational for transactional integrity; consider a separate SQL instance.

4. **Rating Service**
- Manages product ratings and reviews.
- Can be eventual-consistent with denormalized read models.
- Language: lightweight (Go, Node, .NET).
- DB: Document store (MongoDB/Cosmos) or relational with a simpler schema.

5. **Category Service (optional split)**
- If category tree logic becomes complex, treat separately; otherwise keep embedded in Catalog.

6. **Gateway/API Aggregator**
- Front door for clients; routes requests to services and handles auth/rate limiting.
- Could be Kong, Ocelot (.NET), or NGINX.

7. **Common/Shared Libraries**
- DTOs, logging, and error middleware can be packaged as NuGet packages or kept in separate repos for reuse.

---

### 2. Technology and Storage Recommendations

- **Languages**
- **Primary**: Continue with C#/.NET to leverage existing codebase, teams, and integrations.
- **Polyglot option**: Allow teams to pick Go, Node.js, Java, or Python for new services if justified by domain requirements (for example a high-performance order engine in Go).
- **Rationale**: Microservices enable technology experimentation; start with .NET for minimal disruption.

- **Databases**
- **Relational (SQL Server, PostgreSQL)** for user credentials, orders, and transactional consistency.
- **Document/NoSQL (MongoDB, Cosmos DB, DynamoDB)** for ratings and catalog metadata where flexible schema helps.
- **Event store or Kafka** for cross-service events (`OrderCreated`, `ProductUpdated`, `UserRegistered`).
- **Per-service DB ownership**: no shared database; each service owns its schema to avoid coupling.

---

### 3. Communication Patterns

1. **Synchronous HTTP/REST** between gateway and services.
2. **Asynchronous messaging** (RabbitMQ/Kafka) for events: `OrderCreated`, `ProductUpdated`, `UserRegistered`.
3. **API contracts**: use OpenAPI specs; each service publishes its own Swagger.

---

### 4. Migration Strategy

1. **Extract services one at a time**
- Begin with a low-risk domain (Rating or Catalog).
- Create new repos/projects with existing controllers/services mapped.
- Deploy side-by-side with monolith; modify gateway routing.

2. **Data replication**
- For initial reads, replicate data from monolith DB to service DB via CDC or ETL.
- Stop updating monolith for that domain once service is live.

3. **Strangling the monolith**
- Gradually move controllers and business logic into services.
- Replace internal calls with HTTP/message-based calls.

4. **Testing and validation**
- Maintain existing unit/integration tests per service.
- Add contract tests (Pact) to ensure backward compatibility.

5. **Deployment**
- Containerize each service (Docker).
- Use orchestration (Kubernetes or Docker Compose) for local development.
- Set up CI/CD pipelines per service.

6. **Operational concerns**
- Centralized logging (ELK/Seq) and monitoring (Prometheus/Grafana).
- Circuit breakers and retries (Polly).
- Version APIs carefully and deprecate monolith endpoints gradually.

---

### 5. Verification

- **Manual checks**
- Hit gateway endpoints and confirm correct service responses.
- Ensure user registration/login works after migration.

- **Automated tests**
- Run service-specific unit tests with `dotnet test`.
- Execute integration tests targeting individual service databases and through gateway.

- **Load testing**
- Benchmark services independently to validate DB choices.

---

### 6. Decisions

- Start with all services in C#/.NET to leverage code reuse.
- Use SQL Server for transactional services; consider PostgreSQL for new deployments to avoid vendor lock-in.
- Allow eventual polyglot expansion after the first two services are stable.
- Adopt event-driven communication for decoupling and scalability.

---

This plan provides a technical roadmap for decomposing the monolith, choosing language/database per domain, and migrating incrementally while maintaining functionality.
