# SRS Ambiguity Detector

SRS Ambiguity Detector helps product, engineering, and QA teams review Software Requirements Specification (SRS) documents before implementation. Paste requirement text or upload a PDF, DOCX, or TXT file to find wording that may be vague, incomplete, subjective, or difficult to verify. Each finding includes its evidence, an explanation, a severity, and a suggested improvement.

The rule-based analysis is the source of truth. Optional AI enrichment can add an overview and rewrite suggestions using a provider key that the signed-in user supplies. Results are review aids: they do not replace a requirements review, and the detector can produce false positives or miss issues.

## Features

- Analyze pasted requirement text or one PDF, DOCX, or TXT document (up to 10 MiB per upload).
- Split text into requirements and check for vague quantities, subjective wording, missing criteria, unclear references, optional language, ambiguous operators, undefined terms, absolute claims, passive voice, missing constraints, and incomplete statements.
- Review evidence, explanations, recommendations, severity, scores, category summaries, and requirement health indicators.
- Save reports and revisit them in analysis history; explore aggregate activity in the dashboard.
- Create an account, verify email, manage profile and password, and control privacy, retention, export, and account deletion.
- Optionally configure a personal AI provider key for overviews and rewrite suggestions. Keys are encrypted by the backend and are not required to use deterministic analysis.
- Upload and download documents through the configured storage adapter.

## Technology

| Part | Main technologies |
| --- | --- |
| Web app | Next.js App Router, React, TypeScript, Tailwind CSS v4 |
| UI and charts | Motion, Lucide, Recharts |
| API | Python 3.11+, FastAPI, Pydantic v2 |
| Database | PostgreSQL, SQLAlchemy async, Alembic, asyncpg |
| File storage | Local filesystem or Supabase Storage |
| Email | Local development outbox or Resend |
| Tests and checks | Vitest, Testing Library, pytest, Ruff, mypy, ESLint, TypeScript |

## Project layout

```text
backend/       FastAPI application, migrations, and backend tests
frontend/      Next.js application, components, and frontend tests
scripts/       Repository verification and secret scanning
docker-compose.yml  Local PostgreSQL service
.env.example   Backend environment variable reference
README.md      Setup, usage, and deployment guide
```

## Run locally

### Requirements

- Node.js 20 or newer and npm.
- Python 3.11 or newer.
- Docker with Compose, or another PostgreSQL 16-compatible database.

### 1. Start PostgreSQL

From the project root:

```bash
docker compose up -d db
```

The default local database is `srs_ambiguity`, with user `srs` and password `changeme-local-only`. These defaults are for local development only. If you change them, update the database URLs below to match.

### 2. Configure and start the API

```bash
cd backend
python -m venv .venv
# macOS/Linux:
source .venv/bin/activate
# Windows PowerShell instead:
# .venv\Scripts\Activate.ps1
pip install -r requirements.txt -r requirements-dev.txt
Copy-Item .env.example .env   # PowerShell; on macOS/Linux use: cp .env.example .env
```

Edit `backend/.env`. Set at least:

```dotenv
APP_ENV=local
DEBUG=false
BACKEND_CORS_ORIGINS=http://localhost:3000
DATABASE_URL=postgresql+asyncpg://srs:changeme-local-only@localhost:5432/srs_ambiguity
DIRECT_DATABASE_URL=postgresql+asyncpg://srs:changeme-local-only@localhost:5432/srs_ambiguity
JWT_SECRET=<a-long-random-secret>
APP_BASE_URL=http://localhost:3000
EMAIL_PROVIDER=console
STORAGE_BACKEND=local
```

Generate a local JWT secret with `python -c "import secrets; print(secrets.token_urlsafe(32))"`. Leave optional integrations such as Resend, Turnstile, Sentry, Supabase Storage, and AI credentials unset unless you are configuring them. Never commit `.env` files or put backend secrets in frontend variables.

Apply the database migrations and start the API:

```bash
alembic upgrade head
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

The API health endpoint is `http://localhost:8000/api/v1/health/live`.

