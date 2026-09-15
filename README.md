# PowaAgent Skill

[![skills.sh](https://skills.sh/b/powaagent/skills)](https://skills.sh/powaagent/skills)

The PowaAgent skill gives AI agents access to live paid reference data and autonomous x402 payments — without the agent needing to hold a wallet or private keys. PowaAgent signs each payment from your own PowaAgent account wallet and applies your spend controls — and **takes no fee on your x402 payments; you pay only the seller's price.**

---

## What It Does

**Data lookups** — the skill matches your query to the live PowaHub paid service catalog and handles the rest. Supported data includes FX rates, BIN/card metadata, country data, public holidays, IP geolocation, and more — with new services added to the catalog without requiring a skill update.

**x402 payments** — when your agent needs to pay any x402 endpoint (inside or outside the PowaHub catalog), PowaAgent signs the payment from your PowaAgent account wallet so the skill can complete the call. You never need a wallet, a private key, or to sign or fund the payment yourself — the skill handles the 402 exchange and payment header for you.

**Two ways to run it** — if your agent platform is connected to the PowaAgent MCP server, the skill detects that and routes both capabilities through the connector's tools, with nothing to set up in the skill itself. The connector is authenticated by your agent platform, using whatever credentials that connection was set up with — the skill never sees or handles them. With no connector, the skill calls the PowaAgent API directly over HTTP using your configured API key and fetches the public PowaHub catalog unauthenticated, exactly as before.

---

## Installation & Quick Start

> Two modes: `powaagent_flow` (PowaAgent pays for you) and `catalog_only` (you pay). Pick one at setup — see [Choosing a Mode](#choosing-a-mode) below for the full comparison. **With a PowaAgent MCP connector connected there is nothing to set up in the skill** — it uses the connector and skips the interview. You still need a PowaAgent account with a funded wallet, and the connector itself must be authorized in your agent platform.

### Requirements

- **An agent platform** that loads skills (Claude Code, or any platform that reads a `SKILL.md`).
- **With a PowaAgent MCP connector** — connect the PowaAgent MCP server (`https://mcp.powaagent.ai/mcp`) in your agent platform and the skill detects it automatically, with no `skill-config.json` and no API key for the skill to store. **The connector itself is authenticated.** How you authorize it — OAuth when you connect through the web app, or a token for orchestrators that can't do OAuth — is configured on that connection in your agent platform, not in the skill. You still need USDC in your account wallet on Base.
- **For `powaagent_flow`** — a PowaAgent account and API key ([portal.powaagent.ai](https://portal.powaagent.ai)), and USDC in your account wallet on Base. No ETH needed; gas is sponsored.
- **For `catalog_only`** — nothing. No account or key required.

### Quick Start (Human Users)

> **Note for AI assistants:** if you're an agent setting this skill up on a user's behalf, skip to [Quick Start (AI Agent)](#quick-start-ai-agent) below.

#### Install

**Option 1 — skills CLI (recommended)**

```shell
npx skills add powaagent/skills --skill powaagent
```

Installs the skill into every detected agent platform on your machine at once.

**Option 2 — Manual**

Download `SKILL.md` and place it in your agent platform's skills directory (see your platform's docs for the correct path):

- [github.com/powaagent/skills](https://github.com/powaagent/skills) (in the `skills/powaagent/` folder)
- [certaindata.ai/for-agents](https://www.certaindata.ai/for-agents)

#### Configure & Use

1. **Trigger setup.** On first invocation the skill runs a short setup interview automatically; or ask your agent at any time — `Set up PowaAgent`. It writes a local `skill-config.json`.
2. **Add your API key** (`powaagent_flow` only). Setup asks for the env var name (default `POWAAGENT_API_KEY`) and env file (default `~/.env`). Add the key there, then restart your agent gateway:
   ```
   POWAAGENT_API_KEY=your-api-key-here
   ```
3. **Run a lookup.**
   ```
   Look up BIN 424242
   ```
   Once configured, the skill matches the query to a catalog service — in `powaagent_flow` it completes the paid call and returns the result with on-chain settlement details; in `catalog_only` it hands back the request blueprint and payment terms for you to pay yourself.

To change your mode or any setting later, ask your agent to `Reconfigure PowaAgent` — it keeps your existing settings and lets you change only the one(s) you want, or optionally change multiple.

### Quick Start (AI Agent)

> The following steps are for AI agents configuring the skill on a user's behalf. Some steps require the **user** to act in a browser (account sign-up, retrieving an API key). Never print, echo, or log the key value.

**Step 1 — Install.** Run the skills CLI, or confirm `SKILL.md` is already in the platform's skills directory:

```shell
npx skills add powaagent/skills --skill powaagent
```

**If a PowaAgent MCP connector is already connected, skip Steps 2 and 3.** The skill detects it, writes no `skill-config.json`, and stores no API key of its own — go straight to Step 4. The connector's own authentication must already be configured on that connection in the agent platform; the skill neither supplies nor checks it. Run setup only if the user specifically wants `catalog_only`, which deliberately ignores the connector so they can pay from their own wallet.

**Step 2 — Run setup and choose a mode.** Run `Set up PowaAgent` to start the setup interview (it also runs automatically on first invocation when no `skill-config.json` exists and no connector is present). Pick a mode with the user:

- `powaagent_flow` — PowaAgent signs payments from the user's account wallet (needs an account + API key).
- `catalog_only` — catalog matching and request blueprints only (no account).

**Step 3 — Configure credentials** (`powaagent_flow`). Have the user sign up at [portal.powaagent.ai](https://portal.powaagent.ai) and add their API key to the configured env file (default `POWAAGENT_API_KEY` in `~/.env`), then restart the agent gateway.

**Step 4 — Verify.** Run a lookup and confirm the result (and, in `powaagent_flow`, the settlement transaction):

```
Look up BIN 424242
```

### Updating

Run `npx skills update` to upgrade to the latest version. The CLI wipes and recreates the skill directory on update, so your local `skill-config.json` (mode, API-key reference, environment preference) — which lives in that directory — is cleared. The skill re-runs its First-Run setup interview on the next invocation. Your API key is unaffected as long as its env file sits outside the skill directory — which the default (`~/.env`) does, so it survives the update and never needs re-entering. Keep the env file outside the skill directory: if you point it inside, it is wiped along with everything else on update.

### Mode configuration reference

**`powaagent_flow`** — setup asks for:

1. The env var name to store your API key under (default: `POWAAGENT_API_KEY`)
2. The env file path to read it from (default: `~/.env`)
3. Your environment preference — whether to offer sandbox when a data service supports it, or always use production (default: always production)

Your spend controls (per-call, daily, and monthly caps), trust-tier preferences, and endpoint allow/deny lists live on your PowaAgent account and are managed in the [portal](https://portal.powaagent.ai) — not in the skill. They are enforced automatically each time PowaAgent signs a payment.

**`catalog_only`** — no account or API key required. Setup asks only for your environment preference:

- **Ask each time** — when a service has a test endpoint available, the skill offers you the choice of sandbox or production before each call
- **Always production** — skip the prompt and always use the live endpoint (recommended for production or automated environments)

**With an MCP connector** — nothing to configure in the skill. There is no `skill-config.json` and no env var for the skill to read; the connector's authentication is configured on that connection in your agent platform instead. Your spend controls, trust-tier preferences, and allow/deny lists still apply in full: they live on your PowaAgent account and are enforced whenever the connector signs a payment. `catalog_only` is the one exception — if you have deliberately configured it, the skill ignores the connector, because that mode exists so you pay from your own wallet.

---

## Choosing a Mode

The skill runs in one of two modes, chosen once at first setup — or through a connected MCP connector, which needs no mode chosen at all:

| Mode | PowaAgent account | What the skill does |
|---|---|---|
| **MCP connector** | Required (funded wallet; the connector is authenticated by your agent platform) | Detected automatically, with no mode to choose and nothing configured in the skill. Routes both catalog lookups and external x402 endpoints through the connector's tools, which pay from your PowaAgent account wallet under your spend controls. Sandbox is available for catalog services but only when you ask for it explicitly; endpoints outside the catalog are always production |
| `powaagent_flow` | Required | For both catalog data lookups and external x402 endpoints, PowaAgent signs the payment from your PowaAgent account wallet (applying your spend controls) and the skill completes the call and returns the data. No payment plumbing on your side, and PowaAgent takes no fee on the payment. Sandbox (test USDC on Base Sepolia) is available for catalog services that support it — PowaAgent funds your test wallet and signs the test payment for you |
| `catalog_only` | Not required | Matches your query to the catalog and returns the request blueprint and payment terms for you to pay yourself |

**Use `powaagent_flow`** if you want PowaAgent to handle payment for you — for both catalog data lookups and any x402 endpoint — signing from your PowaAgent account wallet with your spend caps and allow/deny lists enforced automatically.

**Use `catalog_only`** if you already have your own x402 wallet or tooling and just want the catalog matching and request details.

**With a PowaAgent MCP connector connected you don't need to choose.** The skill uses the connector and handles payment for you, the same as `powaagent_flow`, with nothing to configure in the skill — no mode to pick and no API key for it to store, since the connector is authenticated by your agent platform. A PowaAgent account with a funded wallet is still required: that is where the payments come from. Configure `catalog_only` only if you want the connector ignored so you can pay yourself.

---

## How Payments Work

In `powaagent_flow` mode you never handle a wallet, a private key, or a payment header. When your agent reaches a paid endpoint:

1. It calls the endpoint and receives a `402 Payment Required`.
2. PowaAgent signs the payment from your PowaAgent account wallet — applying your spend caps and allow/deny lists — and returns a payment header.
3. Your agent retries the call with that header; the payment settles on-chain and the data comes back with a settlement transaction.

The payment goes straight from your wallet to the seller — PowaAgent signs it but never holds your funds.

You only ever need USDC in your wallet — never ETH. Network (gas) fees are sponsored, so there is nothing else to top up.

**With an MCP connector** the same thing happens, except the connector handles the 402 exchange and the payment for you and hands back the data with a payment receipt. Your spend caps, trust-tier preferences, and allow/deny lists are applied exactly as above. For endpoints outside the catalog the skill first asks the connector for the seller's terms — a free check that pays nothing — so it can quote you the real price before you approve it. That is the price as quoted at that moment; the payment re-checks when it settles, so it can differ if the seller changes it, and your spend caps remain the ceiling.

In `catalog_only` mode PowaAgent signs nothing. Your agent pays with its own wallet or tooling, using the request details and payment terms the skill hands back.

---

## Trust Tiers

Every catalog service carries a trust tier, returned with the catalog so you and your agent can see what a service is before paying:

- **Premium** — partner services PowaHub has vetted for data quality, uptime, freshness, and licence validity.
- **Standard** — self-onboarded catalog services. PowaHub does not vouch for their quality, uptime, or licensing.
- **Open Ecosystem** — endpoints outside the catalog altogether (the public x402 Bazaar, or a URL you supply). Not vetted; PowaAgent only signs the payment.

In `powaagent_flow` the tiers also act as a spending preference: PowaAgent signs a payment only for tiers you've enabled on your account — Premium is always on; Standard and Open Ecosystem are optional toggles in the [portal](https://portal.powaagent.ai). The tiers stack (each includes the one above it), and a payment is never signed if the endpoint is on your deny list or would breach a spend cap.

---

## Sandbox Mode

Sandbox uses test USDC on Base Sepolia — no real money is spent. It is offered **only for catalog services that support it** (the catalog flags each one); Open Ecosystem endpoints and any URL you supply always use production. When you choose sandbox, the same service URL is used — the skill just routes the call to the test environment. Selecting it requires your confirmation each time.

How it works depends on your mode:

- **`powaagent_flow`** — PowaAgent funds your Base Sepolia test wallet automatically and signs the test payment for you, just like production. You do nothing extra.
- **`catalog_only`** — the skill hands you the test (Base Sepolia) request and payment terms, and you fund and pay your own test wallet.
- **With an MCP connector** — sandbox works as it does in `powaagent_flow`: your Base Sepolia test wallet is funded and the test payment made for you, with nothing extra to do. The one difference is that it is never chosen for you — ask for it explicitly ("use sandbox") and the skill passes that through. Endpoints outside the catalog are always production on the connector; there is no sandbox for them.

---

## Examples

### Data lookups

```
What is the EUR to USD exchange rate?
```
```
Look up BIN 424242
```
```
What are the public holidays in South Africa this year?
```
```
What is the current BTC price in USD?
```

The skill checks the live catalog first. If a matching service exists, it handles the lookup and returns the result with the on-chain settlement details. If no service matches, it searches the public x402 Bazaar before falling back to web search.

With an MCP connector connected, the skill matches against the connector's service tools instead — same result, no catalog fetch — and confirms the price with you before making the paid call.

### x402 payments

```
Pay this x402 endpoint: https://api.example.com/data
```

In `powaagent_flow` mode the skill calls the endpoint, has PowaAgent sign the payment from your PowaAgent account wallet, and completes the call — returning the data and the settlement transaction. Paying endpoints outside the PowaHub catalog requires the **Open Ecosystem** preference to be enabled on your account (managed in the portal). In `catalog_only` mode the skill returns the request blueprint and payment terms for you to pay yourself.

With an MCP connector, the skill first asks the connector for the endpoint's payment terms — free, and nothing is paid — then confirms the actual amount with you before making the payment. Your spend caps remain the ceiling.

---

## What This Skill Is Not

- **Not an agent-controlled wallet** — payments are signed from your own PowaAgent account wallet, and PowaAgent applies your spend controls. The agent never holds your private keys.
- **Not a substitute for web search** — it checks paid data first and falls back to the web only when nothing matches.
- **Not authoritative or a live feed** — results are indicative reference data, not a system of record or a real-time stream. FX rates are reference rates (not trading signals); BIN data is indicative metadata (not bank-of-record verification); and every call is a discrete request/response, with no subscription or push model.
- **Not a blanket data-quality guarantee** — PowaHub vets **Premium** catalog services for data quality, uptime, and freshness, but does not vouch for **Standard** or open-ecosystem endpoints. For those, you get whatever the seller returns after payment settles, which may be empty or wrong.
- **Not infallible in sandbox** — some sandbox services accept only curated test inputs, so an out-of-set input can return a useless result even though the test payment settled.

---

## Security

The skill is **non-custodial and instruction-only**: it never signs payments and never holds your funds. **Your PowaAgent wallet is non-custodial — you keep custody of the private key, and neither the skill nor PowaAgent's backend can access it.** PowaAgent is granted only a scoped signing delegation: it can authorize a payment within the spend caps and allow/deny lists you've defined on your PowaAgent account, and the signature is produced without your key ever being shared with PowaAgent.

Automated skill scanners rate the skill as elevated risk — for the deliberate capabilities it needs (reading an API credential, calling external endpoints, initiating payments), not a vulnerability, undocumented endpoint, or malicious code.

See **[SECURITY.md](SECURITY.md)** for the full security model — credential handling, the trust boundary, payment controls, the "what this skill does not do" list, and how to report a vulnerability.

---

## Troubleshooting

**The skill doesn't run / nothing happens.** It may not be loaded. Re-install (`npx skills add powaagent/skills --skill powaagent`) or reload your agent, confirm `SKILL.md` is in your platform's skills directory, then try a quick lookup again.

**It can't find your API key, or the key was rejected (`powaagent_flow`).** Check that your env file contains the key line (e.g. `POWAAGENT_API_KEY=your-actual-powaagent-api-key-here`) under the variable name and path you set during setup, then restart your agent gateway. Manage or rotate the key in the [portal](https://portal.powaagent.ai).

**A payment was refused.** PowaAgent applies your spend caps, trust-tier preferences, and allow/deny lists at sign time. When a payment is blocked the skill tells you why — adjust the relevant setting in the [portal](https://portal.powaagent.ai) and retry.

**The skill isn't using my MCP connector.** The skill only uses a connector it can positively identify as PowaAgent's, and it never reads your MCP client configuration to do so. Confirm the PowaAgent MCP server is connected and its tools are visible to your agent, then start a new session so detection runs again. If you configured `catalog_only`, the connector is ignored by design — run `Reconfigure PowaAgent` to change mode.

**Still stuck?** Reach support (below).

---

## Support

- **Portal & account management:** [portal.powaagent.ai](https://portal.powaagent.ai)
- **Contact:** [certaindata.ai/contact](https://www.certaindata.ai/contact)
- **Email:** support@certaindata.ai

---

## License

Licensed under the Apache License 2.0 — see [LICENSE](LICENSE) and [NOTICE](NOTICE). Use of the PowaAgent and PowaHub services accessed by this skill — both provided by CertainData Ltd — is governed by the [PowaAgent Terms](https://www.powaagent.ai/terms).
