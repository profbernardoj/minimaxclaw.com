# Standard Operating Procedures

**Created:** 2026-03-08
**Purpose:** Documented workflows for consistent, safe, and verified operations.

---

## SOP Index

| SOP # | Title | Created |
|-------|-------|---------|
| 001 | Development & Deployment | 2026-03-08 |
| 002 | Morpheus P2P Session Management | 2026-03-09 |
| 003 | XMTP Agent-to-Agent Communication | 2026-03-14 |

---

## SOP-001: Development & Deployment

**Version:** 1.0
**Created:** 2026-03-08
**Purpose:** Ensure consistent, safe, and verified code deployment across all EverClaw-related repositories.

### Overview

All development and deployment tasks follow this 6-stage process. Each stage requires David's review before proceeding to the next.

---

### Stage 1: Planning & Design

**Goal:** Define what will be built and get approval before coding.

#### Steps
1. Analyze requirements and define scope
2. Research existing code, dependencies, and patterns
3. Design the solution architecture
4. Identify affected files and repositories
5. Assess risks and mitigation strategies
6. Present comprehensive plan to David for approval

#### Deliverable
- Written plan with:
  - Problem statement
  - Proposed solution
  - Files/modules affected
  - Dependencies and integration points
  - Risk assessment
  - Estimated scope

#### Gate
**David must approve the design before proceeding to Stage 2.**

---

### Stage 2: Coding

**Goal:** Implement the approved design in manageable stages.

#### Steps
1. Create or switch to appropriate branch
2. Implement changes in logical, reviewable commits
3. **Verify all external dependency URLs, commands, and install methods against their official documentation before using them** (see External Dependency Verification below)
4. Document changes in code comments and changelogs
5. Update version numbers where applicable
6. Report progress to David after each significant stage

#### External Dependency Verification (MANDATORY)

**Never assume a URL, install command, or API endpoint for an external project.** Always verify against the project's official source before writing it into code.

**For each external dependency referenced in code:**
1. Check the project's official README or docs for the canonical install method
2. Test that URLs actually resolve (`curl -fsSL -o /dev/null -w "%{http_code}" <url>`)
3. Verify CLI commands exist and have the expected flags
4. Confirm API endpoints and response formats against official docs
5. Document the source of truth for each dependency's install/URL in a code comment

