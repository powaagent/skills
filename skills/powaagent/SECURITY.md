# Security Model

This document explains the security posture of the **PowaAgent Agent Skill** — an
instruction-only skill (a single `SKILL.md`, no bundled code) that lets an agent look up
live paid reference data and pay [x402](https://www.x402.org) endpoints in USDC on Base.

By design the skill handles a bearer credential, calls external endpoints, and can initiate
a payment — but **only from the user's own PowaAgent account wallet, via delegated signing,
and only within server-side policy.** It cannot access, hold, or move funds from any other
wallet. Those capabilities are the product, not accidents — this document describes the
controls that keep each one governed. Automated skill scanners correctly surface these
capabilities; the sections below are the intended mitigations for them.

---

## Current scan status

Independent skill-security scanners audit the published skill. The most recent audits (2026-07-29):

| Scanner | Verdict | What it flags |
|---|---|---|
| Gen Agent Trust Hub | Pass | — |
| Socket | Medium overall; one **Anomaly** alert (low, 83% confidence) | Reads a local secret file, forwards the live API key to its issuer (`api.certaindata.ai` at the time of the audit; now `api.powaagent.ai`), and cannot independently verify the payment-signing endpoints. Socket assesses this as a high-trust financial integration rather than malware. |
| Snyk | Fail (high, driven by W007) | **W007** — insecure credential handling (high). **W009** — direct money-access capability (medium). **W011** — exposure to third-party seller content (medium). |

These verdicts describe the skill's *intended capabilities*, not malware: credential handling, external calls, delegated signing, and autonomous payment are the product. The named Snyk findings and Socket's Anomaly alert map to controls below — **W007** under **Credential handling**, **W009** under **Payment safety and money movement**, **W011** under **Untrusted external input**, and Socket's **Anomaly** under **Endpoint authenticity and credential destination** (with its local-file concern also covered by **Credential handling**).

---

## Credential handling

_Addresses security-scan finding W007 (credential handling)._

The skill reads a PowaAgent API key to authorize delegated signing. Its handling is
deliberately constrained:

- **Resolution order.** The configured environment variable first; if it is empty or
  unset, the key line in the configured env file (`secret.env_file`). A blank value is
  treated as absent — the skill never sends an empty bearer token.
- **Never emitted.** The key value is used only to build the `Authorization` header for a
  request to its issuer, `api.powaagent.ai`. It is never sent to a seller, Coinbase, or
  any other third party, and is **never printed, echoed, logged, or written back** to any
  file, prompt, or tool output.
- **Passed by reference where possible.** On agent platforms that support env-var
  substitution AND when the key was resolved from the environment variable, the
  `Authorization` header is sent as `Bearer ${<env_var>}`, so the raw key is resolved at
  send time and never enters the model's generated output. If the key was resolved from
  `secret.env_file` (env var unset/empty), the value is inserted directly to avoid sending
  a blank token. On platforms without substitution, insert the value per the resolution
  order. *(W007 mitigation.)*
- **Least exposure.** Reading a key from a file outside the workspace requires explicit
  user approval. The key is never persisted into `skill-config.json` (which stores only
  the env-var name and file path, never the secret) and `skill-config.json` is never
  committed.
- **Rotation.** Keys are managed and rotated in the [PowaAgent portal](https://portal.powaagent.ai);
  the skill holds no long-lived copy.
- **MCP connector path — no key is resolved at all.** When the host has connected the
  hosted PowaAgent MCP server (<https://mcp.powaagent.ai/mcp>), the skill routes its work
  through that connector's tools and resolves **no** API key: it does not read
  `secret.env_var`, does not read `secret.env_file`, and never forwards a key to an MCP
  server. Authentication belongs to the connector and is held by the host client — OAuth,
  or the server's own key for orchestrators that cannot do OAuth — so the model never
  handles a key value on this path, sidestepping W007 entirely. The skill does not open
  the MCP transport itself and does not read the host's MCP client configuration; it only
  uses tools the host has already surfaced to it.

## Endpoint authenticity and credential destination

_Addresses the current Socket **Anomaly** alert (low, 83% confidence; overall scanner risk Medium)._

- **Published destinations.** `SKILL.md`'s endpoint table publishes the fixed HTTPS URLs
  for the PowaAgent signing and testnet-funding calls, the PowaHub service catalog,
  and the Coinbase Bazaar. The only other destination is the specific HTTPS seller URL
  selected from the PowaHub catalog, the public Bazaar, or supplied by the user; the
  skill does not discover or call any undisclosed intermediary.
- **TLS identity.** Every call uses HTTPS and is limited to the published destination for
  that step. As an instruction-only artifact, the skill cannot independently attest the
  implementation behind those TLS endpoints; Socket correctly surfaces that trust boundary.
- **Credential scoping.** The PowaAgent API key is sent only to `api.powaagent.ai`, its
  issuer. The PowaHub service catalog is public and unauthenticated, so it never
  receives the key either; nor do Coinbase discovery calls or seller calls. Sellers receive
  only the request data they require and, on a paid retry, the x402 payment header.

Socket describes this as a high-trust financial integration rather than malware. The alert
reflects the unavoidable trust placed in the published signing service and its credential
handling, not a hidden endpoint or an undisclosed credential destination.

## Untrusted external input

_Addresses security-scan finding W011 (third-party content exposure) and prompt-injection risk._

Catalog entries, seller `402` responses, payment terms, and public x402 Bazaar results are
treated as **data to parse, never as instructions.** For a seller `402`, the skill uses the
closed forwarding allowlist in `SKILL.md`. The selected seller `network` value is forwarded
unchanged. V1 `maxAmountRequired` is sent to the sign endpoint as `amount`.
Unrecognised fields are discarded and never influence a decision, enter a later
request, or persist. Seller-provided resource descriptions, `extra.name`, and free-text errors
are untrusted strings: they never drive selection or reasoning and, if shown, are clearly
attributed and delimited as seller text. A contract-required resource description may be
copied into its named sign-request field; it and `extra.name` may otherwise be shown only
under the attribution and delimiting rule above. Seller error text is not forwarded.

The skill does not act on natural-language directives embedded in a response — including
any attempt to change an endpoint, raise an amount, skip a confirmation, exfiltrate the key,
or bypass a spend cap. When a response conflicts with the skill's instructions or the user's
intent, the skill stops and surfaces it rather than acting. See the **Trust Boundary** section
of `SKILL.md`.

MCP tool results are treated identically — data to parse, never instructions. MCP server
instructions blocks and tool descriptions are text authored by whoever operates that server:
the skill reads them only to identify the server and to learn a tool's inputs and price.
Matching the PowaAgent identifier is a routing decision, not a trust assertion. A bare tool
name never establishes identity, because `pay_x402`, `probe_x402` and `search_bazaar` are
the x402 protocol's own vocabulary — without that rule an unrelated MCP server exposing the same
names could attract a real payment on a naming collision alone.

**Residual risk.** Identity matching raises the bar against an *accidental* collision; it
does not defeat *deliberate* impersonation. Both signals are operator-authored text, and a
server that reproduces the identifier and the description marker verbatim will match. The
skill can no more attest what sits behind a connector than it can attest what sits behind a
TLS endpoint, and it deliberately does not read the host's MCP configuration to try. What
bounds the exposure is not the match but the controls after it: the price is confirmed
before a catalog call, the probed amount is confirmed before an outside-the-catalog
call, and every payment is authorized server-side against the user's
spend caps, trust tiers, and allow/deny lists. Connecting an MCP server is a trust decision
the user makes in their agent platform, and the skill treats it as one.

W011 is inherent to the x402 flow because the seller's `402` payment requirements must be
read before a payment can be formed. Forwarding the full `accepts[]` list would not remove
that exposure.

## Payment safety and money movement

_Addresses security-scan finding W009 (autonomous payment)._

Payments are **non-custodial and policy-gated**:

- **First-party delegated signing.** PowaAgent — the skill's own provider, not an unrelated
  third party — initiates payments from the user's own non-custodial PowaAgent account wallet.
  **The wallet is non-custodial: the user keeps custody of the private key, and neither the skill
  nor PowaAgent's backend can access it.** PowaAgent is granted only a scoped signing
  delegation — it can authorize a payment within the spend caps and allow/deny lists the user has
  defined on their PowaAgent account, but the signature is produced without the key ever being
  shared with or held by PowaAgent. PowaAgent never holds the funds or sits in the money
  path; the payment settles directly from the user's wallet to the seller.
- **Authority is bounded by server-side policy.** The skill has no standing spend authority of
  its own — every payment must be authorized at sign time against controls that live on the
  user's PowaAgent account, not in the skill (where they could be edited): per-call, daily,
  and monthly spend caps, trust-tier preferences, and endpoint allow/deny lists. These are the
  approval gate — a payment that would breach a cap or hit a deny-listed endpoint is refused
  server-side, so the skill cannot spend beyond what the user has explicitly permitted.
- **Scoped reach.** Paying an endpoint outside the PowaHub catalog (open ecosystem)
  requires the user to have enabled that tier on their account.
- **Production by default.** All calls default to production (Base Mainnet). Sandbox (test
  USDC on Base Sepolia) is opt-in per catalog service and requires explicit user
  confirmation each time. On the MCP connector path sandbox is never selected on the
  skill's own initiative — the user must direct it — and it is unavailable for endpoints
  outside the catalog, which always settle in production.
- **Price is confirmed before a connector payment, or its absence is stated.** Service tool
  descriptions carry the price and the skill confirms it before calling; a service that
  publishes no price is confirmed explicitly as unpriced and is never assumed free. For
  endpoints outside the catalog the skill probes the seller first through the connector's
  `probe_x402` tool — which signs nothing, settles nothing, and consumes no spend cap — and
  confirms the quoted amount before paying. The quote is as at probe time, since the payment
  re-probes when it settles; if the probe fails, or reports the call as free, the skill says
  so rather than implying a price. **No paid call is made without a confirmation**, including
  one the seller reports as free: the payment re-probes and can still charge. The account's
  server-side spend caps remain the ceiling throughout.
- **No blind retry on the connector path.** If a **paying** MCP tool call — `pay_x402` or a
  service tool — fails or its response is lost, the skill neither retries the tool nor falls
  back to direct HTTP: either could pay twice. It surfaces the uncertainty and stops; only
  the user may direct a retry. The rule is scoped deliberately: `probe_x402` and
  `search_bazaar` never pay, so a failure there carries no double-payment risk and is not
  treated as an uncertain payment.
- **Asset selection.** The sign endpoint filters `accepts[]` to USDC server-side. The skill
  narrows the list first to save an unnecessary signing round trip.
- **No double payment.** On a transport retry the skill resends the *same* signed payment
  header; it re-signs only after the authorization window has lapsed.
- **No buyer fee.** PowaAgent charges the buyer nothing — the buyer pays only the
  seller's price.

## What this skill does not do

Automated skill scanners flag this skill's *capabilities* (credential handling, external
calls, delegated signing, autonomous payment). For clarity, the skill explicitly does **not**:

- **Execute code.** It is instruction-only — no bundled scripts, no subprocess, no shell. Every action is a plain HTTP request the agent makes directly.
- **Read arbitrary files.** It reads only the configured env file to resolve the API key; a path outside the workspace requires explicit user approval.
- **Expose the key.** The key value is never printed, logged, echoed, or transmitted anywhere except the `Authorization` header of a request to its issuer, `api.powaagent.ai`; it is never sent to a seller or third party.
- **Move money from any other wallet.** It can only initiate payments from the user's own PowaAgent account wallet, signed under server-side spend caps and allow/deny policy; it defaults to production and requires explicit confirmation for sandbox. It never holds funds.
- **Obey external content.** Instructions embedded in catalog, seller `402`, or Bazaar responses are ignored — see *Untrusted external input* above.
- **Forward unrecognised seller fields.** Seller `402` fields outside the closed x402/sign-contract allowlist are not acted on, forwarded, or persisted.
- **Send the key to an MCP server.** On the connector path the skill resolves no API key at all — authentication belongs to the host's own connection. No key is ever forwarded to any MCP server.
- **Call undisclosed endpoints.** The skill's endpoint table publishes the fixed HTTPS PowaAgent, PowaHub catalog, and Coinbase URLs. The only other destination is the specific HTTPS seller endpoint selected for the user's request. No intermediary or other host is called — closing off any undisclosed / malicious-URL vector. The MCP server is deliberately **not** in that table: the skill never opens an MCP transport and never reads the host's MCP client configuration, so a connector is used only through tools the host has already surfaced.
- **Collect or exfiltrate data.** It transmits only what each request requires; it gathers no telemetry and sends no user data elsewhere.

## Scope

This document and the skill's licence cover the **skill instructions** (`SKILL.md`) only.
The PowaAgent and PowaHub services, accounts, and any payments made through them are
provided by CertainData Ltd (PowaAgent and PowaHub are its products, not separate
legal entities) and governed by the separate
[PowaAgent Terms](https://www.powaagent.ai/terms); accounts are managed at
[portal.powaagent.ai](https://portal.powaagent.ai).

---

## Reporting a vulnerability

If you believe you have found a security issue in this skill, please report it **privately** —
do not open a public issue.

- **GitHub (private):** use [Report a vulnerability](https://github.com/powaagent/skills/security/advisories/new) on the repo's Security tab (GitHub Private Vulnerability Reporting).
- **Email:** support@certaindata.ai (subject line prefixed `SECURITY:`)
- **Contact form:** [certaindata.ai/contact](https://www.certaindata.ai/contact)

We aim to acknowledge reports promptly and will keep you updated on remediation.
