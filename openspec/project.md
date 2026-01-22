# Anthropic Subscription Optimization - Project Specification

## Vision

Provide comprehensive, educational documentation for understanding two implementation patterns that enable accessing Anthropic Claude models through subscription plans rather than pay-per-token API access.

## Domain Context

### Economic Problem

Anthropic offers two pricing models:
1. **API Pricing**: Pay-per-token ($3-75 per million tokens)
2. **Subscription Pricing**: Flat monthly fee ($20-200) with usage limits

The "buffet" economics create an arbitrage opportunity: subscription plans offer dramatically better pricing for high-volume users (10-20x savings), but are restricted to Anthropic's official tools (Claude Code, claude.ai).

### The Two Patterns

This project documents two technical approaches that enable subscription access:

#### Pattern 1: OAuth + Beta Headers
**How it works:**
1. Extract OAuth credentials from official Claude Code CLI authentication
2. Use the OAuth token to call Anthropic's API directly
3. Send specific "beta flags" headers that identify the caller as Claude Code
4. System prompt must start with "You are Claude Code..."

**Technical signature:**
```
Authorization: Bearer {oauth_token}
anthropic-version: 2023-06-01
anthropic-beta: oauth-2025-04-20,claude-code-20250219,interleaved-thinking-2025-05-14,fine-grained-tool-streaming-2025-05-14
```

**Key challenge:** OAuth tokens expire periodically, requiring automatic refresh logic.

#### Pattern 2: CLI Proxy Server
**How it works:**
1. Spawn the official Claude Code CLI as a subprocess
2. Wrap it in an HTTP server (usually OpenAI-compatible)
3. Accept HTTP requests and convert to CLI arguments
4. Parse CLI JSON output and return as HTTP response

**Technical signature:**
```
Request: POST /v1/chat/completions
→ Spawn: claude --print --output-format json --system-prompt <prompt>
→ Parse: JSON response
→ Return: OpenAI-formatted JSON
```

**Key challenge:** Subprocess overhead and error handling.

## Tech Stack

### Documentation
- **Markdown**: README.md, pattern specs
- **Mermaid diagrams**: Architecture visualization
- **Code blocks**: Python, TypeScript, Go, Rust examples

### External Resources
- Links to existing implementations (GitHub repos)
- Official Anthropic documentation
- Community discussions (Hacker News, VentureBeat)

## Target Audience

1. **Developers**: Understanding technical patterns
2. **Researchers**: Economic and legal analysis
3. **Power Users**: Personal automation projects
4. **AI Enthusiasts**: Learning about API patterns

## Document Structure

### Main Documentation (`README.md`)
- Economic context and "why this matters"
- Architecture diagrams for both patterns
- Pros/cons comparison
- Legal and ToS considerations
- AI prompts for custom implementations
- Links to existing projects

### Pattern Specifications (`openspec/patterns/`)
- Detailed technical requirements
- Code examples in multiple languages
- Implementation challenges and mitigations

### Implementation Guides (`docs/implementation-guides/`)
- Language-specific setup instructions
- Common pitfalls and solutions
- Testing and verification steps

## Security Considerations

This documentation contains:
- ✅ Technical patterns (no secrets)
- ✅ Publicly available information
- ❌ NO private keys or tokens
- ❌ NO personal OAuth credentials

**Important:** Code examples use placeholder values (e.g., `<CLIENT_ID>`) and explain how to obtain credentials legitimately.

## Legal & Ethical Considerations

### ToS Disclaimer
Every code example and pattern description includes:
- Warning about potential Terms of Service violations
- Clarification that this is educational only
- Risk assessment (OAuth spoofing = high risk, CLI proxy = medium risk)

### Account Risk Documentation
Based on community reports:
- Some users received reviews for heavy usage patterns
- VPN + unusual behavior = higher scrutiny
- Geographic enforcement variations

## Success Metrics

1. **Searchability**: People searching for "Claude subscription bypass" or similar find this repo
2. **Clarity**: Developers can understand the patterns without prior knowledge
3. **Completeness**: All key concepts explained with working examples
4. **Safety**: Legal warnings are prominent and accurate
5. **Utility**: People can use the AI prompts to build custom implementations

## Maintenance

### When to Update
- Anthropic changes OAuth endpoints or flow
- Beta flags are modified or deprecated
- Claude CLI updates break compatibility
- New community implementations are discovered
- Legal/ToS enforcement changes

### Update Process
1. Document change in `docs/changes/`
2. Update affected pattern specs
3. Update README overview
4. Update AI prompts if needed
5. Git commit with descriptive message

## Related Projects

### Existing Implementations (reference only)
- `horselock/claude-code-proxy` (67 stars)
- `mergd/ccproxy`
- `phrazzld/switchboard`

### Documentation Sources
- Anthropic official docs
- Claude Code GitHub repository
- OpenCode documentation
- Hacker News discussions

## Non-Goals

- ❌ NOT providing a runnable implementation
- ❌ NOT hosting tools for bypassing restrictions
- ❌ NOT encouraging ToS violations
- ❌ NOT storing private credentials

## Goals

- ✅ Educate about technical patterns
- ✅ Document economic context
- ✅ Explain legal considerations
- ✅ Provide AI prompts for legitimate use cases
- ✅ Aggregate community knowledge

---

**Status**: 🔵 Draft - Initial documentation created

**Next Steps**:
1. Review and refine README
2. Complete pattern specifications
3. Add language-specific implementation guides
4. Publish to GitHub
5. Share with community
