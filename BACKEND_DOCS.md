# ICCA CTF — Backend Documentation

**Stack:** NestJS · PostgreSQL (Prisma ORM) · Redis · BullMQ · Socket.IO · Clerk Auth  
**Base URL:** `http://localhost:3000/api/v1`  
**API Docs (dev only):** `http://localhost:3000/api/v1/docs`

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Authentication & Authorization](#authentication--authorization)
3. [Modules & API Reference](#modules--api-reference)
   - [Auth](#auth)
   - [Competitions](#competitions)
   - [Challenges](#challenges)
   - [Submissions](#submissions)
   - [Teams](#teams)
   - [Leaderboard](#leaderboard)
   - [Instances (Docker)](#instances-docker)
   - [Certificates](#certificates)
   - [Announcements](#announcements)
   - [Learning Materials](#learning-materials)
   - [Docker Images](#docker-images)
   - [Users](#users)
   - [Admin](#admin)
   - [SuperAdmin](#superadmin)
   - [Dashboard](#dashboard)
   - [Files & Uploads](#files--uploads)
   - [Health / Monitoring](#health--monitoring)
4. [WebSocket Events](#websocket-events)
5. [Background Jobs](#background-jobs)
6. [Common Infrastructure](#common-infrastructure)
7. [Security Features](#security-features)
8. [Deployment](#deployment)

---

## Architecture Overview

```
backend/
├── src/
│   ├── app.module.ts          # Root module
│   ├── main.ts                # Bootstrap (Helmet, CORS, Compression, Validation)
│   ├── common/                # Shared infrastructure
│   │   ├── database/          # Prisma service
│   │   ├── decorators/        # @Auth, @Roles, @RateLimit
│   │   ├── filters/           # HTTP & Validation exception filters
│   │   ├── interceptors/      # Logging, Cache, Transform
│   │   ├── middleware/        # CORS, Security headers
│   │   ├── pipes/             # Validation & Transform
│   │   └── services/          # CryptoService, RateLimitService
│   ├── config/                # App, DB, Redis, Queue configs
│   └── modules/               # Feature modules (see below)
├── prisma/                    # Schema + migrations
└── k8s/                       # Kubernetes manifests
```

The app uses a global prefix `/api/v1` on every route, applies a strict `ValidationPipe` (whitelist + forbidNonWhitelisted), and uses Swagger (development only) auto-generated from decorators.

---

## Authentication & Authorization

Authentication is handled by **Clerk** (JWT-based). Tokens are verified against Clerk's JWKS endpoint.

### Guards

| Guard | Purpose |
|---|---|
| `JwtAuthGuard` | Validates Clerk JWT on every protected route |
| `OptionalAuthGuard` | Validates JWT when present, passes through when absent |
| `RolesGuard` | Checks `user.role` against `@Roles(...)` decorator |
| `AdminGuard` | Requires `ADMIN` or `SUPERADMIN` |
| `SuperAdminGuard` | Requires `SUPERADMIN` only |
| `ContentManagerGuard` | Requires `ADMIN`, `SUPERADMIN`, or approved `CREATOR` |

### Role Hierarchy

```
SUPERADMIN  →  Full system access
ADMIN       →  Manage own competitions, challenges, announcements
CREATOR     →  Create/edit challenges for competitions they are invited to
USER        →  Register, compete, submit flags
```

### JWT Flow

```typescript
// Client: Get Clerk session token, attach as Bearer
Authorization: Bearer <clerk_jwt>

// Backend: ClerkJwtService verifies via JWKS, finds local User by clerkId
const user = await prisma.user.findUnique({ where: { clerkId: payload.sub } });
```

---

## Modules & API Reference

---

### Auth

`/api/v1/auth`

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| `GET` | `/auth/me` | JWT | Get current authenticated user profile |
| `POST` | `/auth/clerk` | Public | Authenticate/sync user with Clerk (`{ clerkId }`) |
| `POST` | `/auth/webhook` | Public (Svix sig) | Receives Clerk lifecycle events (user.created, user.updated, user.deleted) |
| `GET` | `/auth/health` | Public | Service health + Clerk config status |

**Webhook Verification:**

```typescript
// Svix headers required for webhook validation
'svix-id'        → req.headers['svix-id']
'svix-timestamp' → req.headers['svix-timestamp']
'svix-signature' → req.headers['svix-signature']
```

---

### Competitions

`/api/v1/competitions`

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| `POST` | `/competitions` | JWT + Admin | Create a competition |
| `GET` | `/competitions` | Optional JWT | List all public competitions (paginated, filterable) |
| `GET` | `/competitions/my` | JWT | Get competitions the user is registered in |
| `GET` | `/competitions/:id` | Optional JWT | Get competition by ID |
| `GET` | `/competitions/:id/progress` | JWT | Get current user's progress in a competition |
| `PUT` | `/competitions/:id` | JWT + Admin | Update competition details |
| `PATCH` | `/competitions/:id/status` | JWT + Admin | Update status (DRAFT → REGISTRATION\_OPEN → ACTIVE → COMPLETED) |
| `POST` | `/competitions/:id/register` | JWT | Register for a competition |
| `DELETE` | `/competitions/:id/register` | JWT | Unregister from a competition |
| `DELETE` | `/competitions/:id` | JWT + Admin | Delete competition |

**Status lifecycle:**

```
DRAFT → REGISTRATION_OPEN → ACTIVE → PAUSED → COMPLETED
                                   ↘ CANCELLED
```

**Create competition example:**

```typescript
POST /api/v1/competitions
Authorization: Bearer <token>
{
  "name": "ICCA CTF 2026",
  "description": "Annual cybersecurity challenge",
  "startTime": "2026-06-01T09:00:00Z",
  "endTime": "2026-06-03T17:00:00Z",
  "isPublic": true,
  "isTeamBased": true,
  "maxTeams": 50,
  "registrationDeadline": "2026-05-28T00:00:00Z",
  "rules": "No automated tools allowed."
}
```

---

### Challenges

`/api/v1/challenges` and `/api/v1/competitions/:competitionId/challenges`

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| `GET` | `/challenges/public` | Public | List publicly visible challenges |
| `GET` | `/challenges/:challengeId` | JWT | Get challenge details (registered users) |
| `POST` | `/challenges/create` | JWT + ContentManager | Create a standalone challenge |
| `POST` | `/competitions/:cid/challenges` | JWT + ContentManager | Create challenge under competition |
| `GET` | `/competitions/:cid/challenges` | JWT | List challenges in competition |
| `GET` | `/competitions/:cid/challenges/:id` | JWT | Get specific challenge |
| `PUT` | `/competitions/:cid/challenges/:id` | JWT + ContentManager | Update challenge |
| `DELETE` | `/competitions/:cid/challenges/:id` | JWT + ContentManager | Delete challenge |
| `POST` | `/competitions/:cid/challenges/:id/hints` | JWT + ContentManager | Add hint to challenge |
| `POST` | `/competitions/:cid/challenges/:id/hints/:hintId/unlock` | JWT | Unlock a hint (costs points) |

**Supported challenge types:** `WEB`, `CRYPTO`, `FORENSICS`, `REVERSE`, `PWN`, `MISC`, `NETWORK`, `STEGANOGRAPHY`

**Create challenge example:**

```typescript
POST /api/v1/competitions/:cid/challenges
{
  "title": "XSS Basics",
  "description": "Find the reflected XSS vulnerability",
  "category": "WEB",
  "difficulty": "EASY",
  "points": 100,
  "flag": "FLAG{xss_r3fl3ct3d}",
  "isVisible": true,
  "isDynamic": false,
  "hints": [
    { "content": "Check the search parameter", "cost": 10 }
  ]
}
```

---

### Submissions

`/api/v1/submissions`

| Method | Endpoint | Guard | Rate Limit | Description |
|---|---|---|---|---|
| `POST` | `/submissions` | JWT | 5/min | Submit a flag |
| `POST` | `/submissions/public` | JWT | 5/min | Submit a flag for public challenges |
| `GET` | `/submissions/my` | JWT | — | Get current user's submission history |
| `GET` | `/submissions/competition/:id` | JWT | — | Get submissions for a competition |
| `GET` | `/submissions` | JWT + Admin | — | Admin view: all submissions |

**Rate limiting behavior:** After 3 incorrect attempts, a timeout is applied. IP address and user agent are captured for every submission for audit trails.

**Submit flag example:**

```typescript
POST /api/v1/submissions
Authorization: Bearer <token>
{
  "challengeId": "uuid-of-challenge",
  "competitionId": "uuid-of-competition",
  "flag": "FLAG{my_answer}"
}

// Response (correct)
{ "correct": true, "points": 100, "message": "Challenge solved!" }

// Response (incorrect)
{ "correct": false, "message": "Wrong flag. Try again." }
```

**Flag Storage (Security):** Flags are never stored in plaintext. They are hashed with SHA-256 + random salt using `CryptoService`:

```typescript
// CryptoService.hashFlag() stores { hash, salt }
// CryptoService.verifyFlag() uses constant-time comparison to prevent timing attacks
const { hash, salt } = cryptoService.hashFlag("FLAG{secret}");
const isValid = cryptoService.verifyFlag(submitted, storedHash, storedSalt);
```

---

### Teams

`/api/v1/teams`

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/teams` | Create a team (creator becomes captain, invite code generated) |
| `GET` | `/teams` | List all teams (paginated, filterable) |
| `POST` | `/teams/join` | Join team using invite code |
| `GET` | `/teams/:id` | Get team details and members |
| `PUT` | `/teams/:id` | Update team (captain only) |
| `DELETE` | `/teams/:id` | Disband team (captain only) |
| `DELETE` | `/teams/:id/leave` | Leave a team |
| `POST` | `/teams/:id/invite` | Invite a member (captain only) |
| `DELETE` | `/teams/:id/kick/:userId` | Kick a member (captain only) |
| `PATCH` | `/teams/:id/transfer` | Transfer captaincy |
| `GET` | `/teams/membership/:competitionId` | Get current user's team in a competition |

> All team endpoints require `JwtAuthGuard`. Team changes are blocked during active competitions.

---

### Leaderboard

`/api/v1/leaderboard`

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/leaderboard/competition/:id` | Individual rankings for a competition (`?limit=50`) |
| `GET` | `/leaderboard/competition/:id/team` | Team rankings for a competition |
| `GET` | `/leaderboard/global` | Global all-time leaderboard (`?limit=100`) |
| `GET` | `/leaderboard/user/:userId/competition/:id` | Get a specific user's rank in a competition |

Leaderboard data is served from a **Redis cache** via `LeaderboardCacheService` and updated via BullMQ scoring jobs after each successful submission.

---

### Instances (Docker)

`/api/v1/instances`

Dynamic challenges spin up isolated Docker containers per user.

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| `POST` | `/instances` | JWT | Create a Docker instance for a dynamic challenge |
| `GET` | `/instances` | JWT | List instances (filterable by challenge, user) |
| `GET` | `/instances/:id` | JWT | Get instance status |
| `DELETE` | `/instances/:id` | JWT | Stop and remove instance |
| `GET` | `/instances` | JWT + Admin | Admin: list all instances |

**Instance lifecycle:**

```
POST /instances → Docker build → Container start → PORT + FLAG env injected
                                                  ↓
                                   WebSocket broadcast: instance_created
                                   User receives: { hostname, port, flag (hashed) }
```

**DockerService** behavior:
- Clones challenge repository from GitHub
- Builds a Docker image named after the challenge
- Allocates a free port in the range `DOCKER_PORT_START–DOCKER_PORT_END` (default `30000–40000`)
- Injects a per-instance dynamically generated flag via `FLAG_SALT` env var
- Returns `{ hostname: "icca.africa", port: 31042 }`

**Create instance example:**

```typescript
POST /api/v1/instances
Authorization: Bearer <token>
{
  "challengeId": "uuid-of-dynamic-challenge"
}

// Response
{
  "id": "instance-uuid",
  "status": "RUNNING",
  "hostname": "icca.africa",
  "port": 31042,
  "expiresAt": "2026-06-01T11:00:00Z"
}
```

---

### Certificates

`/api/v1/certificates`

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| `POST` | `/certificates/request` | JWT | Request a certificate for a completed competition |
| `GET` | `/certificates/my` | JWT | Get all certificates earned by current user |
| `GET` | `/certificates/:id` | JWT | Get certificate by ID |
| `GET` | `/certificates/verify/:certNumber` | Public | Publicly verify a certificate by its number |
| `GET` | `/certificates` | JWT + Admin | Admin: list all certificates (filterable) |
| `PATCH` | `/certificates/:id/status` | JWT + Admin | Approve or revoke a certificate |

> Certificates require at least one solved challenge. They include `finalRank`, `totalScore`, and optionally `teamMembers` for team-based competitions.

---

### Announcements

`/api/v1/announcements`

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| `POST` | `/announcements` | JWT + Admin | Create a competition announcement |
| `GET` | `/announcements` | JWT | List announcements (filterable by `competitionId`, `type`) |
| `GET` | `/announcements/:id` | JWT | Get single announcement |
| `PUT` | `/announcements/:id` | JWT + Admin | Update announcement |
| `PATCH` | `/announcements/:id/pin` | JWT + Admin | Pin/unpin announcement |
| `DELETE` | `/announcements/:id` | JWT + Admin | Delete announcement |

Announcements are broadcast via WebSocket to all users in the competition room on creation.

---

### Learning Materials

`/api/v1/learning-materials`

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| `POST` | `/learning-materials` | JWT + (ADMIN/CREATOR/SUPERADMIN) | Create a learning material |
| `GET` | `/learning-materials` | JWT | List all materials (with visibility filtering by role) |
| `GET` | `/learning-materials/:id` | JWT | Get a learning material |
| `PUT` | `/learning-materials/:id` | JWT + (ADMIN/CREATOR/SUPERADMIN) | Update a material |
| `PATCH` | `/learning-materials/:id/visibility` | JWT + (ADMIN/CREATOR/SUPERADMIN) | Toggle visibility |
| `DELETE` | `/learning-materials/:id` | JWT + (ADMIN/CREATOR/SUPERADMIN) | Delete a material |

Materials include a `thumbnailUrl`, `linkUrl`, and an array of `resources: [{ label, url }]`.

---

### Docker Images

`/api/v1/docker-images`  
Requires `ADMIN` or `SUPERADMIN` role.

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/docker-images` | Register & build Docker image from a GitHub repository URL |
| `GET` | `/docker-images` | List all Docker image records (paginated) |
| `GET` | `/docker-images/:id` | Get a single image record |
| `PATCH` | `/docker-images/:id` | Update image metadata |
| `DELETE` | `/docker-images/:id` | Delete image record |

**Build behavior:**
- If an identical image already exists and `forceRebuild: false`, the existing record is returned immediately.
- If `forceRebuild: true`, the old record is torn down and a fresh build is kicked off asynchronously.

```typescript
POST /api/v1/docker-images
{
  "name": "web-basic-challenge",
  "githubUrl": "https://github.com/your-org/challenge-repo",
  "forceRebuild": false
}
```

---

### Users

`/api/v1/users`

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| `GET` | `/users/:id/profile` | Public | Get user public profile and stats |
| `POST` | `/users` | JWT + SuperAdmin | Create a user manually |
| `POST` | `/users/invite` | JWT + SuperAdmin | Invite a user by email |
| `GET` | `/users` | JWT + SuperAdmin | List all users (paginated, with role/status filters) |
| `GET` | `/users/:id` | JWT | Get user by ID |
| `PATCH` | `/users/me` | JWT | Update own profile |
| `PATCH` | `/users/:id/role` | JWT + SuperAdmin | Change user role |
| `PATCH` | `/users/:id/toggle-status` | JWT + SuperAdmin | Activate/deactivate user |

---

### Admin

`/api/v1/admin`  
Requires `ADMIN` or `SUPERADMIN`.

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/admin/dashboard` | Overview stats for admin's own competitions |
| `GET` | `/admin/players` | Players in admin's competitions (`?competitionId`, `?page`, `?limit`) |
| `PATCH` | `/admin/players/:userId/ban` | Ban/unban player (`?competitionId` required) |
| `GET` | `/admin/submissions` | Submissions for admin's competitions |

---

### SuperAdmin

`/api/v1/superadmin`  
Requires `SUPERADMIN` only.

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/superadmin/admins` | Create a new Admin account |
| `GET` | `/superadmin/admins` | List all Admin accounts |
| `GET` | `/superadmin/admins/:id` | Get Admin activity report |
| `PATCH` | `/superadmin/admins/:id/toggle-status` | Suspend or activate Admin |
| `DELETE` | `/superadmin/admins/:id` | Delete Admin account |
| `GET` | `/superadmin/audit-logs` | View system-wide audit logs |
| `GET` | `/superadmin/stats` | Platform-wide statistics |

---

### Dashboard

`/api/v1/dashboard`

Aggregated stats endpoint consumed by the admin dashboard frontend.

```typescript
GET /api/v1/dashboard?competitionId=optional-uuid

// Response shape
{
  stats: {
    totalUsers, activeCompetitions, todaySubmissions, activeTeams,
    userGrowth, competitionGrowth, submissionGrowth, teamGrowth
  },
  activityData: [{ date, submissions, users }],
  topUsers: [{ id, username, email, points, solvedChallenges, avatarUrl }]
}
```

---

### Files & Uploads

`/api/v1/files` and `/api/v1/uploads`

Supports both **local** file storage and **S3-compatible** storage (configured by env vars).

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/uploads` | Upload a file |
| `GET` | `/files/:id` | Retrieve file metadata |
| `DELETE` | `/files/:id` | Delete a file |

---

### Health / Monitoring

`/api/v1/health`

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/health` | Service health check (DB, Redis, queue connectivity) |
| `GET` | `/health/metrics` | Internal metrics (request counts, latencies) |

---

## WebSocket Events

**Namespace:** `/events`  
**Auth:** Clerk JWT passed in `handshake.auth.token`

### Connection

```javascript
const socket = io("http://localhost:3000/events", {
  auth: { token: clerkJwt }
});

socket.on("connected", ({ userId, username }) => {
  console.log("Connected to CTF real-time events");
});
```

### Client → Server (subscribe)

| Event | Payload | Description |
|---|---|---|
| `join_competition` | `{ competitionId }` | Join a competition room for live updates |
| `leave_competition` | `{ competitionId }` | Leave a competition room |

### Server → Client (broadcast)

| Event | Payload | Description |
|---|---|---|
| `connected` | `{ userId, username }` | Connection confirmation |
| `leaderboard_update` | `{ competitionId, leaderboard[] }` | Fired after each scored submission |
| `submission_result` | `{ correct, points, challengeId }` | Private to submitting user |
| `competition_update` | `{ competitionId, status }` | Status changes (ACTIVE, PAUSED, etc.) |
| `announcement` | `{ competitionId, title, content }` | New announcement broadcast |
| `instance_created` | `{ instanceId, userId, challengeId }` | Docker instance ready |
| `instance_destroyed` | `{ instanceId }` | Docker instance stopped |

---

## Background Jobs

Managed by **BullMQ** with Redis as the queue backend.

| Queue | Processor | Trigger | Description |
|---|---|---|---|
| `scoring` | `ScoringProcessor` | Correct flag submission | Calculate & persist score, update leaderboard cache |
| `notifications` | `NotificationProcessor` | Various events | Fan out in-app notifications to users |
| `cleanup` | `CleanupProcessor` | Scheduled (cron) | Remove expired Docker instances, stale data |

---

## Common Infrastructure

### Interceptors

- **LoggingInterceptor** — Logs every request method, URL, status code, and duration.
- **CacheInterceptor** — Redis-backed response caching for read-heavy endpoints (leaderboard, public competitions).
- **TransformInterceptor** — Wraps all successful responses in `{ data, statusCode, message }`.

### Pipes

- **ValidationPipe** — Strips unknown fields (`whitelist: true`), rejects non-whitelisted fields, auto-transforms query params.
- **TransformPipe** — Normalizes types before reaching handlers.

### Exception Filters

- **HttpExceptionFilter** — Formats HTTP errors into `{ statusCode, message, error, timestamp, path }`.
- **ValidationExceptionFilter** — Formats class-validator errors into readable field-level messages.

### Rate Limiting

Applied at controller method level via the `@RateLimit({ limit, ttl })` decorator, backed by Redis sliding windows:

```typescript
@Post()
@RateLimit({ limit: 5, ttl: 60 }) // 5 requests per 60 seconds per user
```

---

## Security Features

| Feature | Implementation |
|---|---|
| HTTP Security Headers | `helmet` middleware |
| Response Compression | `compression` middleware |
| CORS | Allowlist-based via `CORS_ORIGINS` env var |
| Flag Storage | SHA-256 + random salt (never stored in plaintext) |
| Constant-time comparison | `crypto.timingSafeEqual` to prevent timing attacks |
| Webhook Verification | Svix signature validation for Clerk webhooks |
| IP + UA Logging | Captured on every flag submission for audit |
| Rate Limiting | Redis sliding windows (5 flag submissions/min) |
| Input Validation | class-validator + class-transformer on all DTOs |

---

## Deployment

### Docker Compose (Development)

```bash
docker-compose up
```

### Docker Compose (Production)

```bash
docker-compose -f docker-compose.prod.yml up -d
```

### Kubernetes

Manifests are in `backend/k8s/`:
- `deployment.yaml` — App deployment
- `postgres.yaml` — PostgreSQL StatefulSet
- `redis.yaml` — Redis StatefulSet
- `ingress.yaml` — Ingress rules
- `hpa.yaml` — Horizontal Pod Autoscaler
- `secrets.yaml` — Secret template

### Required Environment Variables

```env
DATABASE_URL=postgresql://user:pass@host:5432/ctf
REDIS_URL=redis://host:6379
CLERK_SECRET_KEY=sk_...
CLERK_PUBLISHABLE_KEY=pk_...
CLERK_WEBHOOK_SECRET=whsec_...
JWT_SECRET=...
FLAG_SALT=<random-32-byte-hex>
DOCKER_HOSTNAME=icca.africa
DOCKER_PORT_START=30000
DOCKER_PORT_END=40000
FRONTEND_URL=https://your-platform.com
NODE_ENV=production
```
