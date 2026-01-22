<div align="center">

# 🎯 Anthropic Subscription Optimization

**Educational Documentation & Implementation Patterns for Claude Access**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Documentation](https://img.shields.io/badge/Status-Educational-blue)](https://github.com/shreemulay/anthropic-subscription-optimization)

*Understanding how to leverage Claude subscriptions efficiently*

</div>

---

## 📖 Overview

This repository documents two implementation patterns for accessing Anthropic Claude models using your Claude Max/Pro subscription instead of the pay-per-token API. These techniques enable significant cost savings (often 10x or more) while maintaining access to Claude's capabilities.

> ⚠️ **Educational Purpose Only**: This documentation is for learning and understanding API integration patterns. Usage may be subject to Anthropic's Terms of Service.

---

## 💰 Economic Context: Why This Matters

### The "Buffet Analogy"

Anthropic offers two pricing models:

| Model | API (pay-per-token) | Claude Max Subscription |
|-------|---------------------|-------------------------|
| Claude Opus 4.5 | $15/M input, $75/M output | Unlimited* (flat monthly fee) |
| Claude Sonnet 4.5 | $3/M input, $15/M output | Unlimited* (flat monthly fee) |

\*\*Within usage limits and Anthropic's fair use policy

### Real-World Impact

For a developer working extensively with Claude:

- **Monthly Usage**: ~10M tokens (typical heavy usage)
- **API Cost**: ~$150-450/month (pay-per-token)
- **Subscription Cost**: $20-200/month (Claude Pro/Max)
- **Potential Savings**: **5-20x** depending on usage

> *"In a month of Claude Code, it's easy to use so many LLM tokens that it would have cost you more than $1,000 if you'd paid via the API."*
> — Hacker News Discussion, January 2026

---

## 🏗️ Two Implementation Patterns

### Pattern 1: OAuth Token + Beta Headers

A technique that extracts OAuth credentials from the official Claude Code CLI and uses them to make direct API calls.

```
┌─────────────────────────────────────────────────────────────┐
│                    Flow Architecture                        │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Your App ──→ Extract OAuth ──→ Refresh Token ──→ API Call   │
│     │              from Claude Code          │               │
│     │                credentials              │               │
│     │                                         ▼               │
│     │                              anthropic.com/v1/mess     │
│     │                                 with special:          │
│     │                                 - Bearer token         │
│     │                                 - Beta headers         │
│     │                                 - System prefix        │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

#### Key Components

**1. OAuth Token Extraction**

```python
# Load OAuth credentials from Claude Code auth file
OPENCODE_AUTH_PATHS = [
    Path.home() / ".local" / "share" / "opencode" / "auth.json",
    Path.home() / ".opencode" / "data" / "auth.json",
    Path.home() / ".config" / "opencode" / "auth.json",
]

def load_oauth_credentials() -> Optional[dict]:
    for auth_path in OPENCODE_AUTH_PATHS:
        if auth_path.exists():
            data = json.loads(auth_path.read_text())
            anthropic_auth = data.get("anthropic", {})
            if anthropic_auth.get("type") == "oauth":
                return {
                    "access": anthropic_auth.get("access"),
                    "refresh": anthropic_auth.get("refresh"),
                    "expires": anthropic_auth.get("expires", 0),
                    "auth_path": auth_path,
                }
    return None
```

**2. Token Refresh**

```python
async def refresh_oauth_token(refresh_token: str) -> Optional[dict]:
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "https://console.anthropic.com/v1/oauth/token",
            json={
                "grant_type": "refresh_token",
                "refresh_token": refresh_token,
                "client_id": "<CLIENT_ID_FROM_OFFICIAL_APP>",
            },
        )
        if response.is_success:
            data = response.json()
            return {
                "access": data.get("access_token"),
                "refresh": data.get("refresh_token"),
                "expires": int(time.time() * 1000) + data.get("expires_in", 3600) * 1000,
            }
    return None
```

**3. Identity Spoofing Headers**

```python
# These headers tell Anthropic "I am Claude Code CLI"
CLAUDE_CODE_BETA_FLAGS = (
    "oauth-2025-04-20,"
    "claude-code-20250219,"
    "interleaved-thinking-2025-05-14,"
    "fine-grained-tool-streaming-2025-05-14"
)

CLAUDE_CODE_SYSTEM_PREFIX = "You are Claude Code, Anthropic's official CLI for Claude."

