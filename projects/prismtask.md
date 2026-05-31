---
layout: page
permalink: /projects/prismtask/
title: PrismTask
---

## PrismTask

- Built a full-stack task manager for CEN3101 at the University of Florida
- Implemented with **Next.js 16**, **React 19**, **TypeScript**, and **Tailwind CSS**
- Supports authenticated task CRUD, priority filtering, completion toggles, due dates, and local persistence
- Uses HTTP-only session cookies and atomic JSON writes for a simple classroom-demo backend
- Team project by Prism Partners: Yibo Mao, Matthew Sama, Jiantong Yao, and Zaichen Hao
- [GitHub](https://github.com/Felix772/prism-todo-cpp)

> The repository is named `prism-todo-cpp` for historical scaffold reasons. The active application is the Next.js/TypeScript project under `frontend/prismtask`.

## Table of contents
- [Product Goal](#product-goal)
- [User Workflow](#user-workflow)
- [Architecture](#architecture)
- [Task API](#task-api)
- [Data & Auth](#data-auth)
- [Quality Gates](#quality-gates)
- [Trade-offs](#trade-offs)

---

## Product Goal

PrismTask is a classroom full-stack web app focused on a clean daily task workflow:

- log in with a seeded demo account
- create tasks with priority and optional due date
- filter tasks by priority
- edit task fields
- toggle completion
- delete finished or unwanted tasks
- persist changes locally between sessions

The UI pairs a functional task dashboard with a glass-morphism login screen and animated shard background.

---

## User Workflow

Demo login:

| Field | Value |
|---|---|
| Email | `demo@prismtask.com` |
| Password | `password123` |

After login, the user is redirected to `/tasks`. The dashboard supports:

- High / Medium / Low priority selection
- `All`, `High`, `Medium`, and `Low` filters
- editable title, description, priority, completion state, and due date
- logout through `DELETE /api/auth/login`

The app also exposes the same operations through REST endpoints, which makes the UI and backend behavior easy to smoke-test with `curl`.

---

## Architecture

The application is intentionally small and layered:

```text
Presentation
  app/page.tsx
  app/tasks/page.tsx
  app/globals.css

Application / API
  app/api/auth/login/route.ts
  app/api/tasks/route.ts
  app/api/tasks/[id]/route.ts

Data Access
  lib/auth.ts
  lib/taskStore.ts

Persistence
  data/tasks.json
  data/users.json
```

This structure keeps the UI, route handlers, auth helpers, and storage code separate enough to replace the JSON backend later without rewriting the dashboard.

---

## Task API

All task routes require the `prismtask_user` session cookie set by login.

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/auth/login` | Log in and set the session cookie |
| `DELETE` | `/api/auth/login` | Log out and clear the session cookie |
| `GET` | `/api/tasks` | List tasks, optionally filtered by priority |
| `POST` | `/api/tasks` | Create a task |
| `PATCH` | `/api/tasks/{id}` | Update a task |
| `DELETE` | `/api/tasks/{id}` | Delete a task |

Task shape:

```ts
interface Task {
  id: number;
  title: string;
  description: string;
  priority: 'High' | 'Medium' | 'Low';
  completed: boolean;
  dueDate: string;
  createdAt: string;
}
```

`id` and `createdAt` are server-owned and protected from client overwrite.

---

## Data & Auth

The project uses local JSON files for the demo:

- `data/users.json` stores the seeded demo account
- `data/tasks.json` stores task records
- both live under `frontend/prismtask/data`
- real user data is git-ignored

The data store writes atomically:

```text
tasks -> tasks.json.tmp -> rename to tasks.json
```

That means a crash mid-write should leave the previous JSON file intact rather than producing a half-written document.

Authentication is intentionally minimal:

- cookie name: `prismtask_user`
- HTTP-only session cookie
- `SameSite=Lax`
- 8-hour session lifetime
- unauthenticated task routes return `401`

---

## Quality Gates

Development commands from `frontend/prismtask`:

```bash
npm run dev
npm run build
npm start
npm run lint
npm run typecheck
```

The CI-oriented checks are:

- ESLint for project-wide linting
- `tsc --noEmit` for strict TypeScript validation
- Next.js production build for framework-level validation

The project guide also documents the team workflow:

- short-lived feature branches
- reviewed pull requests
- Conventional Commit prefixes
- squash merge into `main`
- CI must pass before merge

---

## Trade-offs

The README is explicit about classroom-demo shortcuts:

- passwords are stored as plain text and should be replaced with bcrypt before any real deployment
- JSON files are used instead of a database
- the `taskStore` module is the intended swap point for Prisma + PostgreSQL
- there is no write lock around `tasks.json`
- state-changing routes do not yet use CSRF tokens

These are acceptable for a course demo, but the page documents them because they show where the system boundary would need to move for production.
