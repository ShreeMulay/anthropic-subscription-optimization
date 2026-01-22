# Claude Subscription Bridge - Project Conventions

## Project Purpose

Educational documentation for understanding two Anthropic Claude subscription access patterns.

**This is NOT a runnable implementation** — it documents how existing tools work.

## Directory Structure

```
anthropic-subscription-optimization/
├── README.md                    # Main documentation (with Mermaid diagrams)
├── LEGAL.md                     # Detailed legal/ToS considerations
├── AGENTS.md                    # This file - conventions
├── openspec/
│   └── project.md              # Technical specification
└── .gitignore
```

## Writing Conventions

### Tone
- Educational and informative
- Clear about risks and consequences
- Not encouraging ToS violations

### Code Examples
- Working code with language tags
- Placeholder values for secrets (e.g., `<CLIENT_ID>`)
- Comments explaining key concepts

### Legal Disclaimers
- Prominent warnings at top of README
- Per-pattern risk levels indicated
- Link to LEGAL.md for details

### Diagrams
- Use Mermaid.js (GitHub renders natively)
- Sequence diagrams for flows
- Flowcharts for decision trees

## Reference Implementations

For runnable code, see:
- [horselock/claude-code-proxy](https://github.com/horselock/claude-code-proxy) (67 stars)
- [mergd/ccproxy](https://github.com/mergd/ccproxy)
- [phrazzld/switchboard](https://github.com/phrazzld/switchboard)

## Maintenance

Update when:
- Anthropic changes OAuth endpoints or beta flags
- Claude CLI updates break compatibility
- New community implementations discovered
- Legal/ToS enforcement changes

## Non-Goals

- ❌ Providing runnable bypass tools
- ❌ Hosting credentials or tokens
- ❌ Encouraging production use
- ❌ Helping with commercial arbitrage
