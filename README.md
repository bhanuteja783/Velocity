# Velozity Real-Time Client Project Dashboard

A production-minded full-stack implementation of the Velozity Global Solutions technical hiring assessment.

## Stack
- React + TypeScript + Vite
- Node.js + Express + TypeScript
- PostgreSQL + Prisma
- Socket.IO over WebSockets
- node-cron for scheduled overdue processing
- Zod server-side validation
- JWT authentication + bcrypt password hashing

## Implemented requirements
- Admin, Project Manager and Developer roles with server-side authorization.
- PMs can only manage projects they own.
- Tasks contain title, description, assignee, status, priority, due date and persistent activity history.
- Status changes are written to PostgreSQL and broadcast through Socket.IO.
- Overdue tasks are flagged by an hourly node-cron job, not during page load.
- Project activity is delivered live to project rooms.
- Admin receives global activity; PM activity is limited to owned projects; developers see activity for assigned tasks.
- The API restores the latest 20 persisted activity events from PostgreSQL after reconnect/login.
- Presence is broadcast as a live Socket.IO count.
- Notifications are persisted, unread counts update through WebSockets, and users can mark one/all as read.
- Dashboard endpoints provide role-specific metrics.
- Task filters are represented by URL query parameters: status, priority, from, to and projectId.
- Every API input is validated server-side and errors use a consistent `{ error: { code, message, details? } }` structure.
- Secrets are environment variables only.
- Seed script creates 1 Admin, 2 PMs, 4 Developers, 3 projects, 18 tasks, two overdue tasks and existing activity.

## Architecture
```text
React/TS (Vercel) ──HTTP/JWT──> Express API ──Prisma──> PostgreSQL
       │                              │
       └──── Socket.IO WebSocket <────┘
                                      │
                               node-cron scheduler
```

Socket.IO was selected over native WebSocket because it supplies rooms, reconnect behavior and presence-friendly primitives while the application transport remains WebSocket. The application does not use long-polling or SSE for its real-time features.

node-cron is sufficient for this deterministic hourly overdue scan. A durable queue would be preferable for a high-volume/retry-heavy workload, but would add infrastructure not required by this assessment.

### Database/indexing
- `Project.ownerId`: PM ownership isolation.
- `Task.projectId`, `assigneeId`, `status`, `priority`, `dueDate`: filters and dashboard queries.
- Composite `Task(projectId,status,priority,dueDate)`: common project task filtering/sorting.
- `Activity(projectId,createdAt)`: project feed retrieval.
- `Notification(userId,readAt,createdAt)`: unread badge and notification history.
- All relations use PostgreSQL foreign keys with explicit delete behavior.

## Local setup
Prerequisites: Node 20+, npm 10+, Docker.

```bash
cp .env.example .env
npm install
npm run db:up
npm run db:generate
npm run db:migrate
npm run db:seed
npm run dev
```

Frontend: http://localhost:5173  
API: http://localhost:4000  
Health: http://localhost:4000/health

### Seed accounts
Password for all accounts: `Password123!`
- admin@velozity.local
- pm1@velozity.local
- pm2@velozity.local
- dev1@velozity.local
- dev2@velozity.local
- dev3@velozity.local
- dev4@velozity.local

## Deployment
The React/Vite frontend is Vercel-compatible. The Socket.IO API should run on a long-lived Node host that supports WebSocket upgrades. Configure `VITE_API_URL` and `VITE_SOCKET_URL` on the frontend, and `DATABASE_URL`, `JWT_SECRET`, `CLIENT_ORIGIN` and `PORT` on the API.

A serverless-only Vercel backend is intentionally avoided for the WebSocket server because persistent Socket.IO connections need a host that supports long-lived WebSocket upgrades.

## Security
- Passwords are bcrypt hashes, never plaintext.
- JWT is verified on every protected HTTP endpoint and Socket.IO handshake.
- Authorization is enforced on the server.
- No credentials are hardcoded.
- Production errors do not expose stack traces.

## API
- `POST /api/auth/login`
- `GET /api/auth/me`
- `GET/POST /api/projects`
- `GET/POST /api/tasks`
- `PATCH /api/tasks/:id/status`
- `GET /api/activity`
- `GET /api/notifications`
- `PATCH /api/notifications/:id/read`
- `POST /api/notifications/read-all`
- `GET /api/dashboard`

## Assessment checklist
- [x] React + TypeScript
- [x] Node.js + Express
- [x] PostgreSQL + Prisma + relational schema
- [x] WebSocket real-time feed
- [x] node-cron background scheduler
- [x] server-side validation
- [x] structured errors
- [x] RBAC and project ownership
- [x] persisted activity logs
- [x] missed activity recovery
- [x] live presence
- [x] persisted notifications + live unread count
- [x] URL-query task filters
- [x] required seed dataset

## Known limitation
The repository contains the complete runnable application and deployment configuration, but an external Vercel/API deployment requires the deployer's own hosting credentials and PostgreSQL connection string; those secrets are intentionally not committed.
