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
