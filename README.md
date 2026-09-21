# SWAP 2K26 Mobile Test Portal

Monorepo-style layout with two workspaces:

- **`backend/`** — Express + TypeScript + MongoDB (Mongoose) REST API
- **`frontend/`** — React + Vite + TypeScript single-page app

## Quick start

### Backend

```bash
cd backend
cp .env.example .env   # fill in MONGODB_URI, JWT_SECRET, etc.
npm install
npm run dev            # boots API on PORT (default 5002)
```

### Frontend

```bash
cd frontend
npm install
npm run dev
```

The app proxies `/api` to the backend (see `vite.config.ts`).

## Candidate authentication

Candidates sign in with their **UID** and a **password** (mobile number is kept for contact only and no longer authenticates).

- **Default password**: `Jmc` + the **last 4 digits of the UID**
  - `SWAP2K260001` → `Jmc0001`
  - `SWAP2K260100` → `Jmc0100`
- Passwords are **case-sensitive** and stored only as **bcrypt hashes** (`passwordHash`, 12 rounds). Plaintext and hashes are never returned or logged.
- UIDs are normalized (trimmed + uppercased) and must match `SWAP2K26` + 4 digits.
- Created on import/manual creation and restored to the default via the **Reset** action in the Candidates screen (the default password is shown once in the response).
- Failed candidate logins are rate-limited per IP (10 failures per 15 min); successful logins are not counted.
- **Migration**: existing candidates can be switched to UID-derived passwords with

```bash
cd backend
npm run migrate:candidate-passwords
```

This updates only `passwordHash`, leaving all candidate info, exam attempts and results untouched.

## Useful scripts

| Command | Where | Purpose |
| --- | --- | --- |
| `npm run dev` | backend / frontend | Development servers |
| `npm run build` | both | TypeScript/Vite production build |
| `npm start` | backend | Run compiled server (`dist/server.js`) |
| `npm test` | backend | Vitest suite (uses in-memory MongoDB; no external DB needed) |
| `npm run typecheck` | backend | `tsc --noEmit` |
| `npm run seed:admin` | backend | Create the admin account from env |
| `npm run seed` | backend | Seed demo candidates (env-flagged) |
| `npm run migrate:candidate-passwords` | backend | Set UID-derived passwords for all candidates |

## API notes

- `POST /api/auth/candidate/login` — body `{ uid, password }`; sets an HTTP-only auth cookie.
- `POST /api/auth/admin/login` — admin auth (email + password) is unchanged.
- Admin endpoints under `/api/admin/*` manage candidates, questions, settings, exam flow and results/ranking (Excel export included).
- Candidate exam endpoints under `/api/exam/*` are cookie-authenticated and role-gated.

## Security

- Passwords stored as bcrypt-only; no plaintext persistence.
- HTTP-only, same-host auth cookies; role-based guards on routes.
- Rate limiting on auth (`/api/auth`, 50/15 min) and API calls, plus a dedicated candidate-login limiter.
- Helmet + CORS configured in `backend/src/app.ts`.
