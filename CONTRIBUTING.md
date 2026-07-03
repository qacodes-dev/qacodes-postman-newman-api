# Contributing

Thanks for helping improve this Postman + Newman sample! This repo is a small,
runnable reference, so contributions should keep it easy to `git clone` → run.

## Getting set up

```bash
git clone https://github.com/qacodes-dev/qacodes-postman-newman-api.git
cd qacodes-postman-newman-api
npm install
cp environments/staging.postman_environment.example.json environments/staging.postman_environment.json
```

The defaults target the public [Postman Echo](https://postman-echo.com) API, so no
secrets are needed to run locally.

## Running the suite

```bash
npm test                 # CLI + HTML + JUnit reports into reports/
npm run test:html        # CLI + HTML report only
npm run test:data        # data-driven run from data/users.csv
npm run test:data:json   # data-driven run from data/users.json
```

Newman exits non-zero if any `pm.test` assertion fails, so a clean `npm test` is the
bar for a green PR.

## Adding or changing a request

The collection at `collections/qacodes-api.postman_collection.json` is the source of
truth. You can edit it two ways:

1. **In Postman (recommended):** import the collection and
   `environments/staging.postman_environment.json`, add or edit a request inside the
   relevant CRUD folder, write assertions in the **Tests** tab, then **export** the
   collection back over the JSON file (Collection v2.1 format).
2. **By hand:** edit the JSON directly — each request's scripts live under its
   `event` array (`prerequest` / `test`).

Please keep these conventions:

- **One `pm.test` per behaviour** (status, a specific field, a header, timing) so a
  failure names exactly what broke.
- Parse the body with `pm.response.json()` before asserting on it.
- Reference the base URL and secrets as `{{BASE_URL}}` / `{{API_KEY}}` — never
  hardcode a host or key in a request.
- Chain requests through collection variables (capture on create, reuse on
  read/update/delete); clean up anything you create.
- Put shared setup (auth header, correlation id) in a collection- or folder-level
  script, not repeated per request.

After editing, run `npm test` (and the data-driven scripts if you touched the
data-driven folder or `data/*`) and confirm everything is green.

## Commit & PR conventions

- Write clear, imperative commit subjects (e.g. "Add negative-case assertions to
  Update folder").
- Keep secrets out of git — real values belong only in the gitignored
  `environments/staging.postman_environment.json` or in CI secrets.
- Open a PR against `main`; fill in the pull request template. CI runs on every push
  and PR and must be green before merge.