def call_anthropic_with_oauth(
    model: str, 
    prompt: str, 
    access_token: str,
    system_prompt: Optional[str] = None
):
    headers = {
        "authorization": f"Bearer {access_token}",
        "anthropic-version": "2023-06-01",
        "anthropic-beta": CLAUDE_CODE_BETA_FLAGS,
        "content-type": "application/json",
    }
    
    # System prompt MUST start with the Claude Code prefix
    final_system = CLAUDE_CODE_SYSTEM_PREFIX
    if system_prompt:
        final_system = f"{CLAUDE_CODE_SYSTEM_PREFIX}\n\n{system_prompt}"
    
    payload = {
        "model": model,
        "max_tokens": 4096,
        "messages": [{"role": "user", "content": prompt}],
        "system": final_system,
    }
    # ... make API call
```

#### Pros/Cons

| Pros | Cons |
|------|------|
| ✅ No CLI dependency required | ❌ Needs OAuth extraction logic |
| ✅ Direct API calls (fast) | ❌ Tokens expire, need refresh |
| ✅ Fallback to API key possible | ❌ ToS violation risk |
| ✅ Works with any HTTP client | ❌ OAuth changes may break it |

---

### Pattern 2: Claude CLI Proxy Server

A technique that spawns the official Claude Code CLI as a subprocess and wraps it in an HTTP API.

```
┌─────────────────────────────────────────────────────────────┐
│                    Flow Architecture                        │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Your App ──→ HTTP Request ──→ Proxy Server ──→ Claude CLI   │
│     │                             │              │            │
│     │                             │              ▼            │
│     │                             │         Subprocess       │
│     │                             │         (claude --print)  │
│     │                             │              │            │
│     │                             ▼              ▼            │
│     │                          Parse JSON   official API      │
│     │                         Convert to                      │
│     │                         OpenAI format                   │
│     │                             │                          │
│     └────────────────────────────▼──────────────────────────┘
│                          Response to app                      │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

#### Key Components

**1. Spawn Claude CLI Subprocess**

```typescript
import { spawn } from 'child_process';

async function spawnClaude(prompt: string, system?: string): Promise<any> {
  const args = ['--print', '--output-format', 'json'];
  
  if (system) {
    args.push('--system-prompt', system);
  }
  args.push(prompt);

  const proc = Bun.spawn(['claude', ...args], {
    stdout: 'pipe',
    stderr: 'pipe',
    stdin: 'pipe',
  });

  const stdout = await new Response(proc.stdout).text();
  const stderr = await new Response(proc.stderr).text();
  const exitCode = await proc.exited;

  if (exitCode !== 0) {
    throw new Error(`Claude CLI failed: ${stderr}`);
  }

  return JSON.parse(stdout);
}
```

**2. Create OpenAI-Compatible HTTP Server**

```typescript
import { Bun } from 'bun';

function convertToOpenAIFormat(claudeResponse: any, requestModel: string) {
  return {
    choices: [{
      message: {
        role: 'assistant',
        content: claudeResponse.result,
      },
      finish_reason: claudeResponse.is_error ? 'error' : 'stop',
    }],
    usage: {
      prompt_tokens: claudeResponse.usage.input_tokens,
      completion_tokens: claudeResponse.usage.output_tokens,
      total_tokens: claudeResponse.usage.input_tokens + claudeResponse.usage.output_tokens,
    },
  };
}

const server = Bun.serve({
  port: 8765,
  async fetch(req) {
    if (req.url.endsWith('/v1/chat/completions')) {
      const body = await req.json();
      const prompt = body.messages.map(m => `${m.role}: ${m.content}`).join('\n');
      
      const claudeResponse = await spawnClaude(prompt, body.system);
      const openAIResponse = convertToOpenAIFormat(claudeResponse, body.model);
      
      return Response.json(openAIResponse);
    }
  },
});
```

**3. Run in Background**

```bash
# Using tmux
tmux new-session -d -s claude-proxy "bun start"

# Using Docker
docker run -d --name claude-proxy -p 8765:8765 ClaudeProxy
```

#### Pros/Cons

| Pros | Cons |
|------|------|
| ✅ Official CLI (no API spoofing) | ❌ Requires Claude CLI installed |
| ✅ Automatic auth handling | ❌ Subprocess overhead |
| ✅ Works with ANY OpenAI client | ❌ Slower than direct API |
| ✅ Minimal ToS risk | ❌ CLI updates may break it |

---

## 🆚 Pattern Comparison

| Aspect | OAuth + Headers | CLI Proxy |
|--------|-----------------|-----------|
| **Setup Complexity** | Medium (OAuth flow) | Low (just spawn CLI) |
| **Performance** | Fast (direct API) | Medium (subprocess) |
| **Stability** | Low (beta flags change) | Medium (CLI updates) |
| **Auth Handling** | Manual (refresh logic) | Automatic (CLI handles) |
| **ToS Risk** | High (spoofing) | Medium (using official CLI) |
| **Use Case** | Production systems | Personal projects |