### 3. Configure and start the web app

In a second terminal:

```bash
cd frontend
npm ci
Copy-Item .env.example .env.local   # PowerShell; on macOS/Linux use: cp .env.example .env.local
npm run dev
```

The local frontend settings should include:

```dotenv
NEXT_PUBLIC_API_URL=http://localhost:8000/api/v1
NEXT_PUBLIC_SITE_URL=http://localhost:3000
```

Open [http://localhost:3000](http://localhost:3000). Local verification and password reset emails are written to the backend development outbox; use the links there to complete account flows. Keep that outbox private because messages contain account links.

## Tests and quality checks

The full repository gate is provided for Bash environments:

```bash
./scripts/verify.sh
```

It runs backend linting, formatting, type checks, tests and import checks; frontend linting, type checks, tests, formatting and a production build; then secret and dependency scans. Some database integration tests need a reachable test PostgreSQL instance. Focused checks can be run separately:

```bash
cd backend
python -m pytest -q
ruff check app tests alembic
mypy app

cd ../frontend
npm run lint
npm run typecheck
npm test
npm run build
```

## Deploy

The frontend and API are separate services. A common setup is Next.js on Vercel and FastAPI on a Python-capable host such as Render, Fly.io, Railway, AWS, Azure, or GCP. The API can also run on another managed container or VM service. Vercel does not host this FastAPI application.

1. Provision PostgreSQL and configure a private document-storage bucket. Use Supabase Storage or another implementation of the storage adapter for production; local storage is suitable only for development or a single instance with persistent disk.
2. Configure backend environment variables in the hosting provider's secret manager. Set `APP_ENV=production`, `DEBUG=false`, exact `BACKEND_CORS_ORIGINS`, `DATABASE_URL`, `DIRECT_DATABASE_URL`, a strong `JWT_SECRET`, `APP_BASE_URL`, Resend email settings, and production storage settings. Configure Turnstile and `ENCRYPTION_MASTER_KEY` when enabling those features. Keep all secrets backend-only.
3. Run database migrations as a release step from the backend environment: `cd backend && alembic upgrade head`. Use a direct database connection for migrations where the provider's pooled URL does not support DDL.
4. Start the API with `uvicorn app.main:app --host 0.0.0.0 --port $PORT` (adapt the port variable to the host). Configure HTTPS and a health check at `/api/v1/health/ready`.
5. Deploy `frontend/` as the Next.js project. Set `NEXT_PUBLIC_API_URL` to the public API base ending in `/api/v1`, `NEXT_PUBLIC_SITE_URL` to the public frontend origin, and any public Turnstile site key. Do not place secrets in `NEXT_PUBLIC_*` variables.
6. Confirm that CORS permits only the frontend origin, email verification works, cookies work across the chosen domains, uploads and downloads use private storage, and health checks report readiness.

Production also requires a secret manager, database and storage backups, and an independently protected copy of the Fernet `ENCRYPTION_MASTER_KEY`; losing this key makes saved user provider credentials unrecoverable. The current rate limiter is process-local, so operate one API instance unless you have accepted that rate limits are multiplied across instances. Configure provider-specific limits and monitoring before scaling.

## API and configuration references

API routes are grouped under `/api/v1`: authentication, analysis, documents, dashboard, AI providers, settings, privacy, and health. The backend and frontend `.env.example` files list available settings and safe defaults. Database schema and API details are implemented in the backend schemas, models, and endpoint modules.

## Known limitations

- Detection is heuristic and currently English-first; review the findings before changing a specification.
- Scanned image-only PDFs are not OCR processed.
- AI enhancement depends on the user's configured provider and can fail independently of deterministic analysis.
- Team workspaces, shared reports, multilingual detection, and PDF/DOCX report export are not included.
- Rate limits are local to one API process.

## Maintainer

Abhishek Maurya

## License

No license is currently included. Until the project owner adds one, reuse and redistribution rights are not specified.
