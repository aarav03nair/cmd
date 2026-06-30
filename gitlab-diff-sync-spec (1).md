# Build Spec: Scheduled GitLab "Diff → Jira Ticket → Feature Branch → MR" Automation

## 1. Objective

Build a script + GitLab CI pipeline that runs **on a schedule**. On each run it inspects a **target repository**, and if the repo's **default branch** is ahead of (differs from) **`master`**, it automatically:

1. Creates a **Jira ticket** on the `COREENGDEV` board.
2. Creates a **feature branch** off the default branch, named `feature/feature.<TICKET_KEY>` (e.g. `feature/feature.COREENGDEV-1001`).
3. Opens a **Merge Request** from that feature branch **into `master`**, linking the Jira ticket.

The only thing an operator should need to provide is the **repository link** (plus credentials, set once as CI variables). Everything else is automated.

> **Note on the branch name:** the requested format literally doubles the word "feature" → `feature/feature.COREENGDEV-1001`. I've kept it exactly as specified. If the doubled `feature.` is a typo and you want `feature/COREENGDEV-1001`, change `BRANCH_PREFIX` in one place (see config).

---

## 2. High-level pipeline shape

The pipeline has **two stages**, exactly as required:

| Stage | Job | Responsibility |
|-------|-----|----------------|
| `detect` | `detect-diff` | Compare `master` vs default branch in the target repo. Output a flag `DIFF_EXISTS=true/false`. |
| `act` | `sync-branches` | Runs **only if a diff exists**. Creates the Jira ticket, the feature branch, and the MR. |

- The pipeline must only run on **scheduled** triggers: gate all jobs with `rules: if: $CI_PIPELINE_SOURCE == "schedule"`.
- Stage 1 passes the diff result to stage 2 via a **dotenv artifact** (`diff.env`).
- Stage 2's `rules` reference that variable so the job is **not created** when there's no diff. The script must **also** re-check the flag internally as a safety net (belt-and-suspenders), so it never acts on a false positive.

The schedule itself is configured in the GitLab UI (**CI/CD → Schedules**), not in YAML. The spec just needs the jobs to be schedule-gated.

---

## 3. Inputs & configuration (CI/CD variables)

All secrets must come from **masked, protected CI/CD variables** — never hardcoded. The script reads everything from environment variables.

### Required
| Variable | Example | Purpose |
|----------|---------|---------|
| `TARGET_REPO` | `https://gitlab.example.com/core-eng/my-service` | The repo to inspect. Accept a full URL **or** a `namespace/project` path **or** a numeric project ID. Pass this as a **scheduled pipeline variable** so each schedule can point at a different repo. |
| `GITLAB_API_URL` | `https://gitlab.example.com/api/v4` | GitLab API root. |
| `GITLAB_TOKEN` | `glpat-…` | PAT / project access token / group token with **`api`** scope. Must have permission to create branches and MRs in the target repo. |
| `JIRA_BASE_URL` | `https://jira.example.com` | Jira root URL. |
| `JIRA_PROJECT_KEY` | `COREENGDEV` | Project key tickets are created under. |
| `JIRA_AUTH` | see §7 | Jira credentials (Cloud = email + API token basic auth; Data Center/Server = PAT bearer). |

### Optional (with sensible defaults)
| Variable | Default | Purpose |
|----------|---------|---------|
| `MASTER_BRANCH` | `master` | The target/protected branch to merge into. |
| `DEFAULT_BRANCH` | *(auto-detected via API)* | Override if you don't want to use the repo's configured default branch. |
| `BRANCH_PREFIX` | `feature/feature.` | Prepended to the ticket key to form the branch name. |
| `JIRA_ISSUE_TYPE` | `Task` | Issue type for the created ticket. |
| `MR_LABELS` | `auto-sync` | Label applied to the MR (used for idempotency — see §6). |
| `JIRA_LABELS` | `auto-sync` | Label applied to the Jira ticket. |
| `DRY_RUN` | `false` | If `true`, log all actions but create nothing. **Build this in** — it's essential for safe first runs. |

---

## 4. Detailed logic

### Stage 1 — `detect-diff`
1. Resolve `TARGET_REPO` into a GitLab **project identifier**:
   - If numeric → use as-is.
   - If a URL → strip scheme/host and trailing `.git`, take the path (`namespace/project`), then **URL-encode** it (e.g. `core-eng%2Fmy-service`).
2. Fetch project metadata: `GET /projects/:id` → read `default_branch` (unless `DEFAULT_BRANCH` is set).
3. Compare branches:
   - `GET /projects/:id/repository/compare?from=<MASTER_BRANCH>&to=<DEFAULT_BRANCH>`
   - A diff exists if the response's `commits` array is non-empty **or** `diffs` array is non-empty.
   - Handle the case where `master` doesn't yet exist (treat as "diff exists" or skip — make this an explicit, documented choice; recommended: log a clear warning and skip rather than error out).
4. Write the result to a dotenv artifact:
   ```
   echo "DIFF_EXISTS=true" >> diff.env     # or false
   echo "DEFAULT_BRANCH=<resolved>" >> diff.env
   echo "PROJECT_ID=<resolved id>" >> diff.env
   ```
