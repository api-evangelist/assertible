---
name: assertible-trigger-test-run
description: >-
  Run an Assertible web service's tests on demand from a script, CI job or deploy
  hook, against a named or transient environment.
api: assertible:assertible-api
operations:
  - runWebServiceTests
generated: '2026-09-04'
method: generated
source: https://assertible.com/docs/guide/automation#trigger-urls
---

# Trigger an Assertible test run

Use this to run tests outside the Assertible dashboard when you are **not** in a
continuous-delivery pipeline. If you are — if this run is happening because code
was just released — Assertible's own guidance is to use
`assertible-deployment-test-gate` instead, so the run is attached to a
deployment record.

## Before you start

- An Assertible API token, copied from the dashboard.
- The **web service id** (UUID), from the service's Settings tab (or a
  **test** id from a test's Settings tab, for a single-test trigger).

## Steps

1. Call `runWebServiceTests` —
   `POST https://assertible.com/apis/{serviceId}/run`, HTTP Basic with the token
   as username and an empty password.

2. Parameters go in a JSON body **or** in the query string, whichever suits the
   caller. Both are supported for every parameter.

3. Scope the run:
   - `environment` — a configured environment name (defaults to `production`).
   - `url` — run against a specific URL, a *transient environment*; pair it with
     a distinct `environment` name.
   - `endpoint` — only the tests for one endpoint.
   - `tests` — an array of test UUIDs. Only valid on a web-service trigger URL;
     when present, only those tests run.

4. Set `wait: true` to hold the response until every test in the run has
   finished and get the final status back.

5. In a context that cannot set an auth header or a body — a Heroku deploy hook,
   for instance — the token may be passed as the `api_token` query parameter
   instead. Prefer HTTP Basic anywhere you control the request: a token in a
   query string ends up in logs.

## Retry and safety rules

- **This call is NOT idempotent.** Unlike `createDeployment`, there is no natural
  key and no idempotency header — every call starts another test run. If a call
  times out, do not blind-retry; a duplicate run means duplicate HTTP traffic
  against the service under test.
- **There is no cancel.** Assertible documents no way to stop a run once it has
  started, and no dry-run mode. The nearest thing to a rehearsal is scoping the
  run with `tests` or `endpoint`.
- Errors use `{"code": ..., "message": ...}` — `401 AuthenticationError` for a
  bad token, `404 NotFoundError` for an unknown service id.
