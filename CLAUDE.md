# CSN Backend — CLAUDE.md

## Completed Services

| Service | Port | Status |
|---|---|---|
| identity-service | 3001 | Done |

## In-Progress Services

None.

## Pending Services

| Service | Port |
|---|---|
| profile-service | 3002 |
| tune-service | 3003 |
| lyrics-service | 3004 |
| voice-service | 3005 |
| video-service | 3006 |
| project-service | 3007 |
| chat-service | 3008 |
| voting-service | 3009 |
| feed-service | 3010 |
| rights-service | 3011 |
| payment-service | 3012 |

## Conventions & Patterns Established

### Source of truth
REST API contracts (`CSN REST APIs.pdf`) are authoritative. If the context PDF and the contracts conflict, always follow the contracts.

### UserId format
- PostgreSQL PK: UUID (`@id @default(uuid())`)
- Human-readable display ID: `USR` + zero-padded `sequenceNumber` (auto-increment). E.g. `USR100001`.
- The `displayId` is computed in the service layer: `"USR" + sequenceNumber.toString().padStart(6, "0")`. It is returned in all API responses as `userId`.

### Response envelope
All endpoints return the standard CSN envelope:
```json
{ "status": "SUCCESS", "message": "...", "data": {} }
{ "status": "ERROR",   "errorCode": "CSN-XXXX", "message": "..." }
```
Exceptions: register/login responses place top-level fields (userId, token, etc.) directly in the root per the contract.

### Tech stack (per service)
- Framework: NestJS + TypeScript
- ORM: Prisma (schema in `prisma/schema.prisma`)
- DB: PostgreSQL (each service owns its own DB)
- Auth: JWT via `@nestjs/jwt` + `@nestjs/passport` + `passport-jwt`
- Validation: `class-validator` DTOs with global `ValidationPipe`
- Secrets: `.env` (never committed); `.env.example` committed

### Folder structure (per service)
```
<service>/
├── src/
│   ├── main.ts               # Bootstrap + global pipes, port from config
│   ├── app.module.ts         # Root module
│   ├── config/configuration.ts
│   ├── prisma/prisma.service.ts
│   ├── shared/response.helper.ts
│   └── <feature>/
│       ├── <feature>.module.ts
│       ├── <feature>.controller.ts
│       ├── <feature>.service.ts
│       ├── <feature>.repository.ts
│       ├── dto/
│       └── jwt-auth.guard.ts (identity only; others import JWT guard from identity)
├── prisma/schema.prisma
├── Dockerfile
├── docker-compose.yml        # Local dev DB
├── .env                      # Local dev (not committed)
├── .env.example
└── package.json
```

### JWT guard
`JwtAuthGuard` in `identity-service/src/auth/jwt-auth.guard.ts` is the canonical guard. Other services will copy this pattern, validating tokens using the shared `JWT_SECRET`.

### Local development (identity-service)
```bash
cd csn-backend/identity-service
docker compose up -d          # start PostgreSQL on port 5432
npx prisma migrate dev        # run migrations
npm run start:dev             # start on port 3001
```

## Pending Decisions

- **OTP (mobile verification)**: Mentioned in context PDF but not in REST API contracts. Not built. Needs contract before implementation.
- **Social OAuth (Google login)**: Mentioned in context PDF but not in REST API contracts. Not built. Needs contract before implementation.

## What to Build Next

**profile-service (port 3002)** — endpoints:
- `GET /profiles/{userId}`
- `PUT /profiles/{userId}`
- `POST /profiles/{userId}/photo`
- `POST /users/{userId}/follow`