5. Always exit `0` (a "no diff" result is success, not failure).

### Stage 2 — `sync-branches` (runs only when `DIFF_EXISTS == "true"`)
1. **Idempotency check first** (see §6). If an open auto-sync MR already exists for this repo → log and exit `0`. Do **not** create duplicates on every scheduled run.
2. Create the Jira ticket:
   - `POST <JIRA_BASE_URL>/rest/api/2/issue` (DC/Server) **or** `/rest/api/3/issue` (Cloud).
   - Body: project key = `COREENGDEV`, issue type, a generated summary and description (include repo name, branches compared, link back to the GitLab pipeline via `$CI_PIPELINE_URL`).
   - Capture the returned **issue key** (e.g. `COREENGDEV-1001`) from the response.
3. Build the branch name: `<BRANCH_PREFIX><TICKET_KEY>` → `feature/feature.COREENGDEV-1001`.
4. Create the feature branch **from the default branch**:
   - `POST /projects/:id/repository/branches?branch=<branch_name>&ref=<DEFAULT_BRANCH>`
5. Create the MR:
   - `POST /projects/:id/merge_requests`
   - `source_branch` = feature branch, `target_branch` = `MASTER_BRANCH`.
   - Title should include the ticket key (e.g. `COREENGDEV-1001: Sync <default> → master`).
   - Description links the Jira ticket URL.
   - Apply `MR_LABELS`.
   - Optionally set `remove_source_branch=true`.
