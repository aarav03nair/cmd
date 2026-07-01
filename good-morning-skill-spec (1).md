# Spec: `good-morning` — Claude Code **skill** for daily MR review triage

> **Purpose.** Build specification for Claude Code. Hand it the whole file and ask it to implement the skill. Scope: list every open GitLab MR where the user is a requested reviewer, enrich each with its linked Jira ticket, and sort by how long that ticket has been open so the longest-blocked work surfaces first. Read-only; it surfaces and ranks, it does not act.

---

## 1. Goal

The user runs one command each morning and gets a single prioritized list: **every open MR where they are a requested reviewer**, each annotated with its linked Jira ticket and **how long that ticket has been open**, sorted so the oldest (most-blocked) tickets appear first.

---

## 2. Trigger and UX

- User-invocable skill named `good-morning`, invoked as `/good-morning`. No install step: auto-loads when placed in the skills directory (§3).
- Optional flags accepted in free text and passed to the script:
  - `--projects <ids>` — limit to specific GitLab project IDs (default: all visible projects).
  - `--since <days>` — only consider MRs updated within N days (default: 30) to bound API cost.
  - `--unreviewed` — (optional) hide MRs the user has already approved (default: show all).
- The skill orchestrates; the deterministic API work lives in a bundled script (§6) that emits JSON. Claude reads the JSON and writes the briefing.

---

## 3. Skill structure

A skill is a folder with `SKILL.md` plus supporting files. Place it at **`~/.claude/skills/good-morning/`** (personal scope — loads automatically, no manifest, no marketplace).

```
~/.claude/skills/good-morning/
├── SKILL.md                   # orchestration + frontmatter
├── scripts/
│   ├── fetch_review_queue.py  # GitLab MRs awaiting my review (+ Jira enrichment)
│   ├── jira.py                # Jira helper (issue age, status, priority)
│   └── gitlab.py              # shared GitLab REST helpers (auth, pagination)
├── config.example.toml        # copy to config.toml; documents every tunable
└── README.md                  # setup, tokens, scopes, troubleshooting
```

`SKILL.md` frontmatter (minimum):

```yaml
---
name: good-morning
description: >-
  Morning review triage. Lists open GitLab MRs awaiting my review, enriched with
  their linked Jira ticket, sorted by how long the ticket has been open. Invoke when
  the user says "good morning", asks what needs reviewing, or runs /good-morning.
user-invocable: true
---
```

The skill invokes its script by path **relative to the skill's own directory**, e.g. `python scripts/fetch_review_queue.py`.

**Dependency note:** target Python 3.9+ using only the **standard library** (`urllib.request`, `json`, `os`, `re`, `datetime`, `tomllib`). Avoid `pip install` so it runs in a locked-down corporate environment. (`tomllib` is stdlib on 3.11+; for 3.9–3.10 use a minimal config parser — document the choice in the README.)

---

## 4. Configuration and secrets

Secrets come from environment variables; behavior tunables from `config.toml`. Nothing sensitive is committed.

**Environment variables (required):**

| Variable | Meaning |
|---|---|
| `GITLAB_URL` | Base URL, e.g. `https://gitlab.example.com` |
| `GITLAB_TOKEN` | Personal access token, **read-only** scope: `read_api` |
| `GITLAB_USERNAME` | The user's GitLab username (matches reviewer) |
| `JIRA_URL` | Base URL, e.g. `https://jira.example.com` |
| `JIRA_TOKEN` | Jira PAT (Data Center) **or** API token (Cloud) |
| `JIRA_EMAIL` | Only for Jira Cloud basic auth; omit on Data Center bearer auth |

**`config.toml` tunables (with defaults):**

