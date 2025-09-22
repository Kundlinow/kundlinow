# kundlinow
This project is a reactive mobile and web application for astrology-based consultations. It allows users to connect with astrologers via chat, voice, or video, and includes a wallet system, real-time communication, and an admin panel.
Developer-focused, detailed guide to run, develop, test and deploy the reactive consultation platform (web + mobile + backend).


## Table of contents

1. [Overview](#overview)
2. [Project structure](#project-structure)
3. [Backend folder blueprint](#backend-folder-blueprint)
4. [Database schema (example)](#database-schema-example)
5. [REST API & WebSocket contract (examples)](#rest-api--websocket-contract-examples)
6. [Real-time architecture (Socket.IO + WebRTC)](#real-time-architecture-socketio--webrtc)
7. [Environment variables](#environment-variables)
8. [Local development & scripts](#local-development--scripts)
9. [Docker / docker-compose (example)](#docker--docker-compose-example)
10. [Kubernetes (minimal)](#kubernetes-minimal)
11. [CI / GitHub Actions example](#ci--github-actions-example)
12. [Testing strategy](#testing-strategy)
13. [Monitoring & logging](#monitoring--logging)
14. [Security best practices](#security-best-practices)
15. [Troubleshooting & tips](#troubleshooting--tips)
16. [Contributing](#contributing)
17. [License](#license)


## Overview

This repository contains a reactive consultation platform: web frontend, mobile app, backend microservice, and real-time subsystems for chat & calls. This README focuses on developer-centric instructions, folder structure, API contract, DB schema examples, and deployment samples so engineers can start coding quickly.


## Project structure

```
root/
 ├── server/                 # Backend service (Node.js: Express or NestJS)
 │   ├── src/
 │   │   ├── api/             # express routes / controllers wiring
 │   │   ├── controllers/     # HTTP controllers
 │   │   ├── services/        # business logic (transactions, wallet handling)
 │   │   ├── models/          # ORM models / schemas
 │   │   ├── repositories/    # DB access layer
 │   │   ├── sockets/         # socket.io handlers
 │   │   ├── jobs/            # background jobs (BullMQ / agenda)
 │   │   ├── middlewares/
 │   │   ├── utils/
 │   │   ├── config/
 │   │   └── tests/
 │   ├── migrations/
 │   ├── prisma/ or ormconfig/
 │   ├── Dockerfile
 │   └── package.json
 ├── client/                 # Web (React.js / Next.js)
 │   ├── src/
 │   │   ├── components/
 │   │   ├── hooks/
 │   │   ├── pages/ or routes/
 │   │   ├── services/       # REST + WebSocket clients
 │   │   └── tests/
 │   ├── public/
 │   └── package.json
 ├── mobile/                 # React Native (Expo)
 │   ├── app/
 │   ├── src/
 │   └── package.json
 ├── infra/                  # k8s manifests, terraform, helm charts
 ├── docs/                   # API docs, sequence diagrams
 ├── scripts/                # helper scripts (migrate, seed, build)
 ├── docker-compose.yml
 └── README.md
```



## Backend folder blueprint

A recommended minimal structure inside `server/src`:

```
src/
 ├── index.ts                # app bootstrap
 ├── config/                 # config loaders (.env, secrets)
 ├── api/
 │   └── routes.ts
 ├── controllers/
 │   ├── auth.controller.ts
 │   ├── consult.controller.ts
 │   └── wallet.controller.ts
 ├── services/
 │   ├── auth.service.ts
 │   ├── consult.service.ts
 │   └── wallet.service.ts
 ├── repositories/
 │   ├── user.repo.ts
 │   ├── consult.repo.ts
 │   └── wallet.repo.ts
 ├── models/                 # prisma schema / typeorm entities / mongoose models
 ├── sockets/
 │   └── chat.socket.ts
 ├── jobs/
 │   └── payout.job.ts
 ├── middlewares/
 │   └── auth.middleware.ts
 ├── utils/
 │   └── validators.ts
 └── tests/
```

### Example: wallet.service.ts (pseudocode for transactional safety)

```ts
async function chargeForSession(userId: string, amount: number, sessionId: string) {
  // Start DB transaction (depends on ORM)
  await db.transaction(async (trx) => {
    const wallet = await walletRepo.findByUserIdForUpdate(userId, trx);
    if (wallet.balance < amount) throw new Error('INSUFFICIENT_BALANCE');
    await walletRepo.decrementBalance(userId, amount, trx);
    await transactionRepo.create({ userId, amount, type: 'debit', sessionId }, trx);
    await consultRepo.markAsStarted(sessionId, trx);
  });
}
```

> Always use row-level locking (`SELECT ... FOR UPDATE`) or ORM equivalent when mutating balances.



## Database schema (example)

Relational store for users, sessions, wallets, and transactions. Use PostgreSQL for core data, MongoDB for chat history (optional). Example SQL (simplified):

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  email TEXT UNIQUE,
  phone TEXT UNIQUE,
  password_hash TEXT,
  role TEXT CHECK (role IN ('user','consultant','admin')),
  dob TIMESTAMP,                -- birth details if required
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE wallets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  balance NUMERIC(12,2) DEFAULT 0,
  currency VARCHAR(3) DEFAULT 'INR',
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  wallet_id UUID REFERENCES wallets(id) ON DELETE CASCADE,
  amount NUMERIC(12,2) NOT NULL,
  type TEXT CHECK (type IN ('credit','debit')),
  meta JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE consultations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  consultant_id UUID REFERENCES users(id),
  status TEXT CHECK (status IN ('pending','active','completed','cancelled')),
  started_at TIMESTAMPTZ,
  ended_at TIMESTAMPTZ,
  price NUMERIC(12,2),
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

For chat messages (if using MongoDB):

```js
// messages collection
{
  _id: ObjectId(),
  consult_id: "uuid",
  sender_id: "uuid",
  type: "text|image|system",
  body: "...",
  created_at: ISODate()
}
```



## REST API & WebSocket contract (examples)

### Authentication

**POST /auth/register**

Request:
```json
{ "name":"datamindai", "email":"datamindai@example.com", "password":"secret", "role":"user" }
```

Response:
```json
{ "id":"uuid", "token":"jwt.token.here", "user": { "id":"uuid", "name":"Datamindai" } }
```

**POST /auth/login**

Request:
```json
{ "email":"datamindai@example.com", "password":"secret" }
```

Response:
```json
{ "token":"jwt.token.here", "user":{ "id":"uuid", "role":"user" } }
```

### Consultations

**POST /consult/start** — initiates a consult (reserves consultant & locks funds)
Request:
```json
{ "userId":"uuid", "consultantId":"uuid", "durationMinutes":30, "price":250 }
```

Response:
```json
{ "consultId":"uuid", "status":"pending", "expiresAt":"2025-09-23T12:00:00Z" }
```

**GET /consult/:id** — fetch consultation state & participants

**POST /consult/:id/end** — finish session (release hold/settle payment)

### Wallet

**POST /wallet/recharge**

Request:
```json
{ "userId":"uuid", "amount":500, "paymentMethod":"razorpay", "receiptId":"rcp_123" }
```

Response:
```json
{ "transactionId":"uuid", "status":"success", "balance":750 }
```

### WebSocket (Socket.IO) events (client ↔ server)

- `connect` — handshake with token auth
- `join_consult` `{ consultId, userId }` — join room
- `leave_consult` `{ consultId }`
- `message` `{ consultId, message: { id, senderId, type, body, createdAt } }`
- `typing` `{ consultId, senderId, isTyping }`
- `call_offer` `{ consultId, offer }` — used to exchange SDP
- `call_answer` `{ consultId, answer }`
- `ice_candidate` `{ consultId, candidate }`
- `session_end` `{ consultId, endedBy }`



## Real-time architecture (Socket.IO + WebRTC)

1. Use Socket.IO for signaling, presence, chat messaging; keep events small and idempotent.
2. Use WebRTC for audio/video: exchange SDP offer/answer via Socket.IO events (`call_offer` / `call_answer`) and send ICE candidates.
3. STUN/TURN: run your own coturn or use a provider (Twilio, Xirsys). Example public STUN (only for testing): `stun:stun.l.google.com:19302`.
4. For production, configure TURN with credentials and relay enabled (required for mobile networks and NAT traversal).
5. Recordings: if recording calls, route media through a media server (Jitsi / Kurento / Janus) or use cloud provider features.



## Environment variables

Example `.env` (server):

```
NODE_ENV=development
PORT=5000

# DB
DATABASE_URL=postgres://username:password@postgres:5432/consultdb
MONGO_URL=mongodb://mongo:27017/consult-chat
REDIS_URL=redis://redis:6379

# Auth
JWT_SECRET=supersecretjwtkey
JWT_EXPIRES_IN=7d

# Payment
RAZORPAY_KEY=rzp_test_xxx
RAZORPAY_SECRET=xxx

# WebRTC / TURN (if using TURN service)
TURN_URL=turn:turn.example.com:3478
TURN_USERNAME=turnuser
TURN_PASSWORD=turnpass
```


## Local development & scripts

Recommended npm scripts (server/package.json):

```json
"scripts": {
  "dev": "ts-node-dev --respawn src/index.ts",
  "build": "tsc -p tsconfig.json",
  "start": "node dist/index.js",
  "migrate": "prisma migrate deploy",      // Or your migration tool
  "seed": "node prisma/seed.js",
  "test": "jest --runInBand"
}
```

Workflow to run locally (assumes docker-compose is used):

1. `docker-compose up -d postgres mongo redis`
2. `npm run migrate`
3. `npm run seed`
4. `npm run dev` (server)
5. Start client & mobile in parallel



## Docker / docker-compose (example)

`Dockerfile` (server):
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
RUN npm run build
CMD ["node", "dist/index.js"]
```

`docker-compose.yml` (minimal):
```yaml
version: '3.8'
services:
  server:
    build: ./server
    env_file: ./server/.env
    depends_on:
      - postgres
      - mongo
      - redis
    ports:
      - "5000:5000"
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: consultdb
    volumes:
      - pgdata:/var/lib/postgresql/data
  mongo:
    image: mongo:6
    volumes:
      - mongodata:/data/db
  redis:
    image: redis:7
volumes:
  pgdata:
  mongodata:
```



## Kubernetes (minimal deployment example)

`server-deployment.yaml` (very basic):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: consult-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: consult-server
  template:
    metadata:
      labels:
        app: consult-server
    spec:
      containers:
        - name: server
          image: yourrepo/consult-server:latest
          ports:
            - containerPort: 5000
          envFrom:
            - secretRef:
                name: consult-secrets
```

Add Service, Ingress, ConfigMaps and Secrets as required. Use HorizontalPodAutoscaler for scaling based on CPU / custom metrics.


## CI / GitHub Actions example

`.github/workflows/nodejs.yml`:
```yaml
name: CI
on: [push, pull_request]
jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 18
      - name: Install dependencies
        run: npm ci
      - name: Run tests
        run: npm test
      - name: Build
        run: npm run build
```

Include a workflow for container image build & push to registry on merge to `main` (use `docker/build-push-action`).



## Testing strategy

- Unit tests: Jest + ts-jest for services and utils.
- Integration tests: Supertest for API endpoints connected to a test database (use docker-compose test stack).
- E2E tests: Playwright or Cypress for UI flows (login, start consult, payment flow).
- Contract tests for WebSocket events (validate payload schemas with AJV).

Example test run command:

```bash
# run tests with a test DB
NODE_ENV=test npm run test
```



## Monitoring & logging

- Structured logging: Winston or Pino, JSON output (include requestId/correlationId).
- Tracing: OpenTelemetry for distributed traces (use for calls across microservices).
- Metrics: Prometheus for app metrics, Grafana for dashboards.
- Alerts: set alerts for high error rate, low balance failures, job queue backlog.


## Security best practices

- Validate & sanitize all user-provided input (AJV / Joi / Zod).
- Use HTTPS everywhere (Let’s Encrypt for ingress).
- Secure JWT secrets & rotate keys periodically.
- Use rate limiting (express-rate-limit) and IP throttling on critical endpoints (auth, payments).
- Store PII encrypted at rest if required by regulations.
- Use RBAC: distinguish `user`, `consultant`, and `admin` roles.
- PCI compliance: do not store raw card details; use payment provider tokens.



## Troubleshooting & tips

- **Audio/video issues**: check TURN server logs; mobile NATs often need TURN relay.
- **Payments failing in prod**: verify webhook endpoints are reachable (ngrok for dev).
- **Chat lag**: check redis performance and socket server CPU/network saturation.
- **Insufficient balance errors**: ensure transactions are atomic and idempotent—support retry semantics.



## Contributing

1. Fork the repository.
2. Create a feature branch: `git checkout -b feat/new-endpoint`
3. Run tests locally: `npm test`
4. Open PR with description & linking related issue.

Add changelog entry for non-trivial changes and reference database migrations.



## License

Released under the MIT License. See `LICENSE` file.



### Quick checklist for onboarding a new dev

- [ ] Create `.env` from `.env.example`
- [ ] Start infra (docker-compose up -d postgres mongo redis)
- [ ] Run `npm run migrate` and `npm run seed`
- [ ] Start server `npm run dev`
- [ ] Start web client `npm run dev` and mobile `npx expo start`
- [ ] Use Postman / Insomnia for testing endpoints and Socket.IO client (socket.io-client)



If you want, I can also:
- add a separate `README-server.md` with full code snippets for controllers/services,
- scaffold a `server` template repo (routes + sample controllers) and provide it as a downloadable zip,
- or include a `prisma/schema.prisma` example file.


