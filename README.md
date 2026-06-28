---
name: kiro-ensemble
description: Spec-driven multi-agent development workflow with lean context management, incremental documentation, and automated code review.
author: Mouafak.Maiza@proton.me (Moafak Maiza)
version: 2.4.0
---

# KiroEnsemble (v2)

Spec-driven multi-agent development workflow with lean context management, incremental documentation, and automated code review.

## Agent Roster

| Agent | Model | Role | Tools |
|-------|-------|------|-------|
| **team-lead** | opus | Orchestrates workflow, delegates tasks, creates MRs | read, subagent, todo, Jira MCP, GitLab MCP |
| **builder** | opus | Implements tasks, commits, runs sync-specs | read, write, shell |
| **validator** | sonnet | Verifies against spec + eslint + tests | read, shell (readonly) |
| **reviewer** | sonnet | Code review against dev, reports findings | read, shell (readonly) |
| **documenter** | sonnet | Incremental docs in app_docs/ | read, write |

## Workflow

```
1. Load spec + domain context (app_docs)
2. Branch setup (read .git/HEAD, create branch if on dev)
3. Create TODO list from tasks.md
4. Per task:
   ├─ builder implements → returns report
   ├─ validator verifies (reads files fresh)
   ├─ builder commits (conventional commit message)
   └─ documenter documents (reads files fresh)
5. Code review (reviewer analyzes full diff)
   └─ Fix loop if findings: builder → validator → commit
6. Sync specs (builder runs sync-specs skill)
7. Final commits (docs + specs) + push
8. MR creation via GitLab MCP (optional)
9. Report
```

## Key Design Decisions

**Lean context management** - Team-lead never reads implementation files. It holds spec + app_docs + builder reports only. Agents read files fresh themselves.

**Incremental documentation** - Documenter runs after each task (not at the end) for focused context.

**Separation of concerns** - Each agent has one job. No agent crosses boundaries.

**Bounded retries** - Max 3 attempts per task with progressive context enrichment. Stage 3 uses validator as diagnostician.

**Commit per task** - Clean git history with conventional commit messages via skill.

## File Structure

```
.kiro/
├── agents/
│   ├── team-lead.json + team-lead-prompt.md
│   ├── builder.json + builder-prompt.md
│   ├── validator.json + validator-prompt.md
│   ├── reviewer.json + reviewer-prompt.md
│   ├── documenter.json + documenter-prompt.md
│   └── README.md
├── settings/
│   └── git-convention-example.json   # workspace, repos, branch/PR format
├── steering/
│   ├── atlassian-mcp-rules.md
│   ├── branching-convention.md
│   ├── ears-reference.md
│   ├── jira-agent-workflow.md
│   ├── jira-ticket-template.md
│   └── response-preferences.md
├── skills/
│   ├── address-pr-review/
│   ├── code-review/
│   ├── record-session/
│   ├── refine-ticket/
│   └── sync-specs/
├── templates/
│   ├── documentation-template.md
│   └── incident-report.md
├── scripts/
│   └── log-session.sh
├── hooks/
│   ├── agentic-team.kiro.hook
│   ├── log-session.kiro.hook
│   ├── check-careful.sh
│   └── init-session-db.sh
└── specs/
    └── {feature-name}/
        ├── requirements.md
        ├── design.md
        └── tasks.md

app_docs/
├── backend/    # Domain docs (one file per service/feature)
└── frontend/   # Domain docs (one file per screen/feature)
```

## Usage

### Start the workflow

Trigger the `Agentic Team` hook, or swap to team-lead directly:

```
/agent swap → team-lead
```

Then:
```
Execute the spec in .kiro/specs/feature-name/
```

### Prerequisites

- Feature branch checked out (or be on dev - team-lead will create one)
- Spec authored in `.kiro/specs/{feature}/`
- MCP servers configured inline in each agent config (`.kiro/agents/*.json`)

## Skills

| Skill | Used by | Purpose |
|-------|---------|---------|
| code-review | reviewer | Holistic code review against dev |
| sync-specs | builder | Update specs to match implementation |
| refine-ticket | team-lead | Refine Jira ticket structure |
| address-pr-review | builder | Fetch PR review comments, implement fixes, validate, commit |
| record-session | all | Log the agent session to the local SQLite database |

## Session Logging (Observability)

Every run can be logged to a local SQLite database for traceability and metrics. This is the source of the usage numbers the framework reports (success rate, tickets, tool usage).

**Database:** `~/.kiro/logs/agent-sessions.db` (lives outside the repo, not git-tracked)

### How it's triggered

Two ways, both run the same `record-session` flow:

- **Automatic** — the `log-session` hook fires on the `agentStop` event at the end of a session, extracts the session data, and logs it. Trivial sessions (a plain question, no real work) are skipped.
- **Manual** — invoke the `record-session` skill with a phrase like "record session", "log session", or "end session".

### First-run vs. update

The log script (`.kiro/scripts/log-session.sh`) is self-initializing. On the first call it detects the DB is missing and runs `init-session-db.sh` to create it and the schema; on every subsequent call it just inserts. Re-logging the same `session_id` upserts (`INSERT OR REPLACE`), so a session can be corrected without duplicating.

### What gets logged

Three tables:

| Table | What it captures |
|-------|------------------|
| `sessions` | One row per session: `id`, `started_at`/`ended_at`, `agent_name`, `project`, `task_summary`, `jira_ticket`, `outcome` (`success`/`partial`/`failed`/`abandoned`), `issues_discovered` |
| `tool_calls` | One row per tool call: `tool_name`, `arguments_summary`, `result_summary`, `exit_code`, `success`, `duration_ms`, `timestamp`. Covers MCP calls, git ops, file writes, lint/test runs, and subagent delegations |
| `compliance_checks` | One row per workflow step: `step_name`, `passed`, plus optional `evidence` (commit SHAs, PR numbers) and `notes` on why a step ran or was skipped |

Indexes on agent, project, and date make "what's covered / what failed / what ran" queries fast.

### Honesty constraints

The skill is explicit: do not fabricate tool calls, do not invent a `started_at` to manufacture a duration, and do not populate `evidence` with anything but real artifacts. Durations come from real tool-call timestamps or an explicit start time, falling back to a zero-width session when no timing data exists.

### Example query

```sql
-- success rate and ticket count over all sessions
SELECT outcome, COUNT(*) FROM sessions GROUP BY outcome;
SELECT COUNT(DISTINCT jira_ticket) FROM sessions WHERE jira_ticket != '';
```

## Retry Protocol

| Stage | Strategy |
|-------|----------|
| 1 | Initial dispatch |
| 2 | Re-dispatch with failure context |
| 3 | Validator diagnoses, builder gets root-cause analysis |
| 4 | Incident report + halt |

## License

Released under the [MIT License](LICENSE). Copyright (c) 2026 Moafak Maiza (Mouafak.Maiza@proton.me).

If you use this project or any substantial part of it, the MIT license requires that the copyright notice and attribution be retained.
