# Backend

The backend API owns identity, authorization, workflow state and audit events. It never trusts the client and never runs user code inside the API process.

## Stack

Node.js, TypeScript, Fastify, Postgres, Redis with BullMQ, Zod for validation. The OpenAPI spec is generated from the Zod schemas.

## Auth

GitHub OAuth only for the MVP. The backend issues sessions and stores them in an httpOnly cookie.

## Modules

- Auth and sessions: login, logout, current user, expiry
- Issues: seeded issues, assignment to a learner, status
- Workspaces: per-user repo from a seeded template, resumable. Bare git repos driven through isomorphic-git, all behind one module so the git CLI can replace it. A snapshot is a commit sha
- Files and commits: save files, commit, diffs. Paths are normalised and confined to the workspace
- Runs: async. POST returns a job id, workers execute through a Runner interface (Judge0 first) and store the result
- Hidden tests: stored separately, versioned, injected only at run time
- Pull requests: draft, submitted, changes requested, approved, merged. Submission triggers a hidden-test run
- Reflection: short self-assessment per PR, stored as data only, never part of the readiness score
- Progress and readiness: score computed from recorded events, set only by the backend
- Audit log: append-only events for commits, runs, PRs and state changes. Autosaves are not logged
- Reaper: scheduled sweep of leftover containers, temp checkouts, stuck jobs and old run rows

## Run contract

Statuses: queued, running, passed, failed, timeout, error, cancelled. One container per run, destroyed afterwards. Idempotency key per request, results cached by snapshot sha plus command. PR-submission runs are prioritised over practice runs.

## Endpoints

- Auth: GET /auth/github, GET /auth/callback, POST /auth/logout, GET /me
- Issues and workspaces: GET /issues, POST /issues/:id/start, GET /workspaces/:id
- Files and commits: GET|PUT /workspaces/:id/files, POST /workspaces/:id/commits
- Runs: POST /workspaces/:id/runs, GET /runs/:id
- Pull requests: POST /workspaces/:id/pull-requests, GET /pull-requests/:id, POST /pull-requests/:id/reflection
- History: GET /workspaces/:id/history

## Security

- Every workspace, run and PR id is checked against the session user
- Rate limits: x runs per hour per user, one active run per user, queue length cap
- Jobs carry a scoped token, never platform credentials
- Hidden test source never appears in any API response
- Logs are redacted before storage
- Readiness gate to a real repo is enforced server-side only
- Guards against path traversal, zip-slip and symlinks
- Git runs with hooks disabled and repo-local config ignored
- Red team attacks the sandbox and backend before the showcase

## Open questions

- Per-run containers vs pooling: stay with one container per run for now, revisit if traffic needs it
- Run rate limits and queue scheduling need a deeper look
- Pick libraries so architecture changes for traffic stay easy later