6. Log a summary: ticket key + URL, branch name, MR URL.
7. If any step fails after the ticket is created, log the created artifacts clearly so a human can clean up (don't leave silent orphans).

---

## 5. Required API calls (reference)

**GitLab** (`GITLAB_TOKEN` via `PRIVATE-TOKEN` header):
- Resolve project: `GET /projects/:id`
- Compare: `GET /projects/:id/repository/compare?from=<base>&to=<head>`
- List MRs (idempotency): `GET /projects/:id/merge_requests?state=opened&target_branch=<master>&labels=<MR_LABELS>`
- Create branch: `POST /projects/:id/repository/branches`
- Create MR: `POST /projects/:id/merge_requests`

**Jira**:
- Create issue: `POST /rest/api/2/issue` (DC/Server) or `/rest/api/3/issue` (Cloud) — confirm which (see §7).
- (Optional) Search existing tickets for idempotency: `GET /rest/api/2/search?jql=...`

---

## 6. Idempotency & safety (important — don't skip)

Because this runs on a schedule, a persistent diff would otherwise spawn a **new ticket + branch + MR on every single run**. Prevent that:

- Before doing anything in stage 2, query GitLab for an **open MR** targeting `master` carrying the `auto-sync` label (or a matching source-branch pattern). If one exists → exit cleanly, create nothing.
- Optionally also check whether the feature branch already exists before creating it.
- Tag both the Jira ticket and the MR with `auto-sync` so they're identifiable.
- Implement `DRY_RUN` and verify a full dry run before going live.

---

## 7. Things to confirm before/while building (flag these explicitly)

1. **Jira flavour** — Cloud vs Data Center/Server. This changes the API version (`/3` vs `/2`) and auth:
   - Cloud → Basic auth with `email:api_token` (base64).
   - DC/Server → `Authorization: Bearer <PAT>`.
   Make the auth method a config switch (`JIRA_AUTH_MODE = basic | bearer`) so either works.
2. **GitLab instance** — self-hosted vs gitlab.com. Should work for both as long as `GITLAB_API_URL` is set.
3. **Token permissions** — the GitLab token must be allowed to push to / create MRs in the **target** repo, which may differ from the repo hosting this pipeline. Confirm token scope and project membership.
4. **`master` as the merge target** — confirm `master` is the correct protected target and that merging default → master is the intended direction.
5. **Jira "board" vs "project"** — tickets are created against the **project key** `COREENGDEV`, not a board directly. Confirm the board maps to that project.

---

## 8. Phased build plan (build & test locally first, GitLab last)

Build this in four phases, in order. Each phase must be **fully testable on a local machine** before moving to the next — nothing touches GitLab CI until Phase 4, and even then it's a manual pipeline run, not a schedule.

To make local testing possible, the script must support reading config from a local **`.env`** file (via `python-dotenv`) in addition to real CI/CD variables, so the same code path runs identically on a laptop and in the pipeline. Provide a `.env.example` with every variable from §3 listed.

### Phase 1 — Diff detection only
**Goal:** prove the GitLab API calls work and diff detection is correct, nothing else.

- Build just the `detect` module: resolve `TARGET_REPO` → project ID, fetch `default_branch`, call `compare`, print whether a diff exists (and a short commit summary if so).
- Runnable directly: `python -m diffsync.detect` reading from local `.env`.
- No Jira calls, no branch/MR creation — read-only against GitLab.
- **Local test:** point `TARGET_REPO` at a real repo where you know master and default branch differ, and a second one where they don't. Confirm correct `true`/`false` output for both.
- ✅ Exit criteria: detection is correct on at least one "diff" repo and one "no diff" repo, run from your machine.

### Phase 2 — Jira ticket + branch + MR creation ("act" logic)
**Goal:** prove each mutating action works in isolation, behind `DRY_RUN`.

- Build the `sync` module: ticket creation, branch creation, MR creation, and the idempotency check (§6) — but call it standalone (not yet wired to Phase 1's output).
- Default `DRY_RUN=true` locally. Run with dry run on first — confirm the logged actions (ticket payload, branch name, MR payload) look correct.
- Then run once with `DRY_RUN=false` against a **throwaway/test repo and test Jira project**, not a real one, to confirm tickets/branches/MRs are actually created correctly end-to-end.
- Run it a second time immediately after — confirm idempotency kicks in and nothing is duplicated.
- **Local test:** `python -m diffsync.sync` reading from local `.env`, same as Phase 1.
- ✅ Exit criteria: one real ticket + branch + MR created on a test repo, a second run creates nothing new, dry run mode produces no side effects.

### Phase 3 — Wire the two together (still local)
**Goal:** the full flow as one program, still run from a laptop, no GitLab CI yet.

- Combine Phase 1 + Phase 2 into the real control flow: detect → if diff → act (with the idempotency/dry-run guards already proven).
- Single entrypoint, e.g. `python -m diffsync.run`, that does exactly what the two-stage pipeline will do, sequentially.
- **Local test:** run the full thing against your "diff" test repo (expect ticket+branch+MR) and your "no diff" repo (expect a clean no-op).
- ✅ Exit criteria: full local dry run and full local live run both behave correctly, matching the acceptance criteria in §11.

### Phase 4 — GitLab CI pipeline wrapper
**Goal:** move the already-proven logic into the two-stage pipeline, run manually first.

- Write the `.gitlab-ci.yml` (see §10), splitting Phase 3's single entrypoint back into the two CI jobs (`detect-diff`, `sync-branches`), wired via the dotenv artifact.
- Add `requirements.txt` and confirm CI/CD variables (§3) are set in the GitLab project settings (masked/protected), matching exactly what worked locally in the `.env` file.
- **Do not** attach a schedule yet. Trigger the pipeline manually from the GitLab UI ("Run pipeline") against the test repo first, with `DRY_RUN=true`, then `false`.
- Only once a manual run behaves correctly end-to-end should a **Schedule** be created (CI/CD → Schedules) pointing `TARGET_REPO` at the real repo(s).
- ✅ Exit criteria: a manually triggered pipeline run on GitLab reproduces the same result as the Phase 3 local run.

> Build order matters here: each phase should be a separate, small Claude Code session/commit so failures are easy to isolate — don't let Claude Code jump ahead to the pipeline YAML before Phases 1–3 are locally verified.

---

## 9. Deliverables

Produce:
1. **`.gitlab-ci.yml`** — two-stage pipeline as described, schedule-gated, dotenv passing between stages.
2. **The script** — Python (use `requests`; minimal dependencies). Single well-structured module or a small package, with functions clearly separated: repo resolution, diff detection, Jira ticket creation, branch creation, MR creation, idempotency check.
3. **`requirements.txt`**.
4. **`README.md`** — setup steps: which CI/CD variables to create, how to create the schedule in the GitLab UI, how to run a dry run, and how to test locally.
5. Inline logging throughout (clear, greppable, with the repo/ticket/MR identifiers).

### Quality requirements
- No hardcoded secrets, URLs, or project IDs.
- Robust error handling: non-2xx API responses must raise with the response body logged.
- Idempotent (see §6).
- `DRY_RUN` honored everywhere that mutates state.
- Exit codes: `0` on success **and** on "no diff / already handled"; non-zero only on genuine errors.

---

## 10. Skeleton `.gitlab-ci.yml` (starting point — refine as needed)

```yaml
stages:
  - detect
  - act

default:
  image: python:3.12-slim
  before_script:
    - pip install --no-cache-dir -r requirements.txt

detect-diff:
  stage: detect
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
  script:
    - python -m diffsync.detect   # writes diff.env
  artifacts:
    reports:
      dotenv: diff.env

sync-branches:
  stage: act
  needs:
    - job: detect-diff
      artifacts: true
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule" && $DIFF_EXISTS == "true"'
  script:
    - python -m diffsync.sync     # creates ticket + branch + MR
```

> If the GitLab version doesn't reliably support dotenv variables inside `rules:if`, fall back to running `sync-branches` on every scheduled run and have the script `exit 0` immediately when `DIFF_EXISTS != "true"`. Keep the internal guard regardless.

---

## 11. Acceptance criteria

- [ ] Setting only `TARGET_REPO` (+ the one-time credential variables) on a schedule is enough to make it work.
- [ ] When default branch == master (no diff), stage 2 does nothing and the pipeline succeeds.
- [ ] When a diff exists, exactly one Jira ticket, one branch `feature/feature.<KEY>`, and one MR (feature → master) are created, linked together.
- [ ] Re-running with the same persistent diff does **not** create duplicates.
- [ ] `DRY_RUN=true` creates nothing but logs every intended action.
- [ ] All failures log the offending API response body and return non-zero.
