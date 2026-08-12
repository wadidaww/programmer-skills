# programmer-skills

> A comprehensive GitHub Copilot agent skills repository covering every role in the software industry. Equips Copilot with deep coding knowledge, best practices, system design expertise, and the full spectrum of engineering roles.

---

## Overview

This repository provides GitHub Copilot with:

- **Role-specific agent instructions** — backend, frontend, full-stack, low-latency, QA, CI/CD, production, security, research, and more
- **Domain skills** — coding standards, backend patterns, frontend patterns, system design, testing, security, performance, and CI/CD
- **Global engineering guidelines** — architecture principles, code review standards, and software development best practices

---

## Structure

```
programmer-skills/
├── .github/
│   └── copilot-instructions.md     # Global Copilot coding guidelines
│
├── agents/                          # Specialized role-based agent instructions
│   ├── architect.md                 # System architect
│   ├── backend-developer.md         # Backend engineer
│   ├── ci-engineer.md               # CI/CD engineer
│   ├── code-reviewer.md             # Code review specialist
│   ├── devops-engineer.md           # DevOps / SRE
│   ├── frontend-engineer.md         # Frontend engineer
│   ├── fullstack-engineer.md        # Full-stack engineer
│   ├── lead-engineer.md             # Tech lead / staff engineer
│   ├── low-latency-engineer.md      # High-performance / low-latency engineer
│   ├── pipeline-engineer.md         # Data & ML pipeline engineer
│   ├── production-engineer.md       # Production / reliability engineer
│   ├── researcher.md                # Research engineer
│   ├── security-engineer.md         # Application security engineer
│   └── test-engineer.md             # QA / test engineer
│
└── skills/                          # Reusable domain knowledge
    ├── backend-patterns/            # API, database, caching, messaging patterns
    ├── ci-cd/                       # CI/CD pipeline patterns and best practices
    ├── coding-standards/            # Language-agnostic and language-specific standards
    ├── frontend-patterns/           # UI component, state, and performance patterns
    ├── performance/                 # Profiling, optimization, and low-latency techniques
    ├── security/                    # OWASP, secure coding, threat modelling
    ├── system-design/               # Architecture patterns and design principles
    └── testing/                     # TDD, BDD, testing pyramid, coverage strategies
```

---

## How to Use with GitHub Copilot

### Global Instructions (always active)

Copy or symlink `.github/copilot-instructions.md` into your project. GitHub Copilot reads this file automatically and applies the guidelines to every suggestion.

### Role-Based Agents

Reference any agent file in your Copilot chat session or workspace instructions to activate that role's expertise:

```
@workspace Use the skills in agents/backend-developer.md
```

Or add role-specific instructions to your project's `.github/copilot-instructions.md`:

```markdown
<!-- In your project's copilot-instructions.md -->
<!-- Backend API service — apply backend developer expertise -->
```

### Skills (Domain Knowledge)

Invoke a skill directly in Copilot chat:

```
@workspace Apply the patterns in skills/system-design/microservices.md
```

---

## Roles Covered

| Role | File |
|---|---|
| System Architect | `agents/architect.md` |
| Backend Developer | `agents/backend-developer.md` |
| CI/CD Engineer | `agents/ci-engineer.md` |
| Code Reviewer | `agents/code-reviewer.md` |
| DevOps / SRE | `agents/devops-engineer.md` |
| Frontend Engineer | `agents/frontend-engineer.md` |
| Full-Stack Engineer | `agents/fullstack-engineer.md` |
| Tech Lead | `agents/lead-engineer.md` |
| Low-Latency Engineer | `agents/low-latency-engineer.md` |
| Pipeline Engineer | `agents/pipeline-engineer.md` |
| Production Engineer | `agents/production-engineer.md` |
| Research Engineer | `agents/researcher.md` |
| Security Engineer | `agents/security-engineer.md` |
| Test / QA Engineer | `agents/test-engineer.md` |

---

## License

MIT