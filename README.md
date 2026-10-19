# Stays · Lisbon Hotel Reservations

A Java and React hotel reservation demo forked from [hotel-java-simple](https://github.com/marcelomiyake/hotel-java-simple). The backend and frontend are organized around domain rules, application use cases, ports, and adapters, while the existing visual system and HTTP API contracts are retained.

![Stays hotel reservation search page](docs/screenshots/hotel-reservation-home.png)

## Case study and known defects

This is the **Java refactored** implementation in [One Hotel Design, Five AI Implementations](https://marcelomiyake.com.br/posts/five-ai-hotel-implementations/), which compares five AI-built versions of the same hotel design, and in its [Part II on the cost of a new feature](https://marcelomiyake.com.br/posts/five-ai-hotel-implementations-part-2/). Part I inspected commit [`429f97f`](https://github.com/marcelomiyake/hotel-java-refactored/commit/429f97f28ffaa47ba208463fd11e555763a07b58).

The study found:

- **The packaged services do not start as built.** As in Java simple, all four service JARs lack a `Main-Class` entry because the POM does not bind the Spring Boot `repackage` goal, and the Docker images start them with `java -jar`.
- From source review, not reproduced: cancellation reads the reservation without a row lock before releasing inventory; booking never checks the guest count against room capacity; reservation lookup and cancellation routes are unprotected; and the duplicate-key fallback returns an existing reservation without re-checking that the request matches.
- Part II: once the sweeper marks a session `ABANDONED`, later checkout and confirmation events cannot change it; a database check left a resumed session at `ABANDONED:DETAILS`. Completion comes from a browser event, not from payment state.

No later commit fixes these. The passing tests and SonarQube Cloud results below do not cover them. This is a local learning implementation, not a production system.

## Product behavior

- Search Lisbon hotels by destination, dates, and guest count; compare availability and daily rates.
- Reserve by room type. PostgreSQL stores daily inventory, supports up to 10% overbooking, and uses version-checked updates for concurrent bookings.
- Replay a reservation with its UUID as an idempotency key. Reusing that key with changed details returns a conflict.
- Read reservation history by email and cancel a confirmed reservation. Cancellation releases inventory and records a simulated refund.
- Let staff edit room types, capacity, inventory, and nightly rates behind the admin key.

The seeded catalog contains two Lisbon hotels. Payments are simulated; the demo does not collect card details or contact a hotel.

## Architecture

The backend is split into five Spring Boot services: hotel catalog, rates, payments, reservations, and shared API concerns. Each business service uses the same dependency direction:

| Layer | Responsibility |
| --- | --- |
| `domain` | Business concepts and invariants such as hotel, reservation, payment status, and rate period |
| `application/command` | State-changing use cases and command handlers |
| `application/query` | Read use cases and query handlers |
| `application/port` | Interfaces for persistence and calls to other services |
| `adapter/in/web` | HTTP controllers and API mapping |
| `adapter/out/jdbc` | Database implementations of persistence ports |
| `adapter/out/http` | Reservation service clients for catalog, rates, and payments |

Commands and queries have separate application interfaces and handlers. Domain objects enforce their own valid state. Application code depends on ports; JDBC, HTTP, and Spring MVC remain adapters around those use cases. This keeps business rules testable without changing the public routes, request headers, or JSON response fields.

The React app follows the same direction:

| Frontend area | Responsibility |
| --- | --- |
| `src/domain` | Booking and search validation rules |
| `src/application` | Hotel query and command interfaces |
| `src/infrastructure` | HTTP gateway implementing the application interfaces |
| `src/presentation` | React screens and interaction flow |
| `src/main.tsx` | Composition of the HTTP gateway and presentation |

`frontend/src/styles.css` retains the existing design system. The API-facing gateway continues to call the same `/api` routes and uses the same request and response contracts.

## Run on local Kind

Requirements: Docker, Kind, kubectl, and `openssl`.

```bash
./scripts/kind-up.sh
```

Open [http://localhost:8080](http://localhost:8080). The script creates the `hotel-reservation` cluster, builds and loads local images, and deploys the app. It generates local database and staff keys in `.kind-db-password` and `.kind-admin-key`; those files are ignored by Git.

```bash
./scripts/kind-down.sh
kubectl -n hotel-reservation get deployments,pods,services
```

## HTTP API contracts

| Route | Purpose |
| --- | --- |
| `GET /api/search?destination=Lisbon&checkIn=YYYY-MM-DD&checkOut=YYYY-MM-DD&guests=2` | Search availability and rates |
| `GET /api/hotels?destination=Lisbon` | Read hotel catalog |
| `GET /api/rates/quote?roomTypeId=…&checkIn=…&checkOut=…` | Quote every night in a stay |
| `POST /api/reservations` | Create or replay a reservation |
| `GET /api/reservations?email=…` | Read a guest’s reservation history |
| `DELETE /api/reservations/{id}` | Cancel a reservation and refund the demo payment |
| `POST /api/admin/hotels/{hotelId}/room-types` | Add a room type (`X-Admin-Key`) |
| `PUT /api/admin/room-types/{id}` | Update a room type (`X-Admin-Key`) |
| `PUT /api/admin/inventory` | Update future inventory (`X-Admin-Key`) |
| `PUT /api/admin/rates` | Update one nightly rate (`X-Admin-Key`) |

## Quality checks

Frontend checks:

```bash
cd frontend
npm ci
npm test
npm run build
npm run lint
```

Java integration tests and JaCoCo reports (Docker socket required):

```bash
cd ..
docker run --rm --network=host \
  -v "$PWD":/workspace \
  -v "$HOME/.m2":/root/.m2 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -w /workspace maven:3.9-eclipse-temurin-25 mvn -B clean verify
```

The verified frontend suite has **15 passing tests** and **92.49% line coverage**. The Java suite has **28 passing tests** and **87.72% aggregate line coverage** across JaCoCo reports.

SonarQube Cloud project: [Hotel Reservation System · DDD Clean Architecture CQRS](https://sonarcloud.io/project/overview?id=marcelomiyake_hotel-dry-kiss-yagni-refactored-ddd-cleanarch-cqrs). The CI-based SonarScanner CLI analysis includes Java, TypeScript, and CSS, imports the JaCoCo and frontend LCOV reports, and currently reports **0 open issues**, **87.3% overall coverage**, and a **passed quality gate**. SonarCloud documents coverage import for CI-based analysis in its [test coverage guide](https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/test-coverage/overview).

After generating the frontend and Java coverage reports with the commands above, export `SONAR_TOKEN` and run the scanner from the repository root:

```bash
docker run --rm --network=host \
  -e SONAR_TOKEN -e SONAR_HOST_URL=https://sonarcloud.io \
  -v "$PWD":/usr/src \
  -v "$HOME/.m2/repository":/maven-repository:ro \
  -w /usr/src sonarsource/sonar-scanner-cli:latest \
  '-Dsonar.java.libraries=/maven-repository/**/*.jar' \
  '-Dsonar.java.test.libraries=/maven-repository/**/*.jar'
```

For Lighthouse, build and serve the production frontend with `npm run preview -- --host 0.0.0.0`, then audit `http://localhost:4173/` with the desktop preset. The latest recorded audit scored **100** for performance, accessibility, best practices, and SEO; its experimental agentic-browsing readiness checks scored **1.0**. This reservation tracking feature does not change the page content or metadata; a fresh audit of this revision is pending Chrome DevTools MCP availability. SEO metadata includes the page title and description, Open Graph and Twitter fields, TravelAgency JSON-LD, `robots.txt`, `llms.txt`, and an AI Catalog manifest.

## Reservation abandonment tracking

The reservation service creates a booking only after checkout is submitted, which leaves earlier abandonment invisible to the backend. The browser now starts an anonymous progress session when a guest opens reservation details, records `DETAILS` and `CHECKOUT` screen events, and records `CONFIRMATION` only after the reservation request succeeds. It sends only a random session UUID and a fixed screen value; it does not send guest name, email, or search details.

PostgreSQL stores a current session row and an append-only event history in `reservations.reservation_funnel_sessions` and `reservations.reservation_funnel_events`. A scheduled service marks a session `ABANDONED` after 30 minutes without activity and records the last screen. The timeout and sweep interval are configurable with `RESERVATION_PROGRESS_INACTIVITY_TIMEOUT` and `RESERVATION_PROGRESS_SWEEP_INTERVAL_MS`. No administrative frontend was added. The frontend’s visible flow and SEO metadata are unchanged.

## AI build record

This implementation was generated with Codex (GPT-6 Luna, max effort). The original prompts, token usage, cost estimates, and change counts for the initial build and the reservation-abandonment feature are in [docs/ai-build-record.md](docs/ai-build-record.md).