```toml
[scope]
projects = []          # empty = all visible projects
updated_within_days = 30
unreviewed_only = false # true = hide MRs the user has already approved

[jira]
# Branch is authoritative. Format: feature/feature.COREENGDEV-1234[-anything].
# Regex matches the key and stops at the digits — trailing text is ignored.
key_regex = "COREENGDEV-[0-9]+"
api_version = "2"      # "2" = Server/Data Center, "3" = Cloud
age_basis = "created"  # "created" | "in_progress" (in_progress uses the changelog)
in_progress_statuses = ["In Progress", "In Development", "In Review"]

# Sorting. Primary key is Jira open-duration (older first). Jira priority is an
# optional tiebreaker; set its weight to 0 to sort purely by age.
[priority]
sort = "jira_age"      # primary sort: longest-open ticket first
priority_weight = 0    # >0 = bump higher-priority tickets up within similar ages

[priority.jira_priority_map]
Highest = 5
High = 4
Medium = 3
Low = 2
Lowest = 1
_default = 3
```

> **Decisions to confirm before first run:** Is GitLab/Jira self-hosted or Cloud (sets `api_version` + auth style)? Does "open duration" mean since the ticket was created or since it entered active development (`age_basis`)? Show all reviewer MRs or only not-yet-approved (`unreviewed_only`)? Defaults: self-hosted Data Center, age-since-created, show all.

---

## 5. Core logic

### 5.1 Find open MRs where the user is a requested reviewer

Group-wide GitLab REST:

```
GET {GITLAB_URL}/api/v4/merge_requests
    ?scope=all&state=opened
    &reviewer_username={GITLAB_USERNAME}
    &updated_after={now - updated_within_days}
    &per_page=100
```

This is the whole list. Paginate via the `Link` header / `page` param until exhausted.

- If `unreviewed_only = true`: drop MRs the user has already approved — `GET /projects/:id/merge_requests/:iid/approvals`, drop if the user is in `approved_by`.
- Verify `reviewer_username` is supported on your GitLab version; if not, fall back to `reviewer_id` (resolve via `GET /users?username=...`).

### 5.2 Link each MR to its Jira ticket

The branch name is authoritative. Branches follow `feature/feature.COREENGDEV-1234[-trailing-text]`, so extract the first `key_regex` match (`COREENGDEV-[0-9]+`) from the MR's `source_branch`. The regex stops at the digits, so trailing text after the number is ignored. A key should be found on essentially every MR; if one isn't (malformed branch), mark `jira: none` — the MR still appears, ranked on MR age alone.

### 5.3 Fetch Jira ticket data

```
GET {JIRA_URL}/rest/api/{api_version}/issue/{KEY}
    ?fields=summary,status,priority,created
    &expand=changelog        # only when age_basis = "in_progress"
```

Compute:
- **`age_days`** — `created`: `now - fields.created`. `in_progress`: first transition into any `in_progress_statuses` value from the changelog; fall back to `created` if it never entered one.
- **`status`**, **`priority`**, **`summary`** for display (and priority as optional tiebreaker).

Auth: Data Center uses `Authorization: Bearer {JIRA_TOKEN}`; Cloud uses basic auth (`{JIRA_EMAIL}:{JIRA_TOKEN}` base64). Choose based on whether `JIRA_EMAIL` is set.

### 5.4 Sort

Primary sort: **Jira `age_days`, descending** (longest-open first). If `priority_weight > 0`, use Jira priority (via `jira_priority_map`) as a secondary nudge so a higher-priority ticket edges above a similar-age lower-priority one. MRs with no Jira key sort by MR age and render last within their band. Show the open-duration prominently so the ordering is self-explanatory.

---

## 6. Script contract

`fetch_review_queue.py` prints one JSON object to stdout; exits non-zero on hard failure (auth, network) with a readable message on stderr.

```json
{
  "generated_at": "ISO-8601",
  "items": [
    {
      "mr": {"id": 123, "iid": 45, "project": "group/repo", "title": "...",
              "web_url": "...", "author": "jdoe", "draft": false,
              "age_days": 6, "updated_at": "ISO"},
      "jira": {"key": "COREENGDEV-1234", "summary": "...", "status": "In Review",
                "priority": "High", "age_days": 19, "reachable": true}
    }
  ]
}
```

