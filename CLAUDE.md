# CLAUDE.md - AI Assistant Guide

## Project Overview

**Project Name**: Distributed Job Manager
**Repository**: sports-betting-analytics (DigitalOcean template-job-manager fork)
**Type**: Distributed task processing system using lease-based coordination
**Architecture**: Monorepo with npm workspaces
**Primary Use Case**: Demonstrates distributed coordination using leases in multi-instance environments

This is a production-quality example of a distributed job management system that coordinates work across multiple instances without traditional message queues, instead using a lease-based coordination pattern.

---

## Repository Structure

```
sports-betting-analytics/
├── task-service/          # Main task management service (Next.js 15)
│   ├── src/app/           # Next.js App Router structure
│   │   ├── api/           # REST API endpoints
│   │   ├── components/    # React UI components
│   │   └── lib/           # Shared utilities and clients
│   ├── prisma/            # Database schema and migrations
│   └── package.json       # Service dependencies
│
├── leases/                # Distributed lease coordination service (Next.js 15)
│   ├── src/app/           # Next.js App Router structure
│   │   ├── api/           # Lease management API
│   │   ├── components/    # React UI components
│   │   └── lib/           # LeaseReference client library
│   ├── prisma/            # Database schema and migrations
│   └── package.json       # Service dependencies
│
├── task-worker/           # Node.js worker for processing tasks
│   ├── main.js            # Worker entry point
│   └── package.json       # Worker dependencies
│
├── .do/                   # DigitalOcean deployment configuration
│   ├── app.yaml           # App Platform spec
│   └── deploy.template.yaml
│
├── .github/               # CI/CD workflows
│   └── workflows/         # GitHub Actions
│
└── package.json           # Root workspace configuration
```

---

## Technology Stack

### Frontend & Framework
- **Next.js 15.1.4** - React framework with App Router
- **React 19.0.0** - UI library
- **Tailwind CSS 3.4.1** - Utility-first CSS
- **SWR 2.3.0** - Data fetching with real-time updates

### Backend & Data
- **Node.js 22.x** - JavaScript runtime
- **Prisma 6.2.1** - ORM for database operations
- **PostgreSQL** - Primary database (2 separate instances)

### Testing
- **Jest 29.7.0** - Test framework
- **@testing-library/react 16.1.0** - Component testing
- **@testing-library/jest-dom 6.6.3** - DOM matchers

### Utilities
- **async-mutex 0.5.0** - Client-side synchronization
- **tailwind-scrollbar 3.1.0** - Custom scrollbars

---

## Core Concepts

### 1. Lease-Based Coordination

**Key Pattern**: Instead of using message queues or traditional database locks, this system uses time-based leases for distributed coordination.

**How it works**:
1. Worker requests a lease for a resource (e.g., a task)
2. Lease service grants exclusive access for 30 seconds
3. Worker periodically renews lease while processing (every 15s)
4. If worker crashes, lease auto-expires after 30s
5. Another worker can then claim the resource

**Benefits**:
- No single point of failure
- Automatic recovery from crashes
- Horizontal scalability
- Simple to reason about

**Implementation**: See `leases/src/app/lib/leases-client.js` for the `LeaseReference` class.

### 2. Heartbeat Monitoring

**Pattern**: Workers send periodic heartbeats to prove they're alive and still processing.

**Configuration**:
- Heartbeat interval: 15 seconds
- Lease duration: 30 seconds
- Grace period: ~15 seconds before timeout

**Flow**:
```
Worker claims task → Start processing → Send heartbeat every 15s
                                      ↓
                            Heartbeat renews lease for 30s
                                      ↓
                         If no heartbeat → Lease expires → Task available again
```

**Implementation**: See `task-service/src/app/api/tasks/[id]/heartbeat/route.js`

### 3. Task Lifecycle

```
SCHEDULED → ASSIGNED (lease acquired) → PROCESSING (heartbeats renewing)
                ↓                               ↓
         (409 conflict)                  COMPLETED/ABANDONED
```

