<<<<<<< HEAD
# StudySync — Study Group & Notes Sharing Platform

One place for students to share notes, join subject study groups, ask & answer questions, chat in
real time, and collaborate in virtual study rooms. **API-first**: the same Spring Boot backend
powers the React web app today and is designed so a future Android client reuses it unchanged
(REST + STOMP, no web-only coupling server-side).

Built from the attached Phase 0 system design. Product name: **StudySync**.

---

## Tech stack

| Layer | Tech |
|---|---|
| Frontend | React 18, Vite 5, MUI 5, Redux Toolkit + RTK Query, React Router 6, STOMP.js + SockJS |
| Backend | Java 17, Spring Boot 3.3 (Web, Security, Data JPA, WebSocket/STOMP, Validation) |
| Database | MySQL 8, schema managed by Hibernate (`ddl-auto=update`) |
| Auth | JWT access (15 min) + rotating refresh (7 d, stored hashed), BCrypt strength 12 |
| Realtime | STOMP over `/ws` (SockJS fallback), JWT-authenticated CONNECT frame |
| Storage | Pluggable `StorageService` — local disk in dev, S3/Cloudinary in prod |

---

## Prerequisites

- **JDK 17** (the backend targets 17).
- **MySQL 8** — a local install, or via Docker (`docker-compose.yml` included).
- **Node 18+** and npm for the frontend.
- Maven is **not** required globally — the backend ships a wrapper (`./mvnw`). An IDE-bundled
  Maven works too.

---

## Run it locally

### 1. Database

```bash
docker compose up -d        # starts MySQL 8 on :3306 (db=studysync, root password=root)
```

No Docker? Just have a local **MySQL 8** running. The app connects as `root`/`root` and
**creates the `studysync` database automatically** (`createDatabaseIfNotExist=true`). If your root
password isn't `root`, set `DB_PASS` (and `DB_USER`/`DB_URL` if needed) before starting the backend.

### 2. Backend (builds the schema on boot)

```bash
cd backend
./mvnw spring-boot:run            # Windows: mvnw.cmd spring-boot:run
```

- API base: `http://localhost:8080/api/v1`
- Interactive API docs (Swagger UI): `http://localhost:8080/swagger-ui.html`
- Hibernate creates/updates the tables and a seeder inserts default subjects on first start.

### 3. Frontend

```bash
cd frontend
cp .env.example .env       # already provided; relative URLs proxy to :8080 in dev
npm install
npm run dev                # http://localhost:5173
```

Open `http://localhost:5173`, register an account, and you're in.

---

## What's implemented (all six v1 modules)

1. **Auth & users** — register, login, refresh (rotation + reuse-detection), logout, profile edit,
   password change (revokes all refresh tokens).
2. **Notes** — validated upload (PDF/DOC/PPT, type + size checks), search by subject/title,
   streamed download with an atomic download counter, owner/admin delete.
3. **Study groups** — create/join/leave, members, owner removes members, shared notes, REST
   discussion thread.
4. **Q&A forum** — ask, answer, up/down-vote (one vote per user/answer, idempotent), answers sorted
   by net votes, view counter, search.
5. **Real-time chat** — direct + group conversations, REST history, live delivery over STOMP,
   typing indicator, attachments. The socket only carries live deltas; history loads via REST.
6. **Virtual study rooms** — open/join/leave, host ends room, participant list, shared-notes panel,
   live room discussion + join/leave/end broadcasts.

Cross-cutting: DTOs at every boundary (entities never exposed), centralized exception handling with
a single `ApiError` envelope, a standard `PageResponse` envelope on every list endpoint, paginated
list endpoints, indexed FKs/search columns, per-IP rate limiting on auth, and CORS scoped to the
web origin.

---

## API shape

Everything is under `/api/v1`. Two envelopes are used everywhere:

- **Error**: `{ timestamp, status, error, message, path, fieldErrors[], traceId }`
- **Page**: `{ content[], page, size, totalElements, totalPages, first, last, sort }`

List endpoints accept `?page=&size=&sort=` plus their own filters. Full endpoint reference: Swagger
UI (above) or the Postman collection in [`docs/StudySync.postman_collection.json`](docs/StudySync.postman_collection.json).

### WebSocket (STOMP at `/ws`)

| Direction | Destination | Purpose |
|---|---|---|
| send | `/app/chat.send` | send a message (persisted, then fanned out) |
| send | `/app/chat.typing` | transient typing indicator |
| send | `/app/room.post` | post to a room discussion |
| send | `/app/presence.ping` | presence heartbeat |
| subscribe | `/topic/conversation/{id}` | conversation messages/typing |
| subscribe | `/user/queue/messages` | personal message inbox |
| subscribe | `/topic/room/{roomId}` | room discussion + participant events |
| subscribe | `/topic/presence/{scopeId}` | presence updates |

