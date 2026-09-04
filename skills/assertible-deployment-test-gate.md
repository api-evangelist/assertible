---
name: assertible-deployment-test-gate
description: >-
  Record a deployment in Assertible and gate the release on the resulting API
  test run, optionally publishing the result back to a GitHub commit.
api: assertible:assertible-api
operations:
  - createDeployment
generated: '2026-09-04'
method: generated
source: https://assertible.com/docs/guide/deployments
---

# Gate a release on Assertible tests

Use this after a deploy has landed and before you shift traffic or declare the
release good.

## Before you start

- You need an Assertible API token. Assertible does not expose a way to create
  or list one over the API — copy it from the dashboard.
- You need the **web service id** (a UUID). It is on the service's Deployments
  tab under "Bash / Command-line". There is no API to look it up.
- If you want GitHub commit statuses, the web service must already be connected
  to the repository in the Assertible dashboard.

## Steps

1. Call `createDeployment` — `POST https://assertible.com/deployments`.
   Authenticate with HTTP Basic: the token is the **username**, the password is
   **empty** (`curl -u $ASSERTIBLE_API_TOKEN:`).

2. Send at minimum `service` (UUID) and `version` (string). Add `environment`
   when the deploy did not go to `production` — the name must match an
   environment configured on that web service, unless you also send
   `environmentUrl` for a transient environment.

3. Set `wait: true` if you want the call to block until the run finishes and
   return the final status. Without it you get `testRun.status:
   "TestRunPending"` and no verdict.

4. For a GitHub status check, send `github: true` **and** `ref` set to the
   **full** SHA1 (`git rev-parse HEAD` — a short hash will not work). If the
   service has several repositories connected, also send `repository`.

5. Narrow the run with `tests` (array of test UUIDs) or `testGroups` (array of
   group names) when you only want part of the suite. They can be combined.

## Interpreting the response

- `200` returns `{ "id": ..., "runId": ..., "testRun": { "status": ... } }`.
  With `github: true`, `runId` and `testRun` come back `null` — the result is
  delivered to GitHub, not inline.
- `400` returns `{"code":"InvalidRequestError","message":"Cannot parse request
  body..."}` — a required key (usually `service`) is missing.
- `401` returns `{"code":"AuthenticationError","message":"Not logged in"}` — the
  token was not sent as the Basic username, or is wrong.

## Retry and safety rules

- **Retrying is safe, with a caveat.** A deployment is uniquely identified by
  `service` + `environment` + `version`, and a repeat POST *updates* that
  deployment rather than creating a second one. There is no `Idempotency-Key`
  header.
- The caveat: because the key is the natural key, a retry that changes an
  optional field (`ref`, `url`) silently overwrites the earlier record, and two
  genuinely different deployments sharing a service/environment/version collapse
  into one. Vary `version` per build.
- **There is no undo.** Assertible documents no operation to cancel, delete or
  roll back a deployment or an in-flight test run. Do not call this expecting to
  be able to take it back.
- No rate limits are published and no `RateLimit-*` or `Retry-After` headers are
  returned, so back off on your own schedule; treat any 5xx as retryable and any
  4xx as terminal.
