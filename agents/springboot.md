# agents.md — Spring Boot 4 Project Guidelines

> Coding-agent instructions for working in this repository.
> Every AI assistant, copilot, or automated tool that touches this codebase **must** follow these rules.

---

## 1  Project Structure and Architecture

Follow the existing package layout exactly. The canonical structure is:

```
src/main/java/com/example/app/
├── Application.java              # Spring Boot main class
├── config/                       # Security, CORS, Jackson, etc.
├── controller/ (or web/)         # REST controllers
├── service/                      # Business-logic services
├── repository/                   # Spring Data JPA repositories
├── domain/ (or model/)           # JPA entities and domain objects
└── dto/                          # Request/response DTOs

src/test/java/...                 # Tests mirror the main source tree
```

When adding new functionality:

- **Controllers are thin.** Validate input, map to DTOs, delegate to a service, return the response. No business logic lives here.
- **Services own business rules.** Mark methods `@Transactional` when they modify the database.
- **Repositories are for data access only.** No business logic in repository classes.
- **DTOs at API boundaries.** Never expose JPA entity classes directly over the wire. If DTOs are used elsewhere in the project, use them everywhere.
- **If the repo uses a different package layout,** follow that layout and copy patterns from the nearest similar feature.

---

## 2  Code Style and Conventions

### 2.1  General Java Style

- `CamelCase` for classes, `camelCase` for methods and variables, `UPPER_SNAKE_CASE` for constants.
- Prefer immutable data where practical; never expose mutable public fields.
- Use `Optional` only as a return type where it makes sense. Never use `Optional` for fields or method parameters.

### 2.2  Spring and REST Conventions

**Controllers**

- Annotate with `@RestController` and `@RequestMapping("/api/...")`.
- Use specific method mappings: `@GetMapping`, `@PostMapping`, `@PutMapping`, `@PatchMapping`, `@DeleteMapping`.
- Return `ResponseEntity<T>` when you need explicit control over status codes or headers; otherwise let Spring infer the status.
- Bind request/response DTOs — never bind entities directly to controller methods.

**Services**

- Annotate with `@Service`.
- Use **constructor injection** — never field injection.
- Apply `@Transactional` on methods that modify the database, when necessary.

**Repositories**

- Extend Spring Data interfaces such as `JpaRepository<Entity, IdType>`.
- Prefer derived query methods (`findByEmail`) before resorting to custom JPQL or `@Query`.

---

## 3  Error Handling

- Centralize exception handling with `@RestControllerAdvice` and `@ExceptionHandler` methods.
- Map domain-specific exceptions to appropriate HTTP status codes (e.g., `EntityNotFoundException` → `404`).
- **Never leak** internal exception messages or stack traces to the client.

---

## 4  Logging

- Use SLF4J: `private static final Logger log = LoggerFactory.getLogger(MyClass.class);`
- Log at appropriate levels: `info` for major events, `debug` for detailed diagnostics, `warn`/`error` for issues.
- **Never log secrets, tokens, or passwords.**

---

## 5  Testing Strategy

Use **JUnit 5** (`@Test`) and Spring's testing support according to the layer under test:

| Layer        | Annotation / Tool                    | Mocking Approach                         |
|--------------|--------------------------------------|------------------------------------------|
| Service      | Plain JUnit 5                        | Mockito for repositories & external deps |
| Controller   | `@WebMvcTest` with `MockMvc`         | Mock services                            |
| Repository   | `@DataJpaTest`                       | In-memory DB or Testcontainers           |
| Integration  | `@SpringBootTest`                    | Full context, real or containerised deps |

Rules for new or changed code:

- Add or update **unit tests** for services and helper classes.
- Add **web tests** for controllers when request/response contracts change.
- Add **integration tests** for critical flows (repository + DB interactions).
- Follow patterns from existing tests in the repo.
- Each test should be **small and focused** — exercise one behaviour per test method.

---

## 6  Database and Migrations

If **Flyway** or **Liquibase** is present (`db/migration` or `db/changelog`):

- Keep schema changes in migration files. **Never hand-edit existing migrations** in a shared environment.
- Add new migration scripts for new tables, columns, or constraints.
- Keep JPA entities in sync with the database schema.
- Use explicit column names and constraints for clarity.

When adding new entities:

- Provide `@Entity`, `@Table`, and `@Id` mappings.
- Use `@Column(nullable = false)` where appropriate.
- Model relationships explicitly with `@OneToMany`, `@ManyToOne`, etc., and configure `fetch` and `cascade` thoughtfully.

---

## 7  API Design and Contracts

- Preserve existing URL structures, HTTP methods, and status codes unless a change is explicitly requested.
- Follow RESTful naming: `/api/users`, `/api/users/{id}`, `/api/users/{id}/roles`.
- Use `GET` for retrieval, `POST` for creation, `PUT`/`PATCH` for updates, `DELETE` for deletion.
- Maintain the project's existing response envelope style (plain objects vs `{ "data": ... }` wrappers).

If **OpenAPI / Swagger** is configured (e.g., springdoc):

- Update API docs when adding or changing endpoints.
- Reuse existing schema components where possible.
- When contracts change, update any documentation or OpenAPI definitions in the repo.

---

## 8  Security and Authentication

- Follow the existing security configuration in `SecurityConfig` (or equivalent).
- **Do not weaken existing security:**
  - Do not introduce `permitAll()` on sensitive endpoints.
  - Do not bypass authentication or authorization logic.
- When adding new endpoints, apply appropriate method-level security annotations (`@PreAuthorize`, etc.) consistent with existing patterns.
- If helper services exist for JWT or password handling, use them — do not reimplement in controller code.
- **Never log or expose** passwords, JWTs, API keys, or secrets.

---

## 9  Git, Branches, and PR Expectations

- Keep diffs **small and scoped** to the requested change.
- Follow the repository's branch strategy. If none is specified, use:
  - `feature/short-description` or `bugfix/short-description`.
- Write commit messages as **concise imperative sentences**: `"Add user search endpoint"` not `"Added user search endpoint"`.
- PR descriptions must include:
  - What changed and why.
  - Any schema or API contract changes.
  - How to test the change (commands and steps).

---

## 10  Commands Quick Reference

### Allowed without asking

```bash
# Maven — build and test
./mvnw clean verify
./mvnw test
./mvnw -Dtest=SomeTestClass test
./mvnw spring-boot:run

# Gradle equivalents
./gradlew clean build
./gradlew test
./gradlew bootRun

# Local infrastructure
docker compose up -d
```

### Ask before running

- Adding new Maven/Gradle dependencies (especially non-Spring-starter or heavy libraries).
- Large refactors (architecture changes, moving many packages).
- Destructive database commands or modifications to production-like configuration.

---

## 11  Do / Don't Checklist

### Do

- Keep changes minimal and focused on the requested task.
- Follow existing patterns and naming from similar classes.
- Add or update tests for any new behaviour.
- Keep controllers thin; push business logic into services.
- Keep configuration in `application.yml` / `application-*.yml` consistent with existing profiles.

### Don't

- Introduce new frameworks or architectural styles without explicit instruction.
- Bypass or weaken security to "make tests pass."
- Expose entity classes directly as API responses if DTOs are used elsewhere.
- Perform repository-wide renames or format changes unless explicitly asked.
- Log secrets, tokens, or full stack traces to clients.
