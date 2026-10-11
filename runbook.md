# Runbook: GitHub Issue Triage workflow

## Purpose

How to operate, verify and recover the local GitHub issue triage workflow.
**Action:** use before any test session and after any failure. **Expected result:** each new issue produces exactly one ledger row and one route-log row. **Evidence:** [test_evidence.md](test_evidence.md).

## Normal operation

1. Open n8n at `http://localhost:5678` and confirm **Sprint 2 — GitHub Issue Triage** shows as Published.
2. In a separate terminal run the `gh webhook forward` command from the README and leave it open.
3. Open a new issue in the repository and wait for a new execution under **Executions**.
4. Open `github_issue_ledger` and `github_issue_route_log` and reload them.

**Expected result:** an execution that follows Opened action? (true), the matching Set route node, Insert pending ledger row and Record internal route; one ledger row with `route_log_status = recorded`; one route-log row with the same `issue_key`. **Evidence:** figures 1 to 14 in test_evidence.md.

## Duplicate delivery

**Action:** GitHub, Settings, Webhooks, the GitHub Command Line webhook, Recent Deliveries, pick the `issues.opened` delivery, click Redeliver (forwarder running).
**Expected result:** the execution takes the If ledger row exists branch and runs Update existing last seen; the ledger still has one row for that `issue_key` with a newer `last_seen_at`; the route-log table is unchanged.
**Evidence:** not yet captured. Capture the route-log table before and after the redelivery.

If GitHub returns HTTP 401 on the redelivery, the forwarder session changed (a restart creates a new webhook and discards the old delivery history). Redeliver a delivery created after the restart.

## Route-log failure

**Action:** create an issue whose title contains `intentional route-log failure test`.
**Expected result:** Intentional failure test? sends the item to Simulate route-log failure; its error output runs Update route-log failed; the ledger row shows `route_log_status = failed` with the safe `error_note`; no route-log row exists for that `issue_key`.
**Evidence:** figures 15 to 18 show issue #12 and current behavior, which does not yet match this expectation (see known issues in test_evidence.md).

For a genuine failure, Record internal route's error output feeds the same Update route-log failed node.

## Recovery action

1. In the ledger, filter for `route_log_status = failed` (or `pending` for a long time).
2. Open the matching failed execution, read the error on Record internal route or Simulate route-log failure, and fix the cause (data table selection or column mapping).
3. Add the missing row to `github_issue_route_log` with the ledger row's `issue_key`, `category`, `route_name` and a current `recorded_at`.
4. Edit the ledger row: set `route_log_status = recorded` and clear `error_note`.

Do not recover by redelivering the webhook: the ledger row already exists, so a redelivery only updates `last_seen_at` and never retries the route-log write.
**Expected result:** one ledger row marked recorded and one route-log row. **Evidence:** none yet; no genuine failed row has been produced.

## Safe shutdown

1. Stop the forwarder with Ctrl+C.
2. In n8n, unpublish (deactivate) the workflow.
3. In GitHub, Settings, Webhooks: delete or disable the GitHub Command Line webhook.
4. Export the workflow, keep the private export outside the repository, and commit only the redacted copy.

**Expected result:** new issues create no executions. **Evidence:** shutdown date and time: ____________.

## What this prototype does not test

- **Production hosting:** forwarding works only while the terminal is open and the laptop is awake; events sent while it is closed are not queued.
- **Webhook authenticity:** no webhook secret or signature check is configured.
- **Concurrent duplicates:** the two ledger checks run in parallel, so two simultaneous deliveries of one new issue could both pass If ledger row does not exist.
- **Issue edits, closes and reopens:** deliberately ignored; an edited title is not reclassified.
- **Volume and Data Table limits:** only a handful of issues were processed.
- **Other repositories and event types:** only one repository and the `issues` event.
