---
name: assertible-sync-spec-tests
description: >-
  Re-import an OpenAPI, Swagger or Postman specification into Assertible from CI
  so tests track the spec as it changes.
api: assertible:assertible-api
operations:
  - syncImportTests
generated: '2026-09-04'
method: generated
source: https://assertible.com/docs/guide/sync
---

# Keep Assertible tests in sync with a specification

Use this when the API description is the source of truth and you want the test
suite to follow it — typically as a job that runs after the spec is published.

## Before you start

- The web service must **already** have an import created from a **URL** (a
  one-off file upload cannot be synced). Imports are listed on the web service's
  Settings → Imports tab, which is also where the **import id** comes from.
- Assertible documents a separate `ASSERTIBLE_SYNC_TOKEN` for this call.

## Steps

1. Call `syncImportTests` —
   `POST https://assertible.com/imports/{importId}/sync`, HTTP Basic with the
   sync token as username and an empty password.

2. Send no body to sync everything. Send `{"tests": ["<uuid>", ...]}` to restrict
   the sync to specific tests; a test id is on the test's Settings tab beside its
   trigger URL.

3. A `200` responds with the number of tests imported or synced.

## What sync will do to the suite

- For OpenAPI and Swagger, each unique **METHOD/ENDPOINT** pair is one Assertible
  test — unless the operation declares an `operationId`, which overrides the key.
  Give every operation a stable `operationId` if you ever intend to move or
  rename a path, or the rename will read as a delete plus a create.
- For Postman collections the key is the collection **Item id**.
- Assertible populates test configuration from the spec, including JSON Schema
  assertions from response definitions.

## Retry and safety rules

- **Retrying is safe.** The matching key is deterministic, so a repeated sync
  updates the same tests rather than duplicating them.
- **There is no undo.** Assertible documents no way to revert a sync to the
  previous test configuration, and no dry-run or preview. A spec change that
  renames paths will rewrite the suite, and you cannot roll it back through the
  API. Review the spec diff before running this in an automated job.
- Errors use `{"code": ..., "message": ...}`; `401 AuthenticationError` on a bad
  token, `404 NotFoundError` on an unknown import id.