**Examples of things that MUST be verified:**
- Install URLs (e.g., `get.openclaw.ai` was assumed but doesn't exist — OpenClaw uses `npm install -g openclaw@latest`)
- CLI flags and subcommands (don't assume `--flag` exists without checking `--help`)
- API base URLs and paths
- Package registry names and slugs
- GitHub repo URLs and branch names

**If in doubt, run the command or fetch the URL in a test before committing.**

> **Lesson (BUG-014, 2026-03-12):** We shipped `curl -fsSL https://get.openclaw.ai | bash` in the installer assuming OpenClaw had a curl-pipe-bash installer. They don't — OpenClaw installs via npm. The broken URL shipped to 30 repos before a user reported it. Always verify first.

#### Deliverable
- Working code that implements the approved design
- Updated CHANGELOG.md
- Updated version in package.json or equivalent
- Confirmation that all external URLs/commands have been verified
- Summary of changes for David's review

#### Gate
**David must review the coded changes before proceeding to Stage 3.**

---

### Stage 3: Testing

**Goal:** Verify the code works as intended and fix any errors.

#### Steps
1. Run functional tests for new features
2. Run regression tests for existing functionality
3. Test edge cases and error handling
4. **Verify all external URLs resolve** (HTTP 200) and external commands execute as expected
5. Verify backward compatibility
6. Fix any bugs discovered during testing
7. Document test results

#### Deliverable
- Test report showing:
  - Tests performed
  - Results (pass/fail)
  - Bugs found and fixed
  - Verification that intended functionality works

#### Gate
**Report test results to David before proceeding to Stage 4.**

---

### Stage 4: PII Scan

**Goal:** Ensure no personal identifiable information is included in deployed code.

#### Steps
1. Scan for email addresses (exclude example.com, noreply, etc.)
2. Scan for API keys, secrets, tokens, passwords
3. Scan for private keys
4. Scan for phone numbers
5. Scan for real names (exclude public usernames)
6. Scan for local file paths with usernames
7. Scan for wallet addresses (verify only public contracts)
8. Remove or exclude any PII found
9. Stage only clean files for commit

#### Deliverable
- PII scan report showing:
  - Categories checked
  - Findings (or "clean")
  - Any files excluded from commit

#### Gate
**Report PII scan results to David before proceeding to Stage 5.**

---

### Stage 5: Primary Deployment

**Goal:** Deploy to the primary EverClaw/EverClaw repository.

#### Steps
1. Commit with descriptive message following conventional commits:
   - `feat:` for new features
   - `fix:` for bug fixes
   - `chore:` for maintenance
   - `docs:` for documentation
2. Verify commit includes all intended changes
3. Push to `EverClaw/EverClaw` (everclaw-org remote)
4. Create and push version tag (e.g., `v2026.3.8`)
5. Verify deployment via `git ls-remote`

#### Deliverable
- Deployment confirmation with:
  - Commit hash
  - Tag version
  - Remote verification (commit hash match)
  - Files changed count

#### Gate
**Report deployment to David before proceeding to Stage 6.**

---

### Stage 6: Ecosystem Deployment

**Goal:** Deploy to all related repositories sequentially with verification.

#### Steps
1. List all repositories to update (flavor repos, forks, etc.)
2. Deploy to each repository one at a time:
   - Push main branch with `--force-with-lease`
   - Verify commit hash matches primary
   - Record success/failure
3. Push tags to all repositories
4. Verify tag deployment
5. Generate final deployment report

#### Repositories (as of 2026-03-08)

| Repository | Remote Name |
|------------|-------------|
| EverClaw/EverClaw (org) | everclaw-org |
| profbernardoj/everclaw | origin |
| 28 flavor repos | Various |

#### Deliverable
- Final report with:
  - Total repos deployed
  - Each repo status (✅/❌)
  - Commit hash verification
  - Tag verification

#### Gate
**Final report to David confirming all deployments complete.**

---

### Stage 7: Package Tag & Docker Image

**Goal:** Create a versioned release tag on EverClaw/EverClaw and trigger Docker image build via GitHub Actions.

#### Steps
1. Bump version in: `package.json`, `SKILL.md`, `Dockerfile` (EVERCLAW_VERSION), `docker-compose.yml`
2. Commit version bump: `git commit -m "release: vYYYY.M.DD — <summary>"`
3. Create annotated tag: `git tag -a vYYYY.M.DD -m "<summary>"`
4. Push to upstream with tags: `git push upstream main --tags`
5. Verify `.github/workflows/docker-build.yml` has correct `OPENCLAW_VERSION` build arg
   - If not: update via GitHub API (`.github` is gitignored locally)
6. Monitor GitHub Actions build: `gh run list --repo EverClaw/EverClaw --limit 3`
7. Verify Docker image published: `gh api orgs/EverClaw/packages/container/everclaw/versions --jq '.[0]'`
8. Update `MEMORY.md` with new version and Docker image status

#### Docker Workflow Details
- Workflow: `.github/workflows/docker-build.yml`
- Triggers: push to `main`, push of `v*` tags, manual `workflow_dispatch`
- Registry: `ghcr.io/everclaw/everclaw`
- Tags: `latest` + version from `package.json` (on main push), git tag (on tag push)
- Platforms: `linux/amd64`, `linux/arm64`
- Build args: `EVERCLAW_VERSION` (from package.json), `OPENCLAW_VERSION` (hardcoded in workflow)

#### Deliverable
- Version tag visible on GitHub releases
- Docker image at `ghcr.io/everclaw/everclaw:<version>` and `:latest`
- GitHub Actions run ID and status

#### Gate
**Verify Docker image is pullable before reporting complete to David.**

---

### Quick Reference

| Stage | Name | Gate |
|-------|------|------|
| 1 | Planning & Design | David approves design |
| 2 | Coding | David reviews code |
| 3 | Testing | Report results to David |
| 4 | PII Scan | Report clean scan to David |
| 5 | Primary Deploy | Report to David |
| 6 | Ecosystem Deploy | Final report to David |
| 7 | Package Tag & Docker | Verify image pullable |

---

### Example: PromptGuard v3.3.0 Upgrade

**Date:** 2026-03-08

| Stage | Duration | Result |
|-------|----------|--------|
| 1. Planning | 5 min | Approved design for v3.3.0 sync |
| 2. Coding | 20 min | 37 files changed, +8591/-2136 |
| 3. Testing | 10 min | All 7 test categories passed |
| 4. PII Scan | 5 min | Clean (plist excluded) |
| 5. Primary Deploy | 2 min | f2429e3 → EverClaw/EverClaw |
| 6. Ecosystem | 5 min | 29 repos deployed, all verified |

**Total time:** ~47 minutes

---

### Notes

- Never skip stages or gates
- Document everything in memory files (`memory/daily/YYYY-MM-DD.md`)
- Update this SOP as process improves
- Reference SOP-001 when David assigns coding tasks

---

## SOP-002: Morpheus P2P Session Management

**Version:** 1.1
**Created:** 2026-03-09
**Updated:** 2026-03-09
**Purpose:** Manage Morpheus P2P staking sessions for decentralized inference — wallet operations, session lifecycle, balance monitoring, and fallback behavior.

### How Morpheus Inference Staking Works

**CRITICAL: MOR is STAKED, not spent.** There are two modes on the Morpheus network:

1. **Staking Mode (`directPayment:false`)** — MOR is temporarily locked in the Diamond smart contract for the session duration. After the session closes, **all staked MOR is returned** to the consumer's wallet (with a short delay). This is how our router is configured.

2. **Direct Payment Mode (`directPayment:true`)** — MOR is paid directly to the provider and NOT returned. This mode is not used in our setup.

**Our configuration uses Staking Mode.** The MOR does NOT "run out." It is locked temporarily per session, then returned. The only real costs are:
- **ETH gas** for opening/closing session transactions (~0.001 ETH per session)
- **Opportunity cost** — staked MOR cannot be used for other sessions simultaneously

**Key implications:**
- You need enough MOR to cover **one session's stake at a time**, not cumulative
- When a session expires and closes, the staked MOR returns to the wallet
- The `SESSION_CLOSER` component in the proxy-router auto-closes expired sessions
- There may be a delay between session close and MOR return (blockchain processing)
- Multiple concurrent sessions require enough MOR to cover all stakes simultaneously

**Verification:** Router logs confirm `directPayment:false` on all session attempts.

### Overview

Morpheus P2P inference requires staking MOR tokens to open sessions with providers. The staked MOR is returned after the session closes. This SOP documents the architecture, operational procedures, and upgrade roadmap for reliable session management.

---

### Architecture

```
OpenClaw → morpheus-proxy (port 8083) → proxy-router (port 8082) → Morpheus P2P Network
                  ↓ (fallback)
           Morpheus Gateway API (api.mor.org)
                  ↓ (fallback)
           Venice API (api.venice.ai)
```

**Key Components:**
| Component | Location | Purpose |
|-----------|----------|---------|
| morpheus-proxy.mjs | `~/morpheus/proxy/` + `skills/everclaw/scripts/` | OpenAI-compatible proxy with balance monitoring |
| proxy-router | `~/morpheus/proxy-router` | Lumerin binary — session creation, provider selection |
| safe-transfer.mjs | `skills/everclaw/scripts/` | Transfer MOR from Safe → Router wallet |
| everclaw-wallet.mjs | `skills/everclaw/scripts/` | Wallet ops: balance, swap, approve, stake |

**Wallet Addresses:**
| Address | Role |
|---------|------|
| *(set via `ROUTER_WALLET` env var)* | Router EOA (sessions, gas) |
| *(set via `SAFE_WALLET` env var)* | Safe Smart Account (MOR reserve) |

**Key Contracts (Base Mainnet):**
| Contract | Address |
|----------|---------|
| MOR Token | `0x7431aDa8a591C955a994a21710752EF9b882b8e3` |
| Diamond (Staking) | `0x6aBE1d282f72B474E54527D93b979A4f64d3030a` |
| Uniswap Router | `0x7b9684855E9d52c67C29c810E4240662c9d2e0d0` |
| Uniswap Quoter | `0xC5290058841028F1614F3A6F0F5816cAd0df5E27` |

---

### Procedure 1: Check Session Health

**When:** Before any session-dependent work, or when inference errors occur.

```bash
# 1. Check proxy health (balance, sessions, fallback mode)
curl -s http://127.0.0.1:8083/health | jq '.'

# 2. Check router logs for session errors
tail -50 ~/morpheus/data/logs/router-stdout.log | grep -i "session\|error\|stake"

# 3. Check MOR balance on-chain
node scripts/everclaw-wallet.mjs balance
```

**Healthy indicators:**
- `morBalance` > 500 MOR
- `fallbackMode` = false
- `consecutiveFailures` = 0
- At least 1 active session

---

### Procedure 2: Fund Router Wallet

**When:** `morBalance` is less than the provider's stake requirement for a single session (~633 MOR for 1-day sessions), or session open fails with "transfer amount exceeds balance"

**Remember:** MOR is staked, not spent. The staked MOR returns after each session closes. You only need enough to cover concurrent session stakes.

```bash
# 1. Check current balances
node scripts/safe-transfer.mjs <amount>        # Dry run

# 2. Execute transfer from Safe → Router
node scripts/safe-transfer.mjs <amount> --execute

# 3. Verify
curl -s http://127.0.0.1:8083/health | jq '.morBalance'
```

**Guidelines:**
- Need enough MOR to cover **one session's stake** (currently ~633 MOR per 1-day session)
- For concurrent sessions across multiple models, multiply by number of simultaneous sessions
- Staked MOR is returned after each session closes (with a short delay)
- Safe holds reserve MOR (currently 5,700 MOR) for additional funding if needed
- Only ETH gas is permanently consumed (~0.001 ETH per session open/close)

---

### Procedure 3: Adjust Session Duration

**When:** Balance is low but sufficient for shorter sessions.

```bash
# Edit launchd plist
# MORPHEUS_SESSION_DURATION: seconds (default 86400 = 1 day)
vim ~/Library/LaunchAgents/com.morpheus.proxy.plist

# Restart proxy
launchctl unload ~/Library/LaunchAgents/com.morpheus.proxy.plist
launchctl load ~/Library/LaunchAgents/com.morpheus.proxy.plist
```

**Duration vs Stake (approximate, MOR is returned after session close):**
| Duration | Stake Locked | Returned After Close |
|----------|-------------|---------------------|
| 1 day | ~633 MOR | ✅ ~633 MOR returned |
| 3 days | ~1,900 MOR | ✅ ~1,900 MOR returned |
| 7 days | ~4,434 MOR | ✅ ~4,434 MOR returned |

**Note:** Longer sessions lock more MOR but it's all returned. The trade-off is liquidity — locked MOR can't be used for other sessions until the current one closes.

---

### Procedure 4: Model Selection

**When:** Selecting models for P2P inference.

**Available on P2P (as of 2026-03-15):**
| Model | Type | Notes |
|-------|------|-------|
| **glm-5** | General | ✅ Best quality GLM, now on P2P |
| **glm-5:web** | Web search | GLM-5 with web access |
| glm-4.7 | General | Previous gen GLM |
| glm-4.7-flash | Fast | Lightweight tasks |
| glm-4.7-thinking | Reasoning | Thinking model |
| glm-4.6 | General | Older GLM |
| kimi-k2-thinking | Reasoning | Best for complex tasks |
| kimi-k2.5 | General | High quality |
| kimi-k2.5:web | Web search | With web access |
| qwen3-235b | Large | 235B parameters |
| qwen3-next-80b | General | 80B next-gen |
| qwen3-coder-480b-a35b-instruct | Code | 480B code specialist |
| llama-3.3-70b | General | Llama 3.3 |
| llama-3.2-3b | Small | Lightweight |
| gpt-oss-120b | Large | Open-source GPT |
| mistral-31-24b | Fast | Mistral |
| MiniMax-M2.5 | General | MiniMax |
| hermes-3-llama-3.1-405b | Large | Hermes |
| hermes-4-14b | General | Hermes 4 |
| venice-uncensored | Uncensored | Dolphin-Mistral |

**Also available:** `:web` variants for most models (web search enabled)

**Check live model list:**
```bash
curl -s -u "admin:$(cat ~/morpheus/.cookie)" http://127.0.0.1:8082/blockchain/models | jq '.models[].Name' | sort
```

---

### Procedure 5: Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `transfer amount exceeds balance` | Not enough MOR | Fund router (Procedure 2) |
| `Unknown model: X` | Model not on P2P | Use Gateway or different model |
| `502 Morpheus session unavailable` | Session creation failed | Check balance, restart proxy |
| High `consecutiveFailures` | Persistent P2P issues | Will auto-fallback after threshold |
| `fallbackMode: true` | Using Gateway instead of P2P | Wait for retry timer or restart proxy |

---

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| MORPHEUS_SESSION_DURATION | 86400 | Session duration in seconds |
| MORPHEUS_RENEW_BEFORE | 3600 | Renew this many seconds before expiry |
| MORPHEUS_WALLET_ADDRESS | (required) | Router EOA address for balance checks |
| MORPHEUS_MOR_THRESHOLD | 500 | Warn when MOR below this |
| MORPHEUS_FALLBACK_THRESHOLD | 3 | P2P failures before Gateway fallback |
| MORPHEUS_FALLBACK_RETRY_MS | 21600000 | How long to stay in fallback (6 hours) |
| MORPHEUS_GATEWAY_API_KEY | (optional) | Gateway API key for fallback |
| MORPHEUS_GATEWAY_URL | https://api.mor.org/v1 | Gateway base URL |

---

### Upgrade Roadmap

| Priority | Feature | Status | Description |
|----------|---------|--------|-------------|
| P0 | Balance monitoring | ✅ Done | Check MOR before sessions |
| P0 | Hybrid fallback | ✅ Done | P2P → Gateway after failures |
| P1 | Provider bid query | ⬜ Planned | Show stake needed before attempting |
| P1 | Auto-adjust duration | ⬜ Planned | Calculate max duration from balance |
| P1 | Session health endpoint | ✅ Done | /health shows balance + sessions |
| P2 | Provider preferences | ⬜ Planned | Filter by max bid or whitelist |
| P2 | Session analytics log | ⬜ Planned | Track stake, cost, provider performance |
| P2 | Safe executor support | ⬜ Planned | Execute sessions directly from Safe |
| P3 | Multi-agent session sharing | ⬜ Planned | Share sessions across agents |

---

### Lessons Learned (2026-03-09, Updated 2026-03-15)

1. **MOR is STAKED, not spent** — All staked MOR is returned after session close. `directPayment:false` confirmed in router logs. Only ETH gas is consumed.
2. **Session stake is pro-rated by duration** — 4,434 MOR/7 days = 633 MOR/day (all returned after close)
3. **Need enough for one session at a time** — MOR doesn't "run out"; you just need enough unlocked MOR to cover the next session's stake
4. **MOR must be in EOA, not Safe** — Router only uses EOA for signing; Safe funds need to be transferred to EOA first
5. **GLM-5 IS on P2P** — As of Mar 15, GLM-5 has providers on P2P. Model list must be refreshed from router (`/blockchain/models`).
6. **Check balance proactively** — Prevents cryptic "ERC20: transfer amount exceeds balance" revert errors
7. **Provider bids vary** — Can't control which provider the router selects; stake amount depends on provider bid
8. **Safe 1-of-2 threshold** — Single owner can execute transfers from Safe to EOA
9. **SESSION_CLOSER auto-closes** — The router has a built-in component that auto-closes expired sessions and triggers MOR return
10. **Model list refresh bug (Fixed 2026-03-15)** — Proxy was parsing router response incorrectly. Router returns `{ models: [...] }`, not a raw array. Fix: `const models = Array.isArray(data) ? data : (data.models || [])`
11. **Model list changes frequently** — Always check live models via `/blockchain/models` endpoint. Don't rely on stale notes from previous sessions.
12. **40+ models now on P2P** — Including glm-5, glm-5:web, hermes-4-14b, MiniMax-M2.5, qwen3-next-80b, qwen3-coder-480b, and many more.

---

## SOP-003: XMTP Agent-to-Agent Communication

**Version:** 3.0
**Created:** 2026-03-14
**Updated:** 2026-03-14 (V3 Trust Framework upgrade)
**Purpose:** Manage XMTP-based agent-to-agent messaging — keypair management, client initialization, V3 message protocol, EIP-191 handshakes, and operational procedures.

### Overview

XMTP enables decentralized, encrypted messaging between AI agents using Ethereum wallet signatures for identity. No on-chain funds are required for messaging — only an Ethereum keypair for signing.

**Architecture (V3):**
```
┌────────────────────────────────────────────────────────────────────┐
│ OPENCLAW ECOSYSTEM                                                 │
│                                                                    │
│  Agent A ◄──► xmtp-comms-guard (middleware) ◄──► Agent B          │
│               │ Schema → Auth → Rate → PII → Injection → Trust │  │
└───────────────┼────────────────────────────────────────────────────┘
                │ XMTP (MLS E2EE)
                ▼
         External Agents
```

All agents **MUST** use `createGuardedXmtpClient` from the `xmtp-comms-guard` skill. Raw `@xmtp/client` calls are prohibited.

---

### XMTP Keypair Management

**Key Principle:** XMTP keypairs are messaging-only identities. They do NOT need funds. Keep them separate from wallets that hold tokens.

**Storage Locations:**
| Location | Content | Security |
|----------|---------|----------|
| `projects/xmtp-poc/identities/<agent>.json` | Private keys + addresses | Gitignored, local only |
| macOS Keychain (`lumerin-wallet`) | Router EOA key | System-level encryption |
| Memory files | Public addresses ONLY | Never store private keys |
| Bagman (future) | All keys encrypted at rest | 1Password / OS Keychain + EIP-7710 |

**Current XMTP Identities:**
| Agent | Address | Network | Created |
|-------|---------|---------|---------|
| morpheusai | `0x528DC1C6A154A88730cE6314C4AcB6f7628F3Cf4` | production | 2026-03-14 |
| everclaw | `0x60F627f6d332bC7dD48E359899999e66fBc5729a` | production | 2026-03-14 |

**Generating New Keypairs:**
```bash
cd projects/xmtp-poc && node keygen.mjs
```
New agents can be added by editing the `agents` array in `keygen.mjs`.

**NEVER:**
- Store private keys in memory files, MEMORY.md, or any committed file
- Reuse token-holding wallets (Router EOA, Safe) for XMTP messaging
- Share keypair files across machines without encrypted transport

---

### XMTP SDK v6 — Key API Differences

The SDK underwent a major rewrite from v0.x to v6.x. Key changes:

| v0.x API | v6.x API | Notes |
|----------|----------|-------|
| `Client.create(signer, { env })` | Same, but signer shape changed | `identifierKind` is now numeric (0=Ethereum) |
| `conversations.newConversation(address)` | `conversations.createDmWithIdentifier({identifier, identifierKind})` | Takes Identifier object |
| `conversation.send(text)` | `conversation.sendText(text)` | `send()` requires EncodedContent |
| `canMessage([address])` | `canMessage([{identifier, identifierKind}])` | Takes Identifier array |
| `client.close()` | (removed in v6) | No explicit cleanup needed |
| `message.sentAtNs` | `message.sentAtNs` (bigint) | Unchanged |

**Platform Fix (macOS arm64):**
The `@xmtp/node-bindings` native binary is compiled against Nix's `libiconv`. On macOS without Nix:
```bash
brew install libiconv

install_name_tool -change \
  /nix/store/7h6icyvqv6lqd0bcx41c8h3615rjcqb2-libiconv-109.100.2/lib/libiconv.2.dylib \
  /opt/homebrew/opt/libiconv/lib/libiconv.2.dylib \
  node_modules/@xmtp/node-bindings/dist/bindings_node.darwin-arm64.node

codesign --force --sign - node_modules/@xmtp/node-bindings/dist/bindings_node.darwin-arm64.node
```

---

### V3 Agent Message Protocol

All messages use the V3 JSON schema at `projects/xmtp-poc/agent-message-schema.json`.

**Protocol version:** `3.0`

**Required fields (every message):**
| Field | Purpose |
|-------|---------|
| `messageType` | HANDSHAKE, RESPONSE, COMMAND, DATA, BYE, INTRODUCTION |
| `version` | Must be `"3.0"` |
| `payload` | Message content (type-specific) |
| `topics` | Topic tags for trust boundary enforcement |
| `sensitivity` | public, guarded, technical, personal, financial |
| `intent` | query, update, introduce, handshake, revoke, status |
| `correlationId` | UUID linking responses to requests |
| `timestamp` | ISO 8601 |
| `nonce` | Base64-encoded 32-byte random (replay protection) |

**V3 Handshake Flow (EIP-191 replay-protected):**
1. Agent A builds HANDSHAKE with signed challenge (`conversationId` + `timestamp` + `nonce` + `version`)
2. Agent A sends structured JSON over XMTP
3. Agent B validates V3 schema, verifies EIP-191 signature against claimed wallet address
4. Agent B sends RESPONSE with counter-signature on the same challenge
5. Agent A verifies counter-signature → mutual authentication complete
6. For trust ≥6: ERC-8004 on-chain lookup (Phase 4)

**Check Pipeline (enforced by `xmtp-comms-guard` middleware):**
1. Schema validation (Zod) — kills 90% of injections
2. Peer authorization (Bagman DB + XMTP consent)
3. Rate limiting (10 msg/min per peer, 256 KB max)
4. PII Guard (existing skill)
5. Prompt Injection Guard (existing skill)
6. Trust & Context Boundary (rule-based + hybrid classifier)

---

### Procedure 1: Test V3 Roundtrip

**When:** After keypair changes, SDK updates, code changes, or network issues.

```bash
cd projects/xmtp-poc
rm -rf data/*.db3*              # Clean local DB state
node test-roundtrip.mjs         # Full V3 bidirectional test with EIP-191
```

**Expected output:** All checks pass with `✅`. Tests cover:
- Agent initialization
- Mutual reachability
- V3 schema validation
- EIP-191 challenge signing
- Counter-signature verification
- Correlation ID linking

---

### Procedure 2: Send a V3 Message

```bash
cd projects/xmtp-poc
node send.mjs --from morpheusai --to 0x60F627f6d332bC7dD48E359899999e66fBc5729a \
  --message "Hello" --topic everclaw --sensitivity technical
```

All messages are V3-structured, schema-validated, and EIP-191 signed before sending.

---

### Procedure 3: Listen for V3 Messages

```bash
cd projects/xmtp-poc
node receive.mjs --agent everclaw --timeout 60
```

The receiver validates all inbound messages against V3 schema. Invalid messages are logged with errors but not processed.

---

### Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Cannot find native binding` | Nix libiconv path mismatch | Run `install_name_tool` + `codesign` patch (see above) |
| `IdentifierKind` enum error | Passing string instead of number | Use `identifierKind: 0` (not `"Ethereum"`) |
| `Content type is required` | Using `send()` for text | Use `sendText()` for plain strings |
| `canMessage` returns false | Agent not yet registered on XMTP | Run `Client.create()` for that agent first (auto-registers) |
| DB lock errors | Stale db3 files | Delete `data/*.db3*` and retry |
| V3 validation fails | Missing fields in message | Check all 9 required fields (see protocol table above) |
| EIP-191 sig fails | Wrong canonical challenge format | Must be: `xmtp-comms-guard:handshake:v3.0\nconversation:...\ntimestamp:...\nnonce:...` |
| Challenge expired | >5 min between send and verify | Handshake TTL is 5 minutes; retry with fresh challenge |

---

### Security (V3 Trust Framework)

The `xmtp-comms-guard` skill at `skills/xmtp-comms-guard/` provides the full V3 security layer:

- **Default policy:** `reject-and-notify` — unknown peers are blocked
- **Storage:** Bagman-encrypted SQLCipher DB (peers, profiles, audit)
- **Consent sync:** XMTP native `setConsent()` / `publishConsent()`
- **Handshake:** Replay-protected EIP-191 with nonce + TTL + version binding
- **Trust tiers:** 10-point scale with profile-based topic/sensitivity enforcement
- **Threat model:** See `skills/xmtp-comms-guard/threat-model.md`
- **Full V3 design spec:** `memory/projects/everclaw/xmtp-trust-framework-v3.md`

---

### Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `@xmtp/node-sdk` | 6.0.0 | XMTP client |
| `viem` | 2.x | Ethereum wallet/signing + EIP-191 |

---

### Lessons Learned

1. **XMTP doesn't need funds** — Pure signing, no gas. Fresh keypairs work immediately.
2. **v6 is a full rewrite** — Most v0.x examples online are obsolete. Check types, not tutorials.
3. **Native bindings are Nix-compiled** — `install_name_tool` + `codesign` workaround required on macOS without Nix.
4. **Deterministic DB key required** — V3 derives encryption key from agent private key for persistent message history. Random keys lose history on restart.
5. **XMTP production network works** — No rate limits or fees encountered during PoC.
6. **`createDmWithIdentifier` vs `createDm`** — The former takes an Ethereum address (Identifier object), the latter takes an inbox ID string.
7. **V3 structured messages kill injections early** — Zod validation at pipeline entry rejects 90%+ of attack vectors before any LLM processing.
8. **EIP-191 canonical format matters** — Challenge must be deterministic string (newline-separated key:value pairs). Any deviation breaks signature verification.