The access JWT is sent in the STOMP `Authorization` header on CONNECT and validated by a channel
interceptor before any SUBSCRIBE/SEND is authorized.

---

## Configuration (env vars, all have dev defaults)

| Var | Default | Notes |
|---|---|---|
| `DB_URL` / `DB_USER` / `DB_PASS` | `jdbc:mysql://localhost:3306/studysync?...`, `root`, `root` | MySQL connection |
| `JWT_SECRET` | dev placeholder | **set a long random value in prod** |
| `JWT_ACCESS_TTL` / `JWT_REFRESH_TTL` | `PT15M` / `P7D` | ISO-8601 durations |
| `CORS_ORIGINS` | `http://localhost:5173` | comma-separated web origins |
| `STORAGE_PROVIDER` | `local` | `local` \| `s3` \| `cloudinary` |
| `STORAGE_DIR` | `./var/uploads` | local storage root |

---

## Production / deployment notes

- **DB**: managed MySQL; Hibernate creates/updates the schema on boot. For stricter prod control,
  set `ddl-auto=validate` and own the schema with a migration tool (Flyway/Liquibase).
- **Secrets**: provide a strong `JWT_SECRET`; never ship the dev default.
- **Storage**: set `STORAGE_PROVIDER=s3` and add an `S3StorageService` implementation
  (`@ConditionalOnProperty(name="storage.provider", havingValue="s3")`) — the `StorageService`
  interface and call sites stay unchanged. Cloud files should be served via short-lived signed URLs.
- **Frontend**: `npm run build` produces static files in `frontend/dist`; serve behind any CDN/host
  and point `VITE_API_BASE_URL` / `VITE_WS_URL` at the deployed API.
- **Scale**: the API is stateless (JWT identity), so it scales horizontally. The STOMP broker is the
  in-memory simple broker; for multi-node, switch to an external broker relay (RabbitMQ/ActiveMQ).
  The auth rate-limiter is in-memory; for multi-node, back it with Redis.

---

## Notable decisions & deviations from the spec

These are intentional and called out so nothing is silent:

- **Lombok** was added (not in the original spec) to keep entities legible. Compile-time only.
- **Build tool**: Maven (the spec resolved "Maven + Gradle → Maven"). Java **17** (not 21) to match
  the available toolchain; Spring Boot 3.3 supports it.
- **Database: MySQL 8** (the spec named PostgreSQL/Flyway). Switched at the developer's request for
  easy local testing. The schema is owned by **Hibernate** (`ddl-auto=update`) with a Java seeder
  for default subjects, instead of Flyway migrations. JPA/Spring Data code is unchanged and
  database-agnostic, so moving back to Postgres is just a driver + `ddl-auto` change.
- **Rate limiting** uses a small in-house token bucket instead of Bucket4j (the published Bucket4j
  coordinates didn't resolve) — same behavior, one fewer dependency. Swap for Redis-backed Bucket4j
  in prod.
- **Group ↔ chat link**: rather than a `conversation_id` FK on `study_groups`, the `Conversation`
  references the group (`group_id`) and is created on demand. Equivalent, avoids a cyclic FK.
- **Room discussion** is live-only over STOMP (no table) — matches the ERD, which has no room-message
  entity. The durable room artifact is the shared-notes panel.
- **Search** uses portable `lower(...) LIKE` rather than DB-specific full-text. On MySQL a `FULLTEXT`
  index is the drop-in upgrade later.
- The prototype's "smart study timetable / weekly goal" is **out of scope** — it has no entity or
  endpoint in the Phase 0 data model/API contract.

---

## Project layout

```
backend/    Spring Boot API — vertical slices per module
  src/main/java/com/studysync/
    common/    BaseEntity, ApiError, PageResponse, GlobalExceptionHandler, exceptions
    config/    CORS, OpenAPI, WebSocket, rate limit, upload props
    security/  JWT service/filter, principal, WS auth interceptor, SecurityConfig
    storage/   StorageService + LocalStorageService + FileController
    user/  subject/  note/  group/  qa/  chat/  room/   (controller→service→repo→entity + dto)
frontend/   React + Vite client, feature-per-folder mirroring the backend
docs/       Postman collection
_inspect/   The original Phase 0 design docs (reference only)
```

## Tests

```bash
cd backend && ./mvnw test
```

Includes a pure unit test for the JWT service. Module/integration tests against a real Postgres are
the natural next step (Testcontainers).
=======
# StudySync
>>>>>>> 9602142204ef6b0b496e1ebc46b623f1964de8a7