**States**:
- `scheduled_at`: Task created
- `started_at`: Worker claimed task
- `last_heartbeat_at`: Last heartbeat received
- `must_heartbeat_before`: Deadline for next heartbeat
- `processed_at`: Task completed
- `processor`: Worker identifier holding the lease

---

## Database Schema

### Task Service Database (PostgreSQL)

**Table: `tasks`**
```sql
CREATE TABLE tasks (
    id                    SERIAL PRIMARY KEY,
    task_data             JSONB NOT NULL,           -- Task payload
    scheduled_at          TIMESTAMP DEFAULT NOW(),
    processor             VARCHAR(255),              -- Worker ID
    last_heartbeat_at     TIMESTAMP,
    must_heartbeat_before TIMESTAMP,                 -- Heartbeat deadline
    started_at            TIMESTAMP,
    processed_at          TIMESTAMP,                 -- NULL = not complete
    task_output           JSONB                      -- Result data
);
```

**Key Queries**:
- Next available task: `WHERE processed_at IS NULL FOR UPDATE LIMIT 1`
- Uses row-level locking to prevent race conditions

### Leases Database (PostgreSQL)

**Table: `leases`**
```sql
CREATE TABLE leases (
    id          SERIAL PRIMARY KEY,
    resource    VARCHAR(255) UNIQUE NOT NULL,      -- Resource identifier
    holder      VARCHAR(255),                       -- Current lease holder
    created_at  TIMESTAMP DEFAULT NOW(),
    renewed_at  TIMESTAMP,
    released_at TIMESTAMP,
    expires_at  TIMESTAMP                           -- Auto-expiry time
);
```

**Key Features**:
- Unique constraint on `resource` prevents double-leasing
- Uses PostgreSQL `FOR UPDATE` for atomic operations
- Uses `INTERVAL '30 seconds'` for expiration calculations

---

## API Endpoints

### Task Service (`task-service/src/app/api`)

| Method | Endpoint | Purpose | Key Response Codes |
|--------|----------|---------|-------------------|
| GET | `/api/tasks` | List all tasks | 200, 500 |
| POST | `/api/tasks/next` | Get next available task + acquire lease | 202, 400, 500 |
| GET | `/api/tasks/[id]` | Get task details | 200, 404, 500 |
| GET | `/api/tasks/started` | List in-progress tasks | 200, 500 |
| GET | `/api/tasks/processed` | List completed tasks | 200, 500 |
| PUT | `/api/tasks/[id]/heartbeat` | Renew task lease | 202, 409 (expired), 500 |
| PUT | `/api/tasks/[id]/complete` | Mark task complete | 202, 409 (not owner), 500 |

**Generator Endpoints**:
| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/generator/status` | Get generator status (STOPPED/STARTED/RUNNING) |
| POST | `/api/generator/start` | Start task generation with lease |
| DELETE | `/api/generator/stop` | Stop task generation |

### Lease Service (`leases/src/app/api`)

| Method | Endpoint | Purpose | Key Response Codes |
|--------|----------|---------|-------------------|
| GET | `/api/leases` | List all leases | 200, 500 |
| POST | `/api/leases` | Create/acquire lease | 201, 409 (conflict), 500 |
| GET | `/api/leases/[id]` | Get lease by ID | 200, 404, 500 |
| PUT | `/api/leases/renew` | Renew existing lease | 201, 404, 500 |
| DELETE | `/api/leases/[id]` | Release lease | 200, 404, 500 |
| GET | `/api/leases/active` | List active leases | 200, 500 |
| GET | `/api/leases/expired` | List expired leases | 200, 500 |
| GET | `/api/leases/released` | List released leases | 200, 500 |
| GET | `/api/leases/renewed` | List renewed leases | 200, 500 |

**Health Check**:
- GET `/api/healthz` - Returns `{status: "ok", timestamp: "..."}`

---

## Development Workflows

### Local Development Setup

**Prerequisites**:
- Node.js 22.x
- PostgreSQL (2 databases required)
- npm 9+

**Environment Setup**:

1. **Task Service** (`.env` in `task-service/`):
```bash
DATABASE_URL="postgresql://postgres:password@localhost:5432/task_manager"
PORT=3000
NEXT_PUBLIC_URL="http://localhost:3000"
SERVICE_LEASES_URL="http://localhost:3001/api/leases"
```

2. **Leases Service** (`.env` in `leases/`):
```bash
DATABASE_URL="postgresql://postgres:password@localhost:5432/leases"
PORT=3001
NEXT_PUBLIC_URL="http://localhost:3001"
```

3. **Task Worker** (`.env` in `task-worker/`):
```bash
TASK_SERVICE_URL="http://localhost:3000"
```

**Initial Setup**:
```bash
# Install all dependencies
npm install