Items pre-sorted per §5.4. Implement pagination, a small retry on `429`/`5xx` with backoff, and `--projects`/`--since`/`--unreviewed` passthrough. Never log token values.

---

## 7. Output format

A compact terminal briefing (illustrative):

```
☀️  Good morning — 4 MRs awaiting your review

 1. [COREENGDEV-1187 · 19d open · High · In Review]   ← longest-blocked
    Fix auth token refresh
    payments/api  ·  MR !231 by r.kapoor
    https://gitlab.../merge_requests/231

 2. [COREENGDEV-1234 · 11d open · Medium · In Progress]
    Add retry to webhook consumer
    payments/api  ·  MR !245 by a.singh
    https://...

 3. [COREENGDEV-1402 · 3d open · Low · In Progress]
    Tidy logging in scheduler
    libs/core  ·  MR !88 by j.doe

 4. [no Jira]  Bump spring-boot 3.3.4   (draft)
    libs/core  ·  MR !90 by j.doe
```

Writer rules: lead with the count; lead each item with the Jira key + open-duration + priority + status chip; longest-open first; mark `[no Jira]`, `(draft)`, and `[Jira … unreachable]` explicitly; keep it skimmable.

---

## 8. Edge cases

- **No Jira key** (malformed branch) → include MR, rank on MR age, label `[no Jira]`.
- **Jira 404/403 or deleted ticket** → include MR, label `[Jira COREENGDEV-x unreachable]`, don't crash.
- **Ticket already Done but MR still open** → include and flag as an anomaly (`status: Done`); it's usually worth a look.
- **`reviewer_username` unsupported** → fall back to `reviewer_id`.
- **Large result sets** → bound by `updated_within_days`; paginate fully.
- **Rate limits** → backoff on `429`; respect `RateLimit-*` headers.
- **Drafts** → included by default and labelled; no special ranking.
- **Time zones** → compute ages in local TZ; show relative ("19d open").

---

## 9. Acceptance criteria

- [ ] `/good-morning` runs end-to-end and prints the briefing.
- [ ] List = exactly the open MRs where the user is a requested reviewer (across all visible projects), bounded by `updated_within_days`.
- [ ] Each item shows its linked Jira key, **open-duration**, status, and priority — or a clear `[no Jira]` / `[unreachable]`.
- [ ] List sorted by Jira open-duration, longest first (with optional priority tiebreak).
- [ ] `unreviewed_only` correctly hides already-approved MRs when enabled.
- [ ] All secrets from env; `config.toml` controls every §4 tunable; nothing sensitive committed.
- [ ] Runs stdlib-only (or documents the parser choice); degrades gracefully on API errors instead of crashing.
- [ ] `--projects`, `--since`, `--unreviewed` flags work.
- [ ] Skill loads from `~/.claude/skills/good-morning/` and `/good-morning` is recognized.

---

## 10. Build order

1. **Verify access first** — mint a read-only GitLab PAT (`read_api`) and a Jira token; confirm a plain request to each API succeeds from where Claude Code runs. This is the main real-world risk; do it before writing the skill.
2. `gitlab.py` + `jira.py` helpers (auth, GET, pagination) — verify against one project.
3. `fetch_review_queue.py` (§5) → confirm JSON contract.
4. `SKILL.md` orchestration + output (§7).
5. `config.example.toml`, `README.md`.

> Verify each GitLab/Jira endpoint and parameter against the API **version your instance runs** (`reviewer_username`, Jira `api/2` vs `api/3`, bearer vs basic auth) before finalizing — stable but version-dependent.

---

## 11. Promoting to a plugin later (optional)

To share with the team or version centrally, wrap the folder in a plugin — **no logic changes**: add `.claude-plugin/plugin.json`, move this skill under `skills/good-morning/`, reference the script via `${CLAUDE_PLUGIN_ROOT}/skills/good-morning/scripts/...`, validate with `claude plugin validate`, and distribute via a marketplace or shared repo. Until then, the standalone skill above is the whole product.
