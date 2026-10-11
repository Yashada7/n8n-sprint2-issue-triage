# n8n Sprint 2: GitHub Issue Triage Automation

Repository URL: https://github.com/Yashada7/n8n-sprint2-issue-triage (public)

## Purpose

An event-driven n8n workflow that receives new GitHub issues from this public repository, classifies each issue from its title with deterministic rules, saves a durable triage record (the ledger), and writes a category-specific route-log entry. A repeated delivery of the same issue updates the ledger row's `last_seen_at` only and never writes a second route-log entry. A failed route-log write is meant to leave a visible `failed` state on the ledger row.

## Services

- **GitHub**: this public repository is the event source (issues webhook).
- **GitHub CLI with the gh-webhook extension**: forwards `issues` events to the local n8n webhook while the command runs.
- **n8n (self-hosted)** at `http://localhost:5678`: hosts the workflow and two Data Tables, `github_issue_ledger` and `github_issue_route_log`.

No tunnel, public domain, external notification service, or LLM is used.

## Node sequence

1. **GitHub issue webhook** (Webhook, POST, path `sprint2-issue-triage`) receives the forwarded event.
2. **Opened action?** (IF) passes only `body.action = opened`; the false output is unconnected, so edits, closes and reopens stop here.
3. **Normalize issue** (Edit Fields) builds one flat item with a stable `issue_key`, the lowercased title and a processing timestamp.
4. **Classify title** (Switch, rules mode) applies three regex rules in order plus a fallback output.
5. **Set bug route / Set feature route / Set help route / Set other route** (Edit Fields) set the fixed `category` and `route_name`.
6. **If ledger row exists** then **Update existing last seen** (Data Table): existing key, update `last_seen_at` only.
7. **If ledger row does not exist** then **Insert pending ledger row** (Data Table): new key, insert the row with `route_log_status = pending` before any route-log write.
8. **Intentional failure test?** (IF) sends titles containing `intentional route-log failure test` to the failure simulation; everything else continues.
9. **Record internal route** (Data Table insert into `github_issue_route_log`, On Error = continue using error output).
10. **Update route-log recorded** (Data Table update): `route_log_status = recorded`, blank `error_note`.
11. **Simulate route-log failure** (Code, On Error = continue using error output) throws a controlled error.
12. **Update route-log failed** (Data Table update): `route_log_status = failed` and a safe fixed `error_note`; fed by the error outputs of Record internal route and Simulate route-log failure.

## Field mapping

### Normalize issue

| Field | Expression |
| --- | --- |
| `repository_full_name` | `{{ $json.body.repository.full_name }}` |
| `issue_number` | `{{ $json.body.issue.number }}` |
| `issue_key` | `{{ $json.body.repository.full_name + "#" + $json.body.issue.number }}` |
| `issue_url` | `{{ $json.body.issue.html_url }}` |
| `title` | `{{ $json.body.issue.title }}` |
| `title_lower` | `{{ $json.body.issue.title.toLowerCase() }}` |
| `reporter` | `{{ $json.body.issue.user.login }}` |
| `opened_at` | `{{ $json.body.issue.created_at }}` |
| `last_seen_at` | `{{ $now.toISO() }}` |

### Ledger and route log

| Column | Ledger value on insert | Route-log value |
| --- | --- | --- |
| `issue_key` | `{{ $json.issue_key }}` | `{{ $json.issue_key }}` |
| `issue_url`, `title`, `reporter`, `opened_at` | same-named field from Normalize issue | n/a |
| `category` | `{{ $json.category }}` | `{{ $json.category }}` |
| `route_name` | `{{ $json.route_name }}` | `{{ $json.route_name }}` |
| `last_seen_at` / `recorded_at` | `{{ $json.last_seen_at }}` | `recorded_at` = `{{ $json.last_seen_at }}` |
| `route_log_status` | fixed `pending`, later `recorded` or `failed` | n/a |
| `error_note` | blank, or a safe fixed message on failure | n/a |

## Exact classification rules

Classification uses only the lowercased title (`{{ $json.title_lower }}`). The first matching rule wins.

| Order | Regex (matches regex) | Category | Route name | Example |
| --- | --- | --- | --- | --- |
| 1 | `\b(bug\|error)\b` | bug | bug-triage | Bug: blank screen after logout |
| 2 | `\b(feature\|request)\b` | feature | feature-requests | Feature request: export task list to CSV |
| 3 | `\b(help\|question)\b` | help | support | Help: change the public webhook address |
| 4 | fallback output | other | general-triage | Documentation: clarify local setup |

In n8n the patterns must be typed with a single backslash and a normal pipe (`\b(bug|error)\b`). A double backslash matches a literal backslash and every issue falls through to `other`.

## Setup notes

1. Create the public repository `n8n-sprint2-issue-triage` with a README and Issues enabled.
2. Run n8n locally at `http://localhost:5678`; create the workflow and both Data Tables with the columns above.
3. Install GitHub CLI, run `gh auth login`, then `gh extension install cli/gh-webhook`.
4. Click **Publish** in n8n. Only the published workflow listens on `/webhook/sprint2-issue-triage`; the editor's "Waiting for test URL" mode uses a different `/webhook-test/` URL that GitHub never calls.
5. In a separate terminal run `gh webhook forward --repo=Yashada7/n8n-sprint2-issue-triage --events=issues --url=http://localhost:5678/webhook/sprint2-issue-triage` and leave it running.
6. Edits made after publishing are not live until the workflow is published again; the execution header shows the version that ran.

## Evidence

Screenshots and results are in [test_evidence.md](test_evidence.md). Other files: [runbook.md](runbook.md), [own_vs_rent.md](own_vs_rent.md), [journals/week-06.md](journals/week-06.md), and the redacted workflow export `sprint2_github_issue_triage.json`.

Test issues: [#6](https://github.com/Yashada7/n8n-sprint2-issue-triage/issues/6), [#7](https://github.com/Yashada7/n8n-sprint2-issue-triage/issues/7), [#8](https://github.com/Yashada7/n8n-sprint2-issue-triage/issues/8), [#10](https://github.com/Yashada7/n8n-sprint2-issue-triage/issues/10), [#11](https://github.com/Yashada7/n8n-sprint2-issue-triage/issues/11), [#12](https://github.com/Yashada7/n8n-sprint2-issue-triage/issues/12).