# Run database migrations
cd task-service && npx prisma migrate deploy && cd ..
cd leases && npx prisma migrate deploy && cd ..

# Build all workspaces
npm run build
```

**Running Services**:
```bash
# Terminal 1: Task Service
cd task-service
npm run dev

# Terminal 2: Leases Service
cd leases
npm run dev

# Terminal 3: Task Worker
cd task-worker
node main.js
```

**Access Points**:
- Task Service UI: http://localhost:3000
- Leases Service UI: http://localhost:3001

### Common Development Commands

**Workspace Commands** (from root):
```bash
npm run build        # Build all workspaces
npm run test         # Test all workspaces
```

**Service-Specific Commands** (in task-service/ or leases/):
```bash
npm run dev          # Start dev server with hot reload
npm run build        # Build for production
npm start            # Start production server
npm run lint         # Run ESLint
npm test             # Run Jest tests
npm run test:watch   # Run tests in watch mode

# Prisma commands
npx prisma migrate dev         # Create new migration
npx prisma migrate deploy      # Apply migrations
npx prisma db push             # Push schema without migration
npx prisma studio              # Open database GUI
```

### Database Migrations

**Creating a new migration**:
```bash
cd task-service  # or leases
npx prisma migrate dev --name describe_your_change
```

**Applying migrations** (production):
```bash
npx prisma migrate deploy
```

**Reset database** (development only):
```bash
npx prisma migrate reset  # WARNING: Deletes all data
```

---

## Testing

### Running Tests

```bash
# All tests
npm test

# Specific workspace
cd task-service && npm test

# Watch mode
npm run test:watch

