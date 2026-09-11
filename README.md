# Velozity Real-Time Client Project Dashboard

A production-minded full-stack implementation of the Velozity Global Solutions technical hiring assessment.

## Stack
- **Frontend:** React + TypeScript + Vite
- **Backend:** Node.js + Express + TypeScript
- **Database:** PostgreSQL + Prisma
- **Realtime:** Socket.IO over WebSockets
- **Background jobs:** node-cron
- **Validation:** Zod on every write/query endpoint
- **Auth:** JWT access token held in memory on the client; password hashes with bcrypt

## Features
- Role-based Admin / Project Manager / Developer access.
- Project ownership isolation for PMs.
- Tasks with status, priority, assignee, due date and persistent activity history.
- Scheduled overdue-task job; overdue is not calculated on page load.
- Live project activity feed and online presence with Socket.IO.
- Last 20 persisted activity events returned from PostgreSQL after reconnect.
- Role-scoped activity visibility.
- Live notification badge and notification dropdown.
- Admin/PM/Developer dashboards.
- Shareable task filters through URL query parameters.
- Structured API errors and server-side validation.
- Seed data: 1 Admin, 2 PMs, 4 Developers, 3 projects, 18 tasks and activity history.

## Architecture
```text
React/TS (Vercel) ──HTTP/JWT──> Express API ──Prisma──> PostgreSQL
       │                              │
       └──── Socket.IO WebSocket <────┘
                                      │
                               node-cron scheduler
```

Socket.IO was chosen over a raw WebSocket implementation because it provides rooms, reconnect handling, acknowledgements and presence-oriented primitives while still using WebSockets for the real-time channel. No polling or SSE is used for application realtime events.

`node-cron` is appropriate here because the only scheduled workload is a small deterministic overdue scan. A durable queue would be preferable for high-volume jobs or retries, but would add infrastructure that this assessment does not require.

### Indexing decisions
- `Project.ownerId` supports PM project isolation.
- `Task.projectId`, `Task.assigneeId`, `Task.status`, `Task.priority`, `Task.dueDate` support dashboards and task filters.
- `Activity.projectId`, `Activity.createdAt` supports project/global feeds.
- `Notification.userId`, `Notification.readAt` supports unread notification queries.
- Composite indexes cover the most common scoped feed/task queries.

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

Web: http://localhost:5173  
API: http://localhost:4000  
Health: http://localhost:4000/health

### Demo accounts
All seeded accounts use password `Password123!`.
- admin@velozity.local
- pm1@velozity.local
- pm2@velozity.local
- dev1@velozity.local
- dev2@velozity.local
- dev3@velozity.local
- dev4@velozity.local

## Deployment
The React frontend is Vercel-compatible. The API must run on a long-lived Node host that supports WebSocket upgrades because the assessment requires persistent Socket.IO connections. Set `VITE_API_URL` and `VITE_SOCKET_URL` in the frontend deployment and `DATABASE_URL`, `JWT_SECRET`, `CLIENT_ORIGIN` on the API host.

A single Vercel serverless function is deliberately not used for the Socket.IO backend because serverless request lifecycles are not a reliable fit for persistent WebSocket connections.

## Security decisions
- Secrets are read only from environment variables.
- Passwords are never stored in plaintext.
- JWT is validated server-side for every protected API route and Socket.IO handshake.
- Authorization is enforced server-side, not only by hiding frontend controls.
- Prisma relations and foreign keys enforce data integrity.
- Error responses use `{ error: { code, message, details? } }` and production responses never expose stack traces.

## Assessment checklist
- [x] React with TypeScript
- [x] Express + TypeScript
- [x] PostgreSQL + Prisma + relational foreign keys
- [x] Socket.IO WebSockets
- [x] node-cron overdue scheduler
- [x] server-side validation
- [x] structured error handling
- [x] role-based authorization
- [x] persistent activity log
- [x] live presence
- [x] missed-event recovery from DB
- [x] notifications and live unread count
- [x] URL query filters
- [x] required seed dataset

## API overview
`POST /api/auth/login`  
`GET /api/auth/me`  
`GET/POST /api/projects`  
`GET/POST /api/tasks`  
`PATCH /api/tasks/:id/status`  
`GET /api/activity`  
`GET/PATCH/POST /api/notifications`  
`GET /api/dashboard`

## License
Assessment implementation for Velozity Global Solutions.
