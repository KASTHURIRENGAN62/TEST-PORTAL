# SWAP 2K26 Mobile Test Portal

A secure, end-to-end multiple-choice examination portal for staff and professors
to conduct institutional tests, with built-in anti-cheat (malpractice)
monitoring, live tracking, auto-grading and Excel-based candidate/question
management.

> **Roles**
> - **Admin** (staff/professor) — manage candidates, questions, settings, monitor
>   live exams, review malpractice, view results & rankings.
> - **Candidate** — UID + password login, instructions screen, timed exam with
>   anti-cheat enforcement, and a results screen after submission.

---

## Table of contents

- [Tech stack](#tech-stack)
- [Repository layout](#repository-layout)
- [Features](#features)
  - [Admin](#admin)
  - [Candidate](#candidate)
  - [Malpractice / anti-cheat](#malpractice--anti-cheat)
- [Getting started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Backend setup](#backend-setup)
  - [Frontend setup](#frontend-setup)
- [Candidate authentication](#candidate-authentication)
- [Configuration](#configuration)
- [Scripts](#scripts)
- [Testing](#testing)
- [API reference](#api-reference)
  - [Auth](#auth)
  - [Exam flow (candidate)](#exam-flow-candidate)
  - [Admin](#admin)
- [Deployment](#deployment)
  - [Vercel (frontend + API rewrites)](#vercel)
  - [Render (single web service)](#render)
  - [Database](#database)
- [Security](#security)
- [License](#license)

---

## Tech stack

| Layer     | Technology |
| --------- | ---------- |
| Frontend  | React 18 + Vite 5 + TypeScript, React Router, Tailwind CSS, Axios |
| Backend   | Express 5 + TypeScript, Zod validation, Helmet, express-rate-limit |
| Database  | MongoDB (Mongoose ODM) — local or Atlas |
| Auth      | HTTP-only signed JWT cookie, bcrypt (12 rounds) password hashing |
| Excel     | `xlsx` for candidate & question import / result export |
| Testing   | Vitest + Supertest + `mongodb-memory-server` (no external DB needed) |

---

## Repository layout

```
api/            Vercel serverless entrypoint (hands /api to the Express app)
backend/        Express + TypeScript REST API (own package)
backend/src/
  controllers/  Request handlers (exam, admin, auth)
  models/       Mongoose schemas (Candidate, Question, ExamAttempt, MalpracticeLog, ...)
  routes/       Auth, exam, admin routers
  middleware/   auth (role guards), validate (zod), rate limits
  validators/   Zod schemas
  scripts/      Seed / migrate scripts (run with tsx)
  tests/        Vitest integration tests
backend/tests/  Integration test suite (server + in-memory Mongo)
frontend/       React + Vite SPA (own package)
frontend/src/
  pages/        Candidate (login, instructions, test, submitted) + Admin (dashboard, ...)
  components/   Reusable UI components
services/     API clients (axios)
scripts/      Root build helpers (build-static.mjs)
vercel.json   Vercel rewrites (SPA + /api passthrough)
render.yaml   Render blueprint for a single web service
```

---

## Features

### Admin

- **Dashboard** — live cards: registered candidates, exams completed, results
  cleared, passed, failed, average score, plus recent activity.
- **Candidates** — create, edit, delete, **import from Excel**, reset
  passwords, and a one-click **reset list** (removes candidates and their
  attempts/answers after the exam is over).
- **Questions** — create/edit/delete, **import from Excel**, per-question
  marks and activation.
- **Results** — auto-graded results, ranking list, score distribution and
  **Excel export**; one-click **result reset**.
- **Settings** — exam title, duration, question count, marks, violation
  thresholds, fullscreen/auto-terminate toggles, and a global exam enable/disable.
- **Live monitor** — per-candidate progress during an active exam.
- **Malpractice panel** — every recorded violation with severity and the
  authoritative minor-violation counter; actions to confirm/clear a suspicion or
  manually terminate an exam; reset of the violation log.

### Candidate

- UID + password login (rate-limited against brute force).
- Instructions screen (shows exam details and rules) before starting.
- Single-question-at-a-time interface: answer with A/B/C/D, clear answer,
  mark for review, answer palette for quick navigation.
- Auto-save of answers, review flags and last position on every change.
- Countdown timer with **automatic submission** when time expires.
- Submission summary; terminated exams report the termination status.

### Malpractice / anti-cheat

Monitoring is **best-effort based on measurable browser signals** — the portal
does not pretend to detect hardware buttons (Android Home, iPhone gesture,
power/volume, assistant apps). It records what the browser can actually report:

| Event type                  | Severity                          | Trigger |
| --------------------------- | --------------------------------- | ------- |
| `TAB_SWITCH`                | **MAJOR**                         | tab loses focus (visibility change) |
| `FULLSCREEN_EXIT`           | **MAJOR**                         | leaving fullscreen after entering it |
| `MULTIPLE_LOGIN_ATTEMPT`    | **MAJOR**                         | second concurrent exam start |
| `WINDOW_BLUR`               | **MINOR** (counted)               | window loses focus |
| `EXAM_SCREEN_HIDDEN`        | **MINOR** (counted)               | page becomes hidden (Home/app-switch) |
| `NAVIGATION_ATTEMPT`        | **MINOR** (counted)               | browser back/forward (popstate); exam route is re-pinned |
| `SHORTCUT_ATTEMPT`          | **MINOR** (counted)               | Ctrl/Cmd+C/V/X/U/S/R, F12, Ctrl/Cmd+Shift+I/J/C |
| `COPY_ATTEMPT` / `PASTE_ATTEMPT` / `CUT_ATTEMPT` | **MINOR** (counted) | clipboard actions on exam content |
| `CONTEXT_MENU_ATTEMPT`      | **MINOR** (counted)               | right-click / mobile long-press (menu is prevented) |

Behaviour highlights:

- **Counting is backend-authoritative.** The exam screen sends events; the API
  decides whether they apply, increments `minorViolationCount` atomically, and
  returns the real count. The UI never decides the count.
- **3 accepted minor violations automatically terminate** the exam (preserves
  saved answers, start time, duration and the violation log; final score 0).
  Warnings are broadcast with "Warning 1 of 3 / Warning 2 of 3" messages.
- **Deduplication** ensures one physical action counts once:
  - Frontend coalesces `blur` + `visibilitychange` + `pagehide` of a single
    screen-leave (~1.5 s) into one report; held keys (`event.repeat`) are
    ignored; a Ctrl shortcut vs. its clipboard event is counted once; text
    selection is blocked but never counted as a violation.
  - Backend dedupes the same event within 2 s and treats
    `WINDOW_BLUR`/`EXAM_SCREEN_HIDDEN` as one *screen-leave family*.
- After termination, answer/review/position calls are rejected with
  **"Examination has been terminated."**
- The exam page stays pinned to itself on back/forward navigation attempts, and
  normal admin navigation is unaffected.

---

## Getting started

### Prerequisites

- Node.js ≥ 18 (Node 22 recommended)
- MongoDB (local `mongod` or an Atlas cluster)

### Backend setup

```bash
cd backend
cp .env.example .env      # then fill in MONGODB_URI, JWT_SECRET, ADMIN_*, ...
npm install
npm run dev               # API on http://localhost:5002
```

The first boot runs `ensureAdmin` automatically (creates the admin account from
env when the database is empty).

### Frontend setup

```bash
cd frontend
npm install
npm run dev               # SPA on http://localhost:5174
```

Vite proxies `/api` to the backend (see `frontend/vite.config.ts`), so the
SPA and API share the same host in development; cookies flow normally.

---

## Candidate authentication

Candidates sign in with their **UID** and a **password** (mobile number is kept
for contact only and no longer authenticates).

- **Default password**: `Jmc` + the **last 4 digits of the UID**
  - `SWAP2K260001` → `Jmc0001`
  - `SWAP2K260100` → `Jmc0100`
- Passwords are **case-sensitive** and stored only as **bcrypt hashes**
  (`passwordHash`, 12 rounds). Plaintext and hashes are never returned or logged.
- UIDs are normalized (trimmed + uppercased) and must match
  `SWAP2K26` + 4 digits.
- Created on import/manual creation and restored to the default via the
  **Reset** action in the Candidates screen (the default password appears once
  in the response).
- Failed candidate logins are rate-limited per IP (10 failures / 15 min);
  successful logins are not counted.
- **Migration**: existing candidates can be switched to UID-derived passwords
  without touching their data:

  ```bash
  cd backend
  npm run migrate:candidate-passwords
  ```

---

## Configuration

Environment variables (see `backend/.env.example`):

| Variable       | Default                    | Purpose |
| -------------- | -------------------------- | ------- |
| `PORT`         | `5002`                     | API port |
| `NODE_ENV`     | `development`              | Runtime environment |
| `MONGODB_URI`  | `mongodb://127.0.0.1:27017/swap2k26` | MongoDB connection string |
| `JWT_SECRET`   | *(required)*               | Secret used to sign auth cookies (`openssl rand -hex 32`) |
| `COOKIE_NAME`  | `swap2k26_token`           | Auth cookie name |
| `TOKEN_TTL_HOURS` | `6`                     | Session length in hours |
| `FRONTEND_URL` | `http://localhost:5174`    | Allowed CORS origin |
| `ADMIN_EMAIL` / `ADMIN_PASSWORD` | *(required)* | Bootstrap admin credentials |
| `SEED_CANDIDATES` | `0`                    | `1` seeds demo candidates when running `npm run seed` |

---

## Scripts

| Command | Where | Purpose |
| --- | --- | --- |
| `npm run dev` | backend / frontend | Development servers |
| `npm run build` | both | TypeScript / Vite production build |
| `npm start` | backend | Run compiled server (`dist/server.js`) |
| `npm run typecheck` | backend | `tsc --noEmit` |
| `npm test` | backend | Vitest integration suite (in-memory MongoDB) |
| `npm run lint` | frontend | ESLint over the SPA |
| `npm run seed:admin` | backend | Create the admin account from env |
| `npm run seed` | backend | Seed demo candidates (env-flagged) |
| `npm run migrate:candidate-passwords` | backend | Adopt UID-derived passwords for all candidates |
| `npm run install:all` | root | Install both workspaces |
| `npm run build` | root | Build backend + frontend and emit static bundle |

---

## Testing

The backend ships a full integration suite using `supertest` against the real
Express app with an **in-memory MongoDB** (`mongodb-memory-server`) — no external
database or network is required:

```bash
cd backend
npm test
```

The suite covers auth and role guards, candidate import/validation, the full
exam lifecycle, scoring/ranking, admin resets (candidates, questions, results,
dashboard, malpractice), and the malpractice flow (counting, deduplication,
cross-type screen-leave dedup, three-minor auto-termination, and answer locking
after termination).

---

## API reference

All endpoints use cookies for authentication. JSON bodies in, JSON responses
out. Successful shapes follow `{ success, data }`; errors follow
`{ success, message }`.

### Auth

| Method | Path | Auth | Description |
| --- | --- | --- | --- |
| `POST` | `/api/auth/candidate/login` | – | Body `{ uid, password }` → sets candidate cookie |
| `POST` | `/api/auth/admin/login` | – | Body `{ email, password }` → sets admin cookie |
| `POST` | `/api/auth/logout` | – | Clears session cookie |

### Exam flow (candidate)

Role-gated to the candidate that owns the session.

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/api/exam/status` | Exam state: `NOT_STARTED` / `ACTIVE` / `SUBMITTED` / `TIMED_OUT` / `TERMINATED`, plus current attempt summary |
| `POST` | `/api/exam/start` | Create the attempt (blocked if one is already active) |
| `GET` | `/api/exam/questions` | Active question paper for the session |
| `GET` | `/api/exam/attempt` | Current attempt: saved answers, marks, last position, minor-violation counter |
| `PUT` | `/api/exam/answer` | Save/clear an answer `{ questionId, selectedAnswer, questionNumber, clear }` |
| `PUT` | `/api/exam/review` | Toggle "mark for review" `{ questionId, marked }` |
| `PUT` | `/api/exam/position` | Persist navigation position `{ questionNumber }` |
| `POST` | `/api/exam/violation` | Report a malpractice event `{ eventType, questionNumber, metadata }`; returns authoritative counter, status, warning message, or termination |
| `POST` | `/api/exam/submit` | Manual/final submission; triggers auto-grading |

### Admin

All under `/api/admin/*`, admin-only.

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/dashboard` | Stats and recent activity |
| `DELETE` | `/dashboard/reset` | Reset dashboard (requires `RESET DASHBOARD` confirmation action) |
| `GET/POST` | `/candidates`, `/candidates/:id` | List, create, fetch, update candidates |
| `POST` | `/candidates/import` | Bulk import from `.xlsx`/`.xls` |
| `POST` | `/candidates/:id/reset-password` | Restore default UID-derived password |
| `DELETE` | `/candidates/reset`, `/candidates/:id` | Reset whole list / delete one |
| `GET/POST/PUT/DELETE` | `/questions...` | Manage the question bank (+ `/questions/import`) |
| `DELETE` | `/questions/reset` | Clear the question bank |
| `GET` | `/results`, `/results/export` | Ranked results; Excel export |
| `DELETE` | `/results/reset` | Clear result/score data (blocked while an exam is active) |
| `GET` | `/malpractice` | Violation log with severity + minor counter |
| `PUT` | `/malpractice/:attemptId` | Action `CONFIRM` / `CLEAR` / `TERMINATE` |
| `DELETE` | `/malpractice/reset` | Clear the violation log (does not zero scores) |
| `GET/PUT` | `/settings` | Read / update exam settings |
| `GET` | `/live` | Live per-candidate progress |

---

## Deployment

The API and the built SPA are served from the **same origin** so the HTTP-only
auth cookie works with zero CORS setup.

### Vercel

- `vercel.json` rewrites `/(.*)` → `/index.html` (SPA) and `/api/:path*` → the
  serverless API entry (`api/`).
- Configure env vars (`MONGODB_URI`, `JWT_SECRET`, `ADMIN_EMAIL`,
  `ADMIN_PASSWORD`, `COOKIE_NAME`, `TOKEN_TTL_HOURS`, `FRONTEND_URL`) in the
  project settings.
- Backend tests and typecheck run automatically via the build pipeline on the
  root `prebuild`/`build` scripts.

### Render

`render.yaml` is a ready-made blueprint that deploys a single web service
serving both the Express API and `frontend/dist` static files:

- Connect the repo at **Render** → *New → Blueprint* (choose `render.yaml`).
- After first deploy, fill in the *secret* env vars marked `sync: false`
  (`MONGODB_URI`, `JWT_SECRET`, `FRONTEND_URL`, `ADMIN_EMAIL`,
  `ADMIN_PASSWORD`) in the dashboard.
- Health check: `GET /api/health`.

### Database

Use a MongoDB Atlas free cluster (or any reachable MongoDB). Set `MONGODB_URI`
to the connection string. On first boot the server creates the index/collections
and the bootstrap admin via `ensureAdmin`.

---

## Security

- Passwords stored as **bcrypt-only**; no plaintext persistence or logging.
- **HTTP-only, same-host** auth cookies; role-based guards on routes.
- **Rate limiting**: auth endpoints (50/15 min), a dedicated candidate-login
  limiter (10 failures/15 min), and general API limits.
- **Helmet + CORS** hardened in `backend/src/app.ts`; Zod validation on every
  body; strict UID/mobile/option enums; file uploads validated and capped at
  5 MB with Excel-extension checks.
- Malpractice counting is done on the server, so client-side tampering cannot
  under-report violations.

---

## License

ISC — see [LICENSE](LICENSE).