# With coverage
npm test -- --coverage
```

### Test Structure

**Test Files**:
- `*.test.js` - Jest unit tests
- Located alongside source files in `src/app/lib/`
- API route tests in `src/app/api/*/PUT.test.js` format

**Key Test Files**:
1. `task-service/src/app/lib/worker-manager.test.js` - WorkerManager class tests
2. `task-service/src/app/api/tasks/[id]/heartbeat/PUT.test.js` - Heartbeat API tests
3. `leases/src/app/lib/leases-client.test.js` - LeaseReference class tests

**Testing Patterns**:
```javascript
// Mock timers for interval testing
jest.useFakeTimers();

// Mock external dependencies
jest.mock('../lib/leases-client');

// Test async operations
await expect(promise).resolves.toBe(value);

// Test API routes
const response = await PUT(request, { params: { id: '1' } });
expect(response.status).toBe(202);
```

---

## Deployment

### DigitalOcean App Platform

**Configuration**: `.do/app.yaml`

**Architecture**:
- 3x task-service instances (1vCPU, 1GB RAM each)
- 2x leases service instances (1vCPU, 1GB RAM each)
- 3x task-worker instances (1vCPU, 1GB RAM each)
- 2x PostgreSQL databases (dev tier)
- Pre-deploy migration jobs

**Deployment**:
```bash
# Via GitHub
git push origin main  # Auto-deploys if connected

# Via doctl CLI
doctl apps create --spec .do/app.yaml
```

**Service Discovery**:
- Internal: Services communicate via internal hostnames (e.g., `http://leases:8080`)
- External: Ingress rules route public traffic
  - `/` → task-service
  - `/leases` → leases service

### CI/CD Pipeline

**GitHub Actions** (`.github/workflows/build-test.yaml`):

**On Pull Requests**:
1. Lint (task-service, leases in parallel)
2. Build (after lint passes)
3. Test (after build passes)

**Node Version**: 22.x
**Cache**: npm and Next.js build cache

---

## Key Conventions for AI Assistants

### 1. File Organization

**API Routes** (Next.js 13+ App Router):
- Location: `src/app/api/[resource]/route.js`
- Named exports: `GET`, `POST`, `PUT`, `DELETE`
- Return `Response` objects with `.json()` or status codes

**Components**:
- Location: `src/app/components/`
- Use `"use client"` directive for client-side React
- Props passed via standard React conventions

**Library Code**:
- Location: `src/app/lib/`
- Export reusable functions and classes
- Keep database logic separate from API logic

### 2. Error Handling

**Consistent Error Format**:
```javascript
return Response.json(
  { error: "Description of error" },
  { status: 500 }
);
```

**Status Codes**:
- `200` - Success (GET)
- `201` - Created (POST)
- `202` - Accepted (async operations)
- `400` - Bad request (missing parameters)
- `404` - Not found
- `409` - Conflict (lease already held, task expired)
- `500` - Server error

**Error Utilities**:
```javascript
import { stringifyError } from '@/app/lib/error';

try {
  // operation
} catch (error) {
  console.error(stringifyError(error));
  return Response.json({ error: error.message }, { status: 500 });
}
```

### 3. Database Operations

**Use Prisma Client**:
```javascript
import { prisma } from '@/app/lib/prisma-client';

// Simple queries
const tasks = await prisma.tasks.findMany();

// Raw SQL for complex operations (leases)
await prisma.$queryRaw`
  SELECT * FROM leases
  WHERE resource = ${resource}
  FOR UPDATE
`;
```

**Transactions** (when atomicity required):
```javascript
await prisma.$transaction(async (tx) => {
  // multiple operations
});
```

### 4. Client Libraries

**LeaseReference Pattern** (`leases/src/app/lib/leases-client.js`):
```javascript
import { LeaseReference } from '@/app/lib/leases-client';

const lease = new LeaseReference(resource, holder, baseUrl);
await lease.acquire();          // GET lease
lease.startAutoRenew(interval); // Auto-renew every N ms
await lease.release();          // DELETE lease
```

**TasksClient Pattern** (`task-service/src/app/lib/tasks-client.js`):
```javascript
import { getNextTask, heartBeat, complete } from '@/app/lib/tasks-client';

const task = await getNextTask(processor);
await heartBeat(task);
await complete(task, { result: "data" });
```

### 5. Environment Variables

**Naming**:
- `NEXT_PUBLIC_*` - Client-side variables (exposed to browser)
- `DATABASE_URL` - Prisma convention
- `SERVICE_*_URL` - Inter-service communication

**Access**:
```javascript
// Server-side
const url = process.env.SERVICE_LEASES_URL;

// Client-side
const publicUrl = process.env.NEXT_PUBLIC_URL;
```

### 6. React/Next.js Patterns

**SWR for Data Fetching**:
```javascript
import useSWR from 'swr';

const { data, error, isLoading } = useSWR(
  '/api/tasks',
  fetcher,
  { refreshInterval: 1000 } // Poll every 1s
);
```

**Client Components**:
```javascript
'use client';  // Required for useState, useEffect, etc.

export default function MyComponent() {
  // component code
}
```

### 7. Code Style

**Formatting**:
- 2-space indentation
- Single quotes for strings
- Semicolons required
- ESLint config in `eslint.config.mjs`

**Async/Await**:
```javascript
// Preferred
const result = await operation();

// Avoid
operation().then(result => { ... });
```

**Destructuring**:
```javascript
// Preferred
const { id, task_data } = task;

// Also good for environment variables
const { DATABASE_URL } = process.env;
```

### 8. Common Pitfalls

**❌ Don't**:
- Modify lease duration without updating heartbeat intervals
- Use database locks instead of leases for distributed coordination
- Forget to release leases on error conditions
- Skip migrations when changing schema
- Commit `.env` files to git

**✅ Do**:
- Always use transactions for multi-step database operations
- Test with multiple instances locally when changing coordination logic
- Handle 409 Conflict responses (indicates resource already leased)
- Use `FOR UPDATE` when selecting rows for modification
- Clean up resources in try/finally blocks

### 9. Adding New Features

**New API Endpoint**:
1. Create `src/app/api/[resource]/route.js`
2. Export named HTTP method functions
3. Add error handling with proper status codes
4. Add tests in `route.test.js`
5. Update this CLAUDE.md if it's a new pattern

**New Database Table**:
1. Update `prisma/schema.prisma`
2. Run `npx prisma migrate dev --name description`
3. Update `.do/app.yaml` if migration job needed
4. Commit migration files

**New Worker Type**:
1. Create new directory in workspace root
2. Add to `workspaces` in root `package.json`
3. Follow task-worker pattern for structure
4. Add to `.do/app.yaml` as worker component

### 10. Debugging Tips

**Check Lease Status**:
- Visit http://localhost:3001/leases in browser
- Look for expired leases that should be released
- Check `expires_at` timestamps

**Task Stuck?**:
- Check `must_heartbeat_before` timestamp
- Verify worker is running and sending heartbeats
- Look for 409 responses in logs (lease conflicts)

**Database Issues**:
```bash
# Open Prisma Studio
cd task-service  # or leases
npx prisma studio

# Check migrations
npx prisma migrate status
```

**Logs**:
```bash
# Task service logs
cd task-service && npm run dev

# Worker logs
cd task-worker && node main.js

# Look for:
# - "Acquired lease" messages
# - 409 status codes (conflicts)
# - Database errors
```

---

## Architecture Decisions

### Why Leases Instead of Message Queues?

**Leases Approach**:
- ✅ No external dependencies (Redis, RabbitMQ, etc.)
- ✅ Automatic cleanup via expiration
- ✅ Simple mental model
- ✅ Works with any SQL database
- ✅ Easy to debug (visible in database)

**Trade-offs**:
- ⚠️ Polling required (not push-based)
- ⚠️ Short delays for lease expiration
- ⚠️ Database load from renewals

**Best For**: Low to medium volume workloads where simplicity > raw throughput

### Why Next.js for Backend Services?

**Benefits**:
- Unified stack (React + API routes)
- Built-in API route handling
- Good developer experience
- Easy deployment (Vercel, DigitalOcean, etc.)
- Server and client code in one project

**Trade-offs**:
- Heavier than Express/Fastify
- More complex than needed for pure APIs

**Alternative**: For pure worker services, see `task-worker/` (plain Node.js)

---

## Resources

### Documentation
- [DigitalOcean Tutorial](https://www.digitalocean.com/community/tutorials/manage-multi-instance-environments-using-leases)
- [Next.js App Router](https://nextjs.org/docs/app)
- [Prisma Docs](https://www.prisma.io/docs)
- [SWR Docs](https://swr.vercel.app)

### Key Files to Understand
1. `leases/src/app/lib/leases-client.js` - Core lease logic
2. `task-service/src/app/lib/worker-manager.js` - Worker coordination pattern
3. `task-service/src/app/api/tasks/next/route.js` - Task distribution
4. `task-worker/main.js` - Worker implementation example
5. `.do/app.yaml` - Deployment architecture

### Quick Reference

**Most Common Tasks**:
```bash
# Start development
npm install && npm run build
cd task-service && npm run dev   # Terminal 1
cd leases && npm run dev          # Terminal 2
cd task-worker && node main.js   # Terminal 3

# Run tests
npm test

# Database changes
cd task-service && npx prisma migrate dev

# Deploy
git push origin main  # If connected to DigitalOcean
```

---

## Changelog

**2025-11-19**: Initial CLAUDE.md created with comprehensive codebase analysis

---

## Questions?

For AI assistants: If you encounter patterns not documented here, consider:
1. Checking existing similar implementations in the codebase
2. Reviewing test files for usage examples
3. Consulting the official Next.js or Prisma documentation
4. Following the existing error handling and response patterns
5. Updating this document with new patterns you discover

For humans: See the [DigitalOcean tutorial](https://www.digitalocean.com/community/tutorials/manage-multi-instance-environments-using-leases) or check the GitHub repository for issues and discussions.
