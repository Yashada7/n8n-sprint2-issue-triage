# Week 6 build journal

(If the course journal template has different headings, move this content into it.)

## What I built

A local n8n workflow that receives GitHub `issues` webhooks through GitHub CLI forwarding, gates to `opened` events, normalizes the payload, classifies the title with regex rules, checks a ledger table for the `issue_key`, and writes a ledger row plus a route-log row for new issues. Existing keys only update `last_seen_at`.

## What went wrong and how I fixed it

| Problem | How I diagnosed it | Fix |
| --- | --- | --- |
| Editor showed "Waiting for test URL" after I clicked Execute step | No test event arrived because GitHub does not call the test URL | Real events appear under Executions only for the published workflow, and edits need a republish (the execution header showed version 1, then 3, then 4) |
| Every issue routed to "other" | Classify title's Parameters showed `\\b` in the patterns | Retyped each pattern with a single backslash; the double backslash came from copying the Word handout |
| Ledger rows had Null fields | Set route node output was only `{ "bug": "bug-triage" }` | Rebuilt each Set route node with `category` and `route_name` and kept the other input fields |
| Redelivery returned 401 | GitHub's Response tab showed `Unauthorized` from its own forwarding service, with no n8n execution | Restarted the forwarder, which created a new webhook; old deliveries were gone, so I created fresh issues |
| Failure test and stored category still wrong | Issue #12 ended recorded with a route-log row; ledger `category` is Null | Diagnosed likely causes (see test_evidence.md); fixes still to apply and retest |

## Extensibility observation

Adding a fifth category (for example `security` with route `security-review`) takes one new Switch rule, one new Set route node and two wires into the existing ledger checks. The duplicate guard, ledger and route log need no changes, so classification is well separated from storage. The weak spot is coupling through item shape: Data Table nodes return the stored row, not the original item, so downstream nodes such as the failure IF silently lose fields like `title_lower`. A more extensible version would reference earlier nodes explicitly with `$('Normalize issue')` or pass a single context object through the whole flow.

## What I would do next

- Apply the two fixes in test_evidence.md, republish, and recapture the failure, redelivery and gate evidence.
- Move the duplicate check ahead of the branches so two parallel checks cannot race.
- Add a webhook secret check and move off the laptop forwarder before any real use.
