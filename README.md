# qa.codes — Postman + Newman API Collection

[![Newman API Tests](https://github.com/qacodes-dev/qacodes-postman-newman-api/actions/workflows/newman.yml/badge.svg)](https://github.com/qacodes-dev/qacodes-postman-newman-api/actions/workflows/newman.yml)

A Postman API collection turned into an automated, CI-gated check with **Newman**. It takes a
collection a tester would normally run by hand and runs it **headless on every push** — covering
the full CRUD lifecycle (create → read → update → delete) with `pm.test` assertions, environment
files for portability, data-driven CSV/JSON iterations, and a shareable HTML report.

This is the reference implementation for the qa.codes project sample:
**<https://qa.codes/practice/project-samples/postman-newman-api>**

## Overview

- **Source of truth:** the exported Postman collection at
  `collections/qacodes-api.postman_collection.json`. Requests are grouped into folders by
  behaviour, each carrying `pm.test` assertions for status code, response body/schema, headers,
  and response time.
- **System under test:** [Postman Echo](https://postman-echo.com) — the stable, no-auth,
  rate-limit-free public demo service purpose-built for HTTP testing. Its `/post`, `/get`,
  `/put`, `/patch`, and `/delete` endpoints echo back the request (body, query args, headers),
  which the assertions validate. This keeps `git clone` → README → run working for anyone and
  keeps CI reliable on every push.
- **Runner:** [Newman](https://github.com/postmanlabs/newman) executes the collection from the
  command line and in CI.
- **Reporting:** `newman-reporter-htmlextra` (rich HTML) + JUnit XML, alongside the live CLI
  summary.
- **CI:** GitHub Actions runs the collection on every push/PR and uploads the reports as an
  artifact.

## Prerequisites

- Node.js 18 or later (CI pins Node 20)
- npm 9 or later
- Git
- [Postman](https://www.postman.com/downloads/) (desktop app) — only if you want to open, edit,
  and re-export the collection interactively
- Basic familiarity with HTTP verbs, status codes, and JSON

## Install

```bash
# 1. Clone
git clone https://github.com/qacodes-dev/qacodes-postman-newman-api.git
cd qacodes-postman-newman-api

# 2. Install newman + newman-reporter-htmlextra locally
npm install

# 3. Copy the environment template and fill in values
cp environments/staging.postman_environment.example.json environments/staging.postman_environment.json

# 4. Verify Newman is available
npx newman --version
```

Set `BASE_URL` and `API_KEY` in `environments/staging.postman_environment.json` (or export them
as shell variables / pass them with `--env-var` for CI). For the public demo the defaults work
out of the box — `BASE_URL=https://postman-echo.com` and any placeholder `API_KEY`.

> If you want to edit the collection interactively, import
> `collections/qacodes-api.postman_collection.json` and
> `environments/staging.postman_environment.json` into Postman.

## Run commands

```bash
# Run the collection against an environment — every request + its pm.test assertions
# Exits non-zero if any assertion fails.
npx newman run collections/qacodes-api.postman_collection.json \
  -e environments/staging.postman_environment.json

# Add the rich HTML (htmlextra) report on top of CLI output
npx newman run collections/qacodes-api.postman_collection.json \
  -e environments/staging.postman_environment.json \
  -r cli,htmlextra --reporter-htmlextra-export reports/report.html

# Data-driven run from a CSV file — one iteration per row, substituting {{column}} variables
npx newman run collections/qacodes-api.postman_collection.json \
  -e environments/staging.postman_environment.json \
  -d data/users.csv

# CI run with HTML + JUnit reports (this is what `npm test` wraps)
npx newman run collections/qacodes-api.postman_collection.json \
  -e environments/staging.postman_environment.json \
  -r cli,htmlextra,junit \
  --reporter-htmlextra-export reports/report.html \
  --reporter-junit-export reports/junit.xml

# Override an environment variable at run time (point at an ad-hoc host without editing files)
npx newman run collections/qacodes-api.postman_collection.json \
  -e environments/staging.postman_environment.json \
  --env-var "BASE_URL=https://api.example.dev"

# npm script wrapper — one entry point (wraps the CI run command above)
npm test
```

Convenience scripts in `package.json`:

| Script | What it does |
| --- | --- |
| `npm test` | CLI + HTML + JUnit reports into `reports/` (the CI command) |
| `npm run test:html` | CLI + HTML report only |
| `npm run test:data` | Data-driven run from `data/users.csv` |
| `npm run test:data:json` | Data-driven run from `data/users.json` |

> **Note:** [Postman Echo](https://postman-echo.com) is a stateless echo service, so each request
> validates against the data it sends back rather than a persisted record. The CRUD lifecycle is
> modelled by generating an id client-side, echoing it through create/read/update/delete, and
> asserting it round-trips — the same assertion patterns you'd use against a real stateful API.

## Environment configuration

Every documented variable also lives in `.env.example` and the Postman environment files.

| Variable | Required | Description | Example |
| --- | --- | --- | --- |
| `BASE_URL` | yes | Root URL of the API under test; referenced in every request as `{{BASE_URL}}` | `https://postman-echo.com` |
| `API_KEY` | yes | API key injected as an `x-api-key` header by a collection-level script | `sk_test_51H...` |
| `AUTH_TOKEN` | no | Bearer token added to `Authorization` at run time when set; blank in the template | `(set after a login request)` |
| `REQUEST_TIMEOUT_MS` | no | Optional per-request timeout passed to Newman as `--timeout-request` | `10000` |

Real values belong **only** in the gitignored `environments/staging.postman_environment.json` or
in CI secrets — never in the committed collection, data files, or `*.example.json`.

## Folder structure

```
collections/
  qacodes-api.postman_collection.json        # Main collection: CRUD folders with pm.test assertions
environments/
  staging.postman_environment.json           # Real values (gitignored)
  staging.postman_environment.example.json   # Committed template with empty secrets — copy and fill in
data/
  users.csv                                  # CSV rows; headers (name,email,role) → {{variables}}
  users.json                                 # JSON alternative for nested iteration data
reports/                                     # Generated HTML + JUnit reports (gitignored)
.github/workflows/newman.yml                 # CI: install, run Newman, upload reports
package.json                                 # newman + htmlextra + the test scripts
```

### What the collection does

Requests are grouped into folders that model one CRUD lifecycle per iteration:

1. **Create** — `POST /post` with a client-generated id; asserts the id and the injected
   `x-api-key` header round-trip, then captures the id into a collection variable.
2. **Read** — `GET /get?userId={id}` (by the captured id) and `GET /get?role=…&page=1` (filtered
   list) — asserts the query params come back correctly.
3. **Update** — `PUT /put` (full replace, adds a `status` field) and `PATCH /patch` (single field).
4. **Delete (teardown)** — `DELETE /delete?userId={id}` then `GET /status/404` asserting a `404`.
5. **Data-Driven** — `POST /post` driven by `{{name}}`/`{{email}}`/`{{role}}` from the data file,
   asserting each row's values round-trip.

Shared setup lives in **collection-level scripts**: a pre-request script injects the auth header
from `API_KEY` and (when `AUTH_TOKEN` is set) a bearer token; the Create request generates a
unique id + suffix so repeated runs never collide; the id is captured with
`pm.collectionVariables.set()` and reused across read/update/delete within the iteration.

## Reporting

- **`newman-reporter-htmlextra`** → `reports/report.html` — a self-contained HTML report with
  per-request timings, request/response bodies, and assertion pass/fail detail.
- **JUnit XML** → `reports/junit.xml` — the format CI parses for native test summaries.
- **CLI** — a live pass/fail summary in the terminal, kept enabled alongside the file reporters.
- Newman exits **non-zero** on any assertion failure, so CI turns red automatically.
- Reports are written to `reports/` (gitignored locally) and uploaded as CI artifacts.

## CI

`.github/workflows/newman.yml`:

- Triggers on `push` and `pull_request`.
- Uses `actions/setup-node@v4` pinned to **Node 20** with npm caching.
- Installs with `npm ci` from the lockfile.
- Runs the collection via `npm test`, emitting HTML + JUnit into `reports/`.
- Injects `BASE_URL` and `API_KEY` from **GitHub Actions secrets** via `--env-var` (with public-demo
  fallbacks) — no secret is committed.
- Uploads `reports/` with `actions/upload-artifact@v4` using `if: always()` so reports survive
  failing runs.

## License

[MIT](./LICENSE) © qa.codes

---

Part of the [qa.codes](https://qa.codes) Project Samples —
[Postman + Newman API Collection](https://qa.codes/practice/project-samples/postman-newman-api).
