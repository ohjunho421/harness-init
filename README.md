# harness-init

Claude Code plugin that bootstraps an **enforced agentic workflow** into any project.

Run `/harness-init` in any project to install a mandatory 8-step workflow that Claude Code actually follows.

## Problem

Setting up CLAUDE.md with workflow instructions doesn't guarantee Claude will follow them. Common failures:

- Claude skips reading docs and jumps straight to code
- No code review happens after modifications
- Available skills/plugins are never invoked
- Change history is never recorded
- No skill pre-loading before implementation (wrong patterns used)

## Solution

`harness-init` solves this with a 3-layer enforcement system:

### 1. Strong CLAUDE.md (auto-loaded every conversation)

- `MANDATORY WORKFLOW` section with MUST/ALWAYS directives
- **Request keyword → skill pre-loading** (e.g. "login" → security-review, "React" → react-patterns)
- File extension-based skill routing (`.tsx` → frontend-design, `auth/` → security-reviewer)
- Explicit `Skill` tool invocation commands (not just text references)
- "Forbidden" statements for skipping steps

### 2. Project Hooks (.claude/settings.json)

- **UserPromptSubmit**: Full 8-step workflow reminder at start of each request
- **PreToolUse (Edit/Write)**: "Did you read the docs & call the right skills first?" check
- **PostToolUse (Edit/Write)**: "Run code review + update docs" reminder

### 3. Structured Feature Docs (docs/features/*.md)

- Frontmatter with `status`, `files`, `tags`, `last_modified`
- Change history table that must be updated after every modification
- Initial docs created during setup (not empty placeholders)

## What Gets Generated

```
project/
├── CLAUDE.md                          # Mandatory 8-step workflow + skill routing
├── .claude/settings.json              # Project hooks (3 types)
└── docs/
    ├── 00-INDEX.md                    # Request -> document mapping
    ├── code-review-checklist.md       # Review feedback accumulation
    └── features/
        ├── {feature1}.md              # Structured feature doc
        ├── {feature2}.md
        └── ...
```

## The 8-Step Mandatory Workflow

| Step | Action | Enforcement |
|------|--------|-------------|
| 1 | Read docs before code | PreToolUse hook reminds on every Edit |
| 2 | **Analyze request → pre-load skills** | Keyword-based skill routing table |
| 3 | Plan if 3+ files | CLAUDE.md MUST directive |
| 4 | Implement with file-based skills | Extension/directory-based routing |
| 5 | Code review after changes | PostToolUse hook + MUST directive |
| 6 | Build verification | CLAUDE.md directive |
| 7 | Update feature docs | PostToolUse hook + specific format template |
| 8 | Commit | User-configured method |

## Skill Routing

### By Request Keyword (STEP 2 — pre-load before coding)

| Keywords | Skill | Purpose |
|----------|-------|---------|
| UI, page, component, design | `frontend-design:frontend-design` | UI design guide |
| React, hooks, state | `ccpp:react-patterns` | React 19 patterns |
| API, REST, GraphQL | `ccpp:api-design-principles` | API design |
| login, auth, OAuth, JWT | `everything-claude-code:security-review` | Security checklist |
| DB, migration, schema | `everything-claude-code:database-migrations` | DB patterns |
| Docker, deploy, CI/CD | `everything-claude-code:deployment-patterns` | Deploy strategy |
| test, TDD | `ccpp:tdd` | TDD workflow |
| Python | `everything-claude-code:python-patterns` | Python patterns |
| Go | `everything-claude-code:golang-patterns` | Go patterns |
| Rust | `everything-claude-code:rust-patterns` | Rust patterns |

> Full mapping table with 40+ entries is in SKILL.md

### By File Extension (STEP 4 — during implementation)

| Extension | Skill |
|-----------|-------|
| `.tsx`, `.jsx`, `.vue`, `.svelte` | `frontend-design:frontend-design` |
| `.css`, `.scss` | `ccpp:tailwind-design-system` |
| `.test.*`, `.spec.*` | `ccpp:tdd` |

### By Directory (STEP 4 — during implementation)

| Directory | Skill |
|-----------|-------|
| `auth/`, `login/`, `session/` | `security-reviewer` agent |
| `api/`, `routes/`, `controllers/` | `security-reviewer` agent |
| `components/`, `pages/`, `ui/` | `frontend-design:frontend-design` |

## Installation

### As a Claude Code Plugin

```bash
claude plugin add --source github ohjunho421/harness-init
```

### Manual Installation

Copy `skills/harness-init/SKILL.md` to `~/.claude/skills/harness-init/SKILL.md`

### Usage

In any project:

```
/harness-init
```

## Requirements

For full skill routing, these plugins are recommended:

- [ccpp](https://github.com/jh941213/my-claude-code-asset) — code review, planning, TDD, build-fix, React patterns
- [everything-claude-code](https://github.com/affaan-m/everything-claude-code) — security, DB, deploy, language-specific patterns
- [frontend-design](https://github.com/anthropics/claude-code-plugins) — frontend UI development
- [context7](https://github.com/anthropics/claude-code-plugins) — library documentation lookup

The harness works without these plugins but skill routing steps will be skipped.

## License

MIT
