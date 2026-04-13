# AI PR Review, Fix & Security Scan

An autonomous GitHub Action that reviews pull requests, fixes bugs, and runs a dedicated security scan — all governed by [node9](https://node9.ai).

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Node9](https://img.shields.io/badge/governed%20by-Node9-6366f1)](https://node9.ai)
[![Python](https://img.shields.io/badge/python-3.10%2B-3776ab)](https://python.org)

---

## What It Does

On every push to a non-main branch, the agent runs a four-step pipeline:

| Step | What happens |
|------|-------------|
| **1. Fix loop** | Reads changed files, fixes real bugs, runs tests (up to 6 turns) |
| **2. Safety check** | Reverts AI changes if tests break after fixes |
| **3. Code review** | Reviews the agent's own fixes — posts as `🔍 node9 Code Review` on your PR |
| **4. Security review** | Data-flow pass on the original diff — posts as `🔒 node9 Security Review` on your PR |

**Two possible outcomes:**

- **Agent found nothing to fix** — one PR (yours), two review comments on it
- **Agent fixed bugs** — your PR gets review comments + a separate `[node9] AI fixes` PR with the agent's changes for you to review

---

## Quick Start

### 1. Add the workflow

Create `.github/workflows/node9-review.yml` in your repo:

```yaml
name: node9 AI PR Review

on:
  push:
    branches: ['**', '!main', '!master', '!node9/**']

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Configure git
        run: |
          git config user.name "node9[bot]"
          git config user.email "bot@node9.ai"
          git fetch origin main
      - uses: node9-ai/node9-pr-agent@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          node9_api_key: ${{ secrets.NODE9_API_KEY }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          test_cmd: 'npm test'
```

### 2. Set secrets

In your repo: **Settings → Secrets and variables → Actions**

| Secret | Required | Description |
|--------|----------|-------------|
| `ANTHROPIC_API_KEY` | Yes | Anthropic API key — [console.anthropic.com](https://console.anthropic.com) |
| `NODE9_API_KEY` | No | node9 SaaS key for audit trail — [node9.ai](https://node9.ai) |

That's it. Push to any feature branch and the agent runs automatically.

---

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `anthropic_api_key` | Yes | — | Anthropic API key |
| `github_token` | Yes | — | GitHub token for PR operations |
| `node9_api_key` | No | — | node9 SaaS key for audit trail |
| `test_cmd` | No | `npm test` | Command to run tests |
| `base_ref` | No | `main` | Base branch to diff against |

Examples for `test_cmd`: `pytest`, `cargo test`, `go test ./...`, `npm run build && npm test`

---

## What You'll See

After each push to a feature branch, the agent posts two comments on your PR:

**`🔍 node9 Code Review`** — logic and correctness review of the agent's own fixes (or the original diff if no fixes were needed)

**`🔒 node9 Security Review`** — focused data-flow analysis of the original PR diff, looking for:
- User-controlled inputs reaching filesystem sinks (`path.join`, `open()`)
- Execution sinks (`exec`, `spawn`, `subprocess`)
- Network sinks (URLs constructed from input)
- Deserialization of untrusted input
- Validation gaps (blocklist vs allowlist, unanchored regex)

Findings are rated **HIGH / MEDIUM / LOW**.

---

## Security Model

The agent runs with `node9` in `standard` mode — every tool call (file read, file write, bash command) is logged to the node9 audit trail and dangerous operations are blocked.

**Prompt injection mitigations:**
- Security review instructions live in the system message, separate from the untrusted diff
- The diff is explicitly labelled as untrusted data in the user message
- Model output is scanned for `GITHUB_TOKEN`, `ANTHROPIC_API_KEY`, and `NODE9_API_KEY` before posting to GitHub

**Tool safety:**
- File reads and writes are sandboxed to `GITHUB_WORKSPACE` via `safe_path()`
- Tool inputs are validated against an allowlist before dispatch
- Branch names are sanitized to prevent shell injection

---

## License

Apache 2.0 — see [LICENSE](LICENSE).
