# Velozity Real-Time Client Project Dashboard

Full-stack implementation of the Velozity Global Solutions technical hiring assessment.

## Stack
- React + TypeScript + Vite
- Node.js + Express + TypeScript
- PostgreSQL + Prisma
- Socket.IO over WebSockets
- node-cron for scheduled overdue processing
- Zod server-side validation
- JWT access tokens + rotating refresh tokens in an HttpOnly cookie
- bcrypt password hashing

## Implemented requirements
- Admin, Project Manager and Developer roles with server-side authorization.
- Admin APIs for clients/users; Admin and PM can create projects assigned to clients.
- PMs can only manage projects they own.
- Developers can only retrieve/update their assigned tasks.
- Tasks contain title, description, assignee, status, priority, due date and persistent activity history.
- Status changes are written to PostgreSQL and broadcast through Socket.IO.
- Overdue tasks are flagged by an hourly node-cron job, not during page load.
- Project activity is delivered live to project rooms.
- Admin receives global activity; PM activity is limited to owned projects; developers see activity for assigned tasks.
- The API restores the latest 20 persisted activity events from PostgreSQL after reconnect/login.
- Presence is broadcast as a live Socket.IO count.
- Notifications are persisted, unread counts update through WebSockets, and users can mark one/all as read.
- Dashboard endpoints provide role-specific metrics.
- Task filters use URL query parameters: status, priority, from, to and projectId, making filtered views shareable.
- Every API input is validated server-side and errors use a consistent `{ error: { code, message, details? } }` structure.
- Secrets are environment variables only.
- Seed script creates 1 Admin, 2 PMs, 4 Developers, 3 clients, 3 projects, 18 tasks, two overdue tasks and existing activity.

## Architecture
```text
React/TS (Vercel) ──HTTP/JWT──> Express API ──Prisma──> PostgreSQL
       │                              │
       └──── Socket.IO WebSocket <────┘
                                      │
                               node-cron scheduler
```

Socket.IO was selected over native WebSocket because it supplies rooms, reconnect behavior and presence-friendly primitives while the application transport remains WebSocket. The application does not use long-polling or SSE for real-time features.

node-cron is sufficient for this deterministic hourly overdue scan. A durable queue would be preferable for a high-volume/retry-heavy workload, but would add infrastructure not required by this assessment.

### Authentication/token storage
The access token is short-lived and held in React memory only. The refresh token is a cryptographically random value stored hashed in PostgreSQL and delivered as an HttpOnly, SameSite cookie. Refresh rotates the token and issues a new short-lived access token. This avoids localStorage refresh-token exposure and satisfies the assessment's cookie requirement.

### Database/indexing
- `Project.ownerId`: PM ownership isolation.
- `Project.clientId`: client/project joins.
- `Task.projectId`, `assigneeId`, `status`, `priority`, `dueDate`: filters and dashboard queries.
- Composite `Task(projectId,status,priority,dueDate)`: common project task filtering/sorting.
- `Activity(projectId,createdAt)`: project feed retrieval.
- `Notification(userId,readAt,createdAt)`: unread badge and notification history.
- `RefreshToken(userId,expiresAt)`: token rotation/expiry cleanup.
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
- Access JWTs are short-lived and refresh tokens are HttpOnly cookies.
- JWT is verified on every protected HTTP endpoint and Socket.IO handshake.
- Authorization is enforced on the server.
- No credentials are hardcoded.
- Production errors do not expose stack traces.

## API
- `POST /api/auth/login`
- `POST /api/auth/refresh`
- `POST /api/auth/logout`
- `GET /api/auth/me`
- `GET/POST /api/clients`
- `GET /api/users`
- `GET/POST /api/projects`
- `GET/POST /api/tasks`
- `PATCH /api/tasks/:id/status`
- `GET /api/activity`
- `GET /api/notifications`
- `PATCH /api/notifications/:id/read`
- `POST /api/notifications/read-all`
- `GET /api/dashboard`

## Explanation (assessment field, 150–250 words)
The hardest part was keeping the activity feed both real-time and correctly role-filtered without turning the browser into the source of truth. Every status transition is first authorized and persisted in PostgreSQL, then emitted to a Socket.IO project room. This gives currently connected viewers an immediate event while the database remains the durable source for reconnects. On login/reconnect, the API independently queries the latest 20 events using the authenticated user's role: administrators receive global activity, project managers receive activity from owned projects, and developers receive events for tasks assigned to them. The same server-side authorization is applied to task reads and status updates, so changing a frontend control or token claims cannot expose another user's data.

Authentication uses a short-lived access token in memory plus a rotating refresh token in an HttpOnly cookie. This avoids putting long-lived credentials in localStorage. For overdue work, node-cron periodically transitions eligible tasks to Overdue and records that change as activity, rather than deriving it during rendering.

If I had more time, I would split the API into domain services/controllers and add integration tests around every authorization matrix and WebSocket event contract. I would also add a durable queue if scheduled work became high-volume.

## Assessment checklist
- [x] React + TypeScript
- [x] Node.js + Express
- [x] PostgreSQL + Prisma + relational schema
- [x] WebSocket real-time feed
- [x] node-cron background scheduler
- [x] server-side validation
- [x] structured errors
- [x] RBAC and project ownership
- [x] persistent activity logs
- [x] missed activity recovery
- [x] live presence
- [x] persisted notifications + live unread count
- [x] URL-query task filters
- [x] refresh token in HttpOnly cookie
- [x] required seed dataset

## Known limitation
External Vercel/API deployment requires the deployer's own hosting credentials and PostgreSQL connection string; those secrets are intentionally not committed.
