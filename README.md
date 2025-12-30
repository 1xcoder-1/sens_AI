# AI Career Coach (sens_AI)

An AI-powered career development platform to build better resumes, generate tailored cover letters, and practice for interviews — powered by Google Gemini, Inngest background jobs, and Prisma + PostgreSQL.

Fast preview:
- Next.js 15 (App Router) + React 19
- Auth: Clerk
- AI: Google Gemini
- Background jobs: Inngest
- ORM: Prisma (PostgreSQL)
- Styling: Tailwind CSS & shadcn-like components
- Docker-ready with docker-compose (Postgres + Adminer)

---

Table of contents
- Features
- Tech stack
- Quickstart — local development
- Database & Prisma
- Environment variables (.env)
- Docker & docker-compose
- Deployment notes (Vercel / Docker)
- Background jobs (Inngest) & AI integration
- API & important routes
- Project structure
- Contributing, security & code of conduct
- Troubleshooting

## Features
- AI-powered resume builder and improvement suggestions
- Generate personalized cover letters using Google Gemini
- Interview preparation: quizzes, timers, scoring and feedback
- Industry insights (salary ranges, trends, recommended skills)
- User onboarding, profile, and progress tracking
- Background job processing for heavy tasks (Inngest)
- Production-ready Dockerfile, and docker-compose for local full-stack dev

## Tech stack
- Frontend: Next.js 15 (App Router), React 19, Tailwind CSS, Framer Motion
- Backend & infra: Node.js (18+), Prisma, PostgreSQL, Inngest
- Auth: Clerk
- AI: Google Gemini API
- Dev tooling: Prisma CLI, Turbopack (dev), ESLint, Tailwind
- Container: Docker, docker-compose

---

## Quickstart — Local development

Prerequisites
- Node.js 18+ (Dockerfile uses node:18-alpine)
- npm (or yarn)
- PostgreSQL (you can use docker-compose to run one)
- A Clerk account (for auth) and Google Gemini API key (for AI features)

1) Clone
```bash
git clone https://github.com/1xcoder-1/sens_AI.git
cd sens_AI
```

2) Install dependencies
```bash
npm install
# or if you prefer clean installs:
# npm ci
```

Note: package.json runs `prisma generate` as a postinstall step.

3) Create a local env file
Copy the template (project contains `local.env` as a reference):
```bash
cp local.env .env.local
# Open .env.local and fill the required values
```

Minimum required environment variables (see next section for full list).

4) Initialize database & Prisma migrations (development)
If you use a local Postgres (ensure DATABASE_URL in .env.local points to it):

```bash
npx prisma migrate dev --name init
# This will create/migrate the DB and generate the Prisma client
```

Note for production: use `npx prisma migrate deploy`.

5) Start dev server
```bash
npm run dev
```
Open http://localhost:3000

If you are using Turbopack, development runs as configured by `next dev --turbopack`.

---

## Database & Prisma

- Prisma client is generated via `npx prisma generate` (postinstall). The Prisma client instance is reused via a global variable in `lib/prisma.js`.
- Main models (see `prisma/schema.prisma`):
  - User: profile + Clerk mapping + industry + relations (Resume, CoverLetter, Assessment)
  - Resume: single-per-user markdown content, ATS score, feedback
  - CoverLetter: generated content, job/company metadata, status (draft/completed)
  - Assessment: quiz results, questions, score, improvement tips
  - IndustryInsight: salary ranges, growth/demand, top skills, trends

Run migrations:
- Dev:
  ```bash
  npx prisma migrate dev
  ```
- Prod:
  ```bash
  npx prisma migrate deploy
  ```

Open Prisma Studio:
```bash
npx prisma studio
```

---

## Environment variables

Copy `local.env` to `.env.local` and fill values. Important variables used by the application:

- DATABASE_URL — e.g. postgresql://postgres:password@localhost:5432/ai_career_coach
- NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY — Clerk publishable key
- CLERK_SECRET_KEY — Clerk API secret
- NEXT_PUBLIC_CLERK_SIGN_IN_URL — typically /sign-in
- NEXT_PUBLIC_CLERK_SIGN_UP_URL — typically /sign-up
- NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL — e.g. /onboarding
- NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL — e.g. /onboarding
- GEMINI_API_KEY — Google Gemini API key for AI generation
- NEXT_PUBLIC_APP_URL — e.g. http://localhost:3000

Example:
```
DATABASE_URL=postgresql://postgres:password@db:5432/ai_career_coach
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_...
CLERK_SECRET_KEY=sk_...
GEMINI_API_KEY=...
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

Security note: never commit real keys to git. Use CI secrets or deployment platform variable stores.

---

## Docker & docker-compose

There is a Dockerfile and docker-compose.yml configured for local development (app + Postgres + Adminer).

Build image (production-styled build):
```bash
docker build -t ai-career-coach .
```

Run the container:
```bash
docker run -p 3000:3000 \
  -e DATABASE_URL="postgresql://postgres:password@host:5432/ai_career_coach" \
  -e GEMINI_API_KEY="..." \
  -e NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="..." \
  -e CLERK_SECRET_KEY="..." \
  ai-career-coach