---

## 📚 Existing Implementations

These patterns are already implemented by various projects:

| Project | Pattern | Language | Stars |
|---------|---------|----------|-------|
| [horselock/claude-code-proxy](https://github.com/horselock/claude-code-proxy) | OAuth Proxy | Node.js | 67 |
| [mergd/ccproxy](https://github.com/mergd/ccproxy) | CLI Proxy | Go | - |
| [phrazzld/switchboard](https://github.com/phrazzld/switchboard) | Service Proxy | Python | - |

> 💡 **Tip**: Review these projects for production-ready code. This documentation focuses on explaining *how* they work, not providing another implementation.

---

## 🤖 AI Prompts for Custom Implementation

Below are prompts you can copy-paste to ask an AI (like Claude, GPT-4, etc.) to build a custom implementation based on these patterns.

### Prompt for OAuth + Headers Pattern

<details>
<summary>📋 Click to expand prompt</summary>

```
I need you to build a Python module that implements the Anthropic Claude OAuth access pattern.

## Requirements

1. Extract OAuth credentials from Claude Code CLI auth file (~/.local/share/opencode/auth.json, ~/.opencode/data/auth.json, or ~/.config/opencode/auth.json)
2. Implement automatic token refresh when expired
3. Make API calls to anthropic.com/v1/messages with these special headers:
   - authorization: Bearer {access_token}
   - anthropic-version: 2023-06-01
   - anthropic-beta: oauth-2025-04-20,claude-code-20250219,interleaved-thinking-2025-05-14,fine-grained-tool-streaming-2025-05-14
4. System prompt MUST start with: "You are Claude Code, Anthropic's official CLI for Claude."
5. If system_prompt provided, append it after the prefix: "{PREFIX}\n\n{USER_PROMPT}"
6. Fallback to API key authentication if OAuth fails

## Technical Details

- Use httpx for async HTTP calls
- Model mapping: anthropic/claude-opus-4.5 -> claude-opus-4-20250514, etc.
- Token refresh: POST https://console.anthropic.com/v1/oauth/token with grant_type="refresh_token"
- Refresh on expiry (with 60s buffer)
- Save updated tokens back to auth file

## Expected API

```python
async def call_anthropic(
    model: str,
    prompt: str,
    max_tokens: int = 4096,
    system_prompt: Optional[str] = None,
) -> dict:
    """Call Anthropic API - tries OAuth first, falls back to API key.
    
    Returns:
        dict with 'response', 'usage', 'provider', 'model' keys
    """
```

Please provide complete, working code with proper error handling and docs.
```
</details>

### Prompt for CLI Proxy Pattern

<details>
<summary>📋 Click to expand prompt</summary>

```
I need you to build a Claude Code CLI proxy server that exposes an OpenAI-compatible API.

## Requirements

1. Create an HTTP server (port 8765) with these endpoints:
   - GET /health - Health check
   - GET /models - List available models (claude-opus-4, claude-sonnet-4, etc.)
   - POST /v1/chat/completions - OpenAI-compatible chat completions

2. For /v1/chat/completions:
   - Accept OpenAI request format: {model, messages, temperature, max_tokens, system}
   - Convert messages to CLI prompt format: "role: content\nrole: content"
   - Spawn 'claude' CLI subprocess with args: --print --output-format json [--system-prompt SYSTEM] PROMPT
   - Parse CLI JSON response
   - Convert Claude response to OpenAI format:
     - choices[0].message.content = claude.result
     - usage.prompt_tokens = claude.usage.input_tokens
     - usage.completion_tokens = claude.usage.output_tokens
     - usage.total_tokens = input + output

3. Technologies:
   - Use Bun.serve() for the HTTP server
   - Use Bun.spawn() for CLI subprocess
   - TypeScript for type safety

## Error Handling

- CLI failure → 500 error with stderr details
- Invalid JSON → 500 error with debug info
- Timeout → 500 error

## Expected CLI Response Format

```typescript
interface ClaudeCodeResponse {
  type: 'result' | 'error';
  duration_ms: number;
  result: string;
  usage: {
    input_tokens: number;
    output_tokens: number;
  };
}
```

Please provide complete, working TypeScript code.
```
</details>

### Prompt for Multi-Language Implementation

<details>
<summary>📋 Click to expand prompt</summary>

```
I need you to help me design a multi-language implementation strategy for Anthropic Claude subscription access.

## Context

I want to document and provide code examples for TWO patterns:

1. OAuth + Beta Headers pattern (direct API calls)
2. CLI Proxy pattern (subprocess wrapper)

## Task

For each pattern, provide:

1. **Python implementation** (using httpx, http.server)
2. **TypeScript/Node.js implementation** (using axios, express)
3. **Go implementation** (using net/http, os/exec)
4. **Rust implementation** (using reqwest, tokio)

## For Each Language

Provide:
- Complete, runnable code
- Installation instructions (requirements.txt, package.json, go.mod, Cargo.toml)
- Run command (python main.py, node index.js, go run main.go, cargo run)
- Test curl command to verify
- Error handling patterns
- Comments explaining key parts

## Focus Areas

- OAuth: Token refresh, header construction, fallback logic
- CLI Proxy: Subprocess management, response parsing, OpenAI format conversion
- Both: Type safety, async handling, error handling

## Output Format

Create a folder structure:
```
/pattern1-oauth
  /python
  /typescript
  /go
  /rust
/pattern2-cli-proxy
  /python
  /typescript  
  /go
  /rust
```

Each folder contains:
- Main source file
- Dependencies file
- README.md with run instructions
- example-curl.sh to test

Please provide all implementations. Make them production-ready with proper error handling.
```
</details>

---

## ⚠️ Legal & ToS Considerations

### Terms of Service

From Anthropic's Claude Code documentation and GitHub issues:

> "The OAuth API is for Claude Code only. Using OAuth tokens with third-party applications is a violation of our Terms of Service."

**Key Points:**
- ✅ Using Claude Code CLI directly: Intended use
- ⚠️ Proxying CLI: Gray area (technical wrapper, still official CLI)
- ❌ Spoofing headers with OAuth tokens: Explicit ToS violation
- ❌ Automated scraping: May trigger abuse filters

### Account Risks

Based on community reports (Jan 2026):
- Some users received account reviews for heavy usage via third-party tools
- VPN + unusual usage patterns = higher risk
- Geographic location matters (enforcement varies)

### Economic Reality

Anthropic's crackdown in January 2026 was primarily about:
1. Preventing businesses from arbitrage (consumer plans for commercial use)
2. Protecting revenue streams from API usage
3. Technical safeguards ("tightened")

---

## 🔧 Prerequisites

### For OAuth + Headers Pattern

- Claude Code CLI installed andauthenticated
- HTTP client library (httpx, axios, etc.)
- OAuth access to your Claude account

### For CLI Proxy Pattern

- Claude Code CLI installed and authenticated
- HTTP server framework (Bun.serve, Express, etc.)
- Sub/process spawning capabilities

### Common Requirements

- Active Claude Pro or Max subscription
- Understanding of OAuth 2.0 flows
- Basic HTTP/API knowledge

---

## 🔄 Failure Modes & Mitigations

| Failure | Detection | Mitigation |
|---------|-----------|------------|
| Token expired | 401 error | Auto-refresh with refresh token |
| Beta flags changed | 400/403 error | Fallback to API key |
| CLI not found | Spawn error | Prompt user to install Claude CLI |
| CLI update breaks | Parse error | Version checking, graceful degradation |
| Rate limit | 429 error | Exponential backoff, queue |

---

## 📖 Further Reading

### Official Documentation
- [Anthropic API Documentation](https://docs.anthropic.com/)
- [Claude Code GitHub](https://github.com/anthropics/claude-code)
- [OpenCode Documentation](https://opencode.ai/docs/)

### Community Discussions
- [Hacker News - Anthropic Crackdown](https://news.ycombinator.com/item?id=46578701)
- [VentureBeat Coverage](https://venturebeat.com/technology/anthropic-cracks-down-on-unauthorized-claude-usage-by-third-party-harnesses)
- [OpenCode Issue #6930](https://github.com/anomalyco/opencode/issues/6930)

### Related Technologies
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/)
- [OpenRouter](https://openrouter.ai/) - Alternative API aggregation
- [OpenCode](https://opencode.ai/) - AI-powered code editor

---

## 🤝 Contributing

This is an educational documentation repository. Contributions welcome for:

- Additional language implementations
- Better error handling examples
- Updated patterns as Anthropic changes
- Clarifications and corrections

**Guidelines:**
1. Fork the repository
2. Create a branch for your contribution
3. Add or improve documentation
4. Submit a pull request

---

## 📄 License

MIT License - Educational use only.

> This documentation is provided for learning purposes. Usage of these techniques may violate Anthropic's Terms of Service. Users are responsible for their own compliance.

---

## 🙏 Acknowledgments

- Community members who discovered and documented these patterns
- The maintainers of existing proxy implementations
- Anthropic for providing Claude and the amazing tools they've built

---

<div align="center">

*Built for the community to understand, learn, and build responsibly.*

**⭐ Star this repo if it helped you!**

</div>