```

Run full stack with docker-compose (recommended for local dev):
```bash
# starts: app, postgres, adminer
docker-compose up --build
```

Notes:
- docker-compose uses volumes to mount the app during dev. If you prefer a fresh image without volumes for prod, remove the volumes or use a dedicated Dockerfile build.
- Adminer is available at http://localhost:8080 for DB browsing (user: postgres, password as set in docker-compose).

---

## Deployment

Vercel (recommended for Next.js):
- Push repository to GitHub.
- Create a Vercel project and connect the repo.
- Set environment variables in Vercel dashboard (all variables from .env.local).
- Set up production database (managed Postgres) and Prisma migrations:
  ```bash
  # in CI or via SSH:
  npx prisma migrate deploy
  ```
Docker / self-hosted:
- Use the Dockerfile to build and run the container.
- Ensure DATABASE_URL points to a managed Postgres instance.
- Run migrations as part of your deploy pipeline (use `npx prisma migrate deploy`).

Production best-practices:
- Use secrets manager for API keys (Clerk, Gemini).
- Use TLS and secure headers.
- For heavy AI jobs, rely on Inngest/AWS or background workers to avoid web-timeout issues.

---

## Background jobs (Inngest) & AI integration

- Inngest is configured under `app/api/inngest/route.js` and a client exists in `lib/inngest/`.
- Heavy or longer-running AI tasks (e.g., generating industry insights) are performed in background functions and triggered via Inngest.
- Google Gemini is used for content generation (resume improvements, cover letters, interview questions). Ensure `GEMINI_API_KEY` is set.

If you want to run Inngest locally or connect to Inngest cloud:
- Configure the Inngest client per your provider account and environment variables.

---

## API & Important Endpoints

Application actions are implemented under `actions/` (server-side helpers called by pages). Representative endpoints/features:
- Inngest webhook: POST/GET/PUT at /api/inngest/
- Typical app actions (server-side callers, some may be exposed as API routes):
  - getUserOnboardingStatus
  - getResume / save resume / improve resume (AI)
  - getCoverLetters / create cover letter / delete
  - getAssessments / create assessment / submit answers
- Auth-protected pages are handled by Clerk provider in `app/layout.js`.

For background job functions see `lib/inngest/function.js` and `lib/inngest/client.js`.

---

## Project structure (high-level)
- app/ — Next.js App Router pages & nested layouts (landing page, dashboard, resume, interview, ai-cover-letter)
- app/api/ — server API routes (Inngest)
- components/ — UI components, reusable shadcn-like components
- actions/ — server functions to read/write data and orchestrate actions
- lib/ — helpers (Prisma client, Inngest client, utilities)
- prisma/ — schema + migrations
- public/ — static assets
- Dockerfile / docker-compose.yml — container setup
- local.env — env var reference

---

## Contributing & Code of Conduct
- Please read CONTRIBUTING.md and CODE_OF_CONDUCT.md in the repository before contributing.
- Typical workflow:
  - Fork → feature branch → PR with description → link relevant issue/changes
  - Tests & linting before merging
- SECURITY.md has instructions for reporting vulnerabilities.

---

## Troubleshooting & common pitfalls

- Node version mismatch:
  - Use Node 18.x (Dockerfile uses node:18-alpine). Using too-new or too-old Node might break build.
- Prisma connection errors:
  - Verify DATABASE_URL, ensure Postgres is reachable and migrations applied.
  - On `prisma generate` errors, re-run `npx prisma generate` and ensure the client is installed.
- Clerk auth not working:
  - Ensure both publishable and secret keys are set and that Clerk is configured to allow your NEXT_PUBLIC_APP_URL as an allowed origin.
- Gemini / AI errors:
  - Check GEMINI_API_KEY and API quotas.
- Dev hot reload / Prisma duplicates:
  - lib/prisma.js uses a global cached PrismaClient in dev to prevent connection storms. Do not change unless you understand hot-reload implications.

---

## Tests & CI
- The repository references testing in documentation, but there are no included test scripts in package.json. Recommended:
  - Add Jest + React Testing Library for unit/integration
  - Add Cypress for end-to-end
  - Add GitHub Actions to run lint/test/migrations on PRs

Suggested scripts (example):
```json
"scripts": {
  "dev": "next dev --turbopack",
  "build": "next build",
  "start": "next start",
  "lint": "next lint",
  "test": "jest"
}
```

---

## Helpful commands (summary)
- Install deps: npm install
- Generate Prisma client: npx prisma generate
- Run migrations (dev): npx prisma migrate dev
- Run dev server: npm run dev
- Build: npm run build
- Start (production): npm start
- Docker-compose local: docker-compose up --build

---

If you'd like, I can:
- add a .env.example file with placeholders,
- generate a minimal GitHub Actions CI pipeline for migrations,
- or produce step-by-step deployment instructions for Vercel with screenshots.

Thank you for the opportunity — this README should make onboarding and production deployment much smoother.
