# HobbyCloud — Product Requirements Document

**Status:** Draft v2 · **Date:** 2026-08-09
*(v1 2026-07-26. Changelog at §11.)*

**One-liner:** Push a directory, get a private URL your team opens with their work login, and let that app spend and query against your real systems through connections the builder never holds a credential for.

**Thesis:** Agents made small software trivial to build and left it hard to deploy safely. The deploy part is commoditizing — Vercel, Val Town, and Naïve all ship it. The part that isn't: giving an app scoped access to a company's real systems without handing a human the keys. We ship the frictionless surface to get adoption and build the brokered-credential layer underneath to keep it.

**Non-negotiable architectural rule:** *No credential to a system HobbyCloud did not issue ever lands in an app's runtime environment.* Every decision below follows from this. Breaking it once, even in v0, forfeits the moat.

- **Covered (never injected):** LLM/provider API keys, database credentials, OAuth tokens, CRM and internal-API credentials. The app receives a HobbyCloud-issued scoped token or handle; we hold the real material and swap it in at the boundary.
- **Not covered (may be env vars):** the Builder's own non-credential config and secrets to systems we broker nothing for (e.g. a personal webhook URL). These carry no scoping story and claim none.
- **Why the wording changed:** v1 said "no customer credential" in one place and "no credential, ever" in another while Stage 0 injected API keys as env vars. Those could not all hold. A rule broken once stops constraining anything, so it is now scoped to exactly what we can enforce from week one — and §4 Stage 0 now enforces it for provider keys, which v1 did not.

---

## 1. Problem

### 1.1 Who
| Persona | Description | Cares about |
|---|---|---|
| **Builder** (primary) | Semi-technical: ops, analytics, PM, junior eng. Ships agent-built tools. Cannot write a Dockerfile and shouldn't have to. | Time from "it works locally" to "my teammate is using it" |
| **Lead** (economic buyer) | Eng manager, head of ops, platform lead. Owns the mess. | Knowing what exists, killing it fast, not being the reason for a breach |
| **Admin** (gatekeeper) | IT / security / data owner. Can veto. | No standing credentials, revocability, an audit trail |

### 1.2 Evidenced problems

Evidence from Hacker News (fetched, verified). Reddit and LinkedIn were unreachable from this environment — the Builder persona is therefore **under-validated** and is the top research gap.

| ID | Problem | Evidence | Strength |
|---|---|---|---|
| P1 | Deploying a private app for a team is disproportionately hard | "Launch HN: Onu — turn scripts into internal tools" 160pts/95c; "Ask HN: Where do you deploy internal Streamlit apps?" asks specifically about "Auth / network model"; one builder: the choice was "between fully open or days spent on OAuth" | **Strong** — demand-side |
| P2 | Metered API keys in shared apps produce surprise bills | "Lost $300 due to an API key leak from vibe coding" 8pts/**14 comments**; HN comment: "Even an app with 0 AI features exposes dollars-per-query video models unless the key is manually scoped"; docs push users to ship "an open proxy that allows AI billing abuse" | **Strong** — dollar-denominated |
| P3 | Nobody knows if the deployed thing is safe | "Hacker Lists Vibecoded Apps: 198 Scanned, 196 Found Vulnerable"; Veracode cited at 45% of AI code samples vulnerable; Lovable left projects exposed 48 days | **Real but unmonetizable** — see 1.3 |
| P4 | Tools are useless without company data; wiring it up means a human holds a credential | Inferred from YC RFS ("every company will want to customize the environment this software runs in") + **verified absence**: neither Vercel's marketplace nor any of Naïve's 50 primitives broker scoped access to a database the customer already runs (§2.3, §2.4) | **Unvalidated demand, verified whitespace — must test (§8.2)** |
| P5 | Leads have no inventory of what their team deployed or what it can reach | Inferred | Weak evidence, strong intuition |

### 1.3 The trap to avoid

~15 vibe-coding security products on HN, all at 1–6 points: Vibe Security, SafeVibe, Isitsecure, SecureNow, Vibe Coding Security Hook, Vibe Security Radar, "Provably secure vibe coding". Incident posts get engagement; prevention products get none. One commenter: repeated breaches likely won't shift perception. Another: a tampered "event calendar for bar trivia" harms little.

**Conclusion: security is not the wedge.** It is the feature that closes the Admin, sold on the back of P1 (works) and P2 (saves money). Never lead with fear.

### 1.4 Source of truth — YC RFS, "A Cloud for Small Software" (Pete Koomen), verified quotes

- Small Software = "purpose-built tools that will only ever have one or a small handful of users"
- Building is easy; such software is "still hard to deploy and share"
- AWS/Azure were built for "shipping Big Software that scales with many users… at the cost of complexity"
- A purpose-built cloud "could delete most of this complexity"
- Benchmark: "Small software should be as easy to share with your colleagues as a Google Doc"
- Three hard problems: (1) "every company will want to customize the environment this software runs in" (2) "auth & permissions are hard" (3) "allowing nontechnical users to share arbitrary code is tricky to do securely"

**Honest note:** the capability-brokering architecture in this PRD is *our* thesis, not Koomen's text. His "environment customization" may well mean runtime environments (deps, egress, limits), not data access. Do not assert our reading as his in an interview. Also note the RFS emphasis is *frictionlessness* — an admin-configured capability graph is in tension with "as easy as a Google Doc", which is why the surface must stay trivial (§3).

---

## 2. Competitive position

Verified from vendor docs, July–August 2026.

| Player | What they have | The gap we exploit |
|---|---|---|
| **Vercel** | **Closest competitor to our MVP.** AI Gateway: one key → hundreds of models, BYOK, zero token markup, per-app attribution, spend monitoring, provider fallback. Deployment Protection at every tier. Marketplace provisions Neon/Supabase Postgres | Viewer identity is the gap. Vercel Authentication admits only **Vercel users with team access rights** — 20 non-technical colleagues means 20 accounts in your team. Own-IdP login is **Passport: Enterprise, contact-sales**. Password protection is Enterprise or a **$150/mo Pro add-on**. AI Gateway usage docs show balance/generations/reporting — **monitoring, no per-project hard cap that rejects**. Serverless + Next.js-shaped; not a home for long-running Streamlit |
| **Naïve** (usenaive.ai) | 50 primitives across identity/legal, money, cloud, content, market data, trust. Governance spine is real: approvals enforced at the primitive boundary (`payment.charge Revoked`), vault with runtime-only retrieval, **revocable MCP sessions where "identity and policy travel with every call"** | **Six primitives already frozen** — the entire orchestration layer (CEO/Staff/Objectives/Tasks/Cron) deprecated in favour of `team({lead})` on `runtime.durable()`; cron is "declared, and refused today". 8 market-data primitives are visible resale. Every credential in their system is one **they** provisioned or captured — **zero coverage of scoping a database the customer already runs**. Sells to agents that need to be born; we sell to companies that already exist |
| **Val Town** | Literally "The deployment platform for internal tools". 100ms deploys, crons, SQLite, S3, CLI, agent plugins for Claude/Codex/Cursor + MCP server. Logos: OpenAI, Anthropic, Shopify, Google | Landing page says **nothing** about auth/permissions on shared code, secrets, or team RBAC. Ahead on DX, absent on governance. (§8.3 gap: read their docs, not their landing page) |
| **OpenRun** (OSS, ClaceIO) | Everything we'd build: "OAuth/OpenID/SAML/Client-cert based auth, with full RBAC", audit logging, scale-to-zero, Streamlit/Gradio/FastHTML, service bindings that provision Postgres/MySQL. "Easy SAML without paying the SSO tax" | Entry price is a `.star` config file, GitOps, Docker or Kubernetes, self-hosting, and secrets deferred to Vault/AWS SM. Their own words: "on-ramp to Kubernetes". **Our Builder would never install it** |
| **Replit** | Free tier already has "private or password-protected deployments". Core $25, Pro $100. Enterprise: SSO/SAML, custom groups, single-tenant, VPC peering | Audit logs and secrets management absent from pricing page. SSO behind "contact sales". Runtime serves Replit-built apps only |
| **Squadbase** | **Pivoted** — now an AI data-analyst/BI tool. SSO+SAML and audit log gated to $75/seat Team | No longer a competitor for app hosting |
| **Retool / Power Platform** | Adjacent incumbents with distribution; Power Platform already owns identity via Microsoft | They require building *in their* builder. We are agent-neutral |

### 2.1 What we must NOT claim as differentiation
Fast deploys (Val Town 100ms, Naïve 22s). Agent integration (Val Town ships it). Private URLs (Replit free tier, Vercel all tiers). Streamlit support (OpenRun). **An LLM proxy that keeps the provider key out of the app (Vercel AI Gateway, free, no markup — so we do not build one; see §2.2.2).** Governance features in the abstract (OpenRun has all of them, free). Fresh managed Postgres (Vercel marketplace, Naïve AgentDB, OpenRun service bindings).

### 2.2 What is genuinely ours
1. **Non-member viewer identity at $25.** Workspace SSO for people with no account on our platform, included in the base tier. Vercel charges Enterprise for this exact capability and Replit says "contact sales". This is the wedge.
2. **Enforced spend ceilings, not spend dashboards.** Everyone meters; we reject at 100%. P2 is a story about a cap that didn't exist, not a dashboard nobody watched. Feature-sized gap — a competitor closes it with a form field — so it wins the first sale, never the category.
3. **Brokered scoped access to systems the customer already runs.** Verified whitespace across Vercel, Naïve, and OpenRun — all three provision *fresh* resources, the easy version. This is the moat.
4. **Long-running Python tools as a first-class runtime.** Streamlit/Gradio hold session state and don't fit serverless. Our Builder ships them because that's what an agent writes for someone who can't write a Dockerfile.

**Ranked by durability:** 3 ≫ 1 > 4 > 2. Note that 1 is a packaging arbitrage — durable only because Vercel closing it would cannibalise Enterprise. It buys time; it is not a moat. Everything depends on 3 landing.

### 2.3 Positioning discipline
On the deploy surface we are bracketed: Vercel above, Naïve beside, Val Town alongside. **Do not invite the comparison.** Lead with Streamlit, where none of them compete. Next.js is supported so the Builder is never blocked, never marketed.

### 2.4 The lesson from Naïve's deprecations
They shipped 50 primitives and then froze the abstraction layer tying them together. Breadth bought neither focus nor defensibility. Our answer is §3.6: three connectors, in sequence, each gated on validation.

---

## 3. Product principles

1. **Frictionless surface, governed interior.** A Builder does zero setup. An Admin does one-time setup that all future apps inherit.
2. **Private by default.** Public requires an explicit confirm. Never the reverse.
3. **The app holds no brokered credential.** Not env vars containing provider or customer keys — a HobbyCloud-issued token or handle resolved at request time.
4. **One control, three states.** Permissions UI is domain / named emails / public. No custom roles until forced.
5. **Agent-neutral.** Never compete with the build step. Be where every agent deploys.
6. **Depth before breadth on connectors.** One connector done to Admin-approval standard beats five shallow ones.
7. **Buy isolation and breadth; build only the policy layer.** The enforced ceiling, the scope engine, and the audit corpus are ours. Microvms, model routing, and connector tails are someone else's business (§10).
8. **Every capability passes one checkpoint.** `(app, actor|null, capability, target, args) → allow | deny | require_approval`. LLM spend is the first policy type, not a special case (§4.0).

---

## 4. Roadmap

### 4.0 The two decisions everything inherits

**D1 — Buy isolation.** Fly Machines, one Firecracker microVM per app. Per-tenant kernel isolation as a purchased property. v1 budgeted 3–5 weeks of "build-and-attack" hardening in Stage 1 while §7 simultaneously said rent compute from Fly/Cloudflare — the doc implied both. Resolved: **buy.** Building shared-kernel container isolation ourselves makes Stage 0 8–10 weeks instead of 5, before anyone has confirmed they want the product. Consequence: **isolation is a Stage 0 concern, not Stage 1** — Stage 0's exit criterion ("25 apps from strangers") *is* the untrusted multi-tenant condition. External pentest still required before volume signup, but the surface under test is our proxy and control plane, not a hand-rolled sandbox.

**D2 — The policy checkpoint is general from day one.** One interface: `(app, actor|null, capability, target, args) → allow | deny | require_approval`, with every decision appended to an immutable log. Spend is policy type #1; DB scopes, Slack posts, and (much later) card authorizations are types #2–4. Shape borrowed from Naïve's approvals primitive — enforce at the capability boundary, log allowed/denied per call — but scoped to one policy type at a time.
- **`actor` is nullable from the first commit.** Stage 0 always has a human behind the request. Stage 3 does not. Recording `(app, null, …)` as a legal shape now is what makes agent-triggered work later without a schema migration, and §9's rule is right that the corpus cannot be backfilled.
- **The app is a principal, not a config bundle.** It holds scopes, it is revocable, and actions attribute to *it* plus whoever authorized the grant.
- **Failure mode this avoids:** writing an OpenAI-specific token counter. That makes Stage 2 a rewrite — the same warning v1 correctly gave about the secret store.

### Stage 0 — MVP: "It deploys, it's private, and it can't bankrupt you" (weeks 1–5)
**Goal:** a stranger signs up, ships a private team app, and sets a spend ceiling. **Solves:** P1 + P2.

> **Why P2 moved up from v1's Stage 1.** To meter outbound LLM calls you must sit in the path. Once you sit in the path, there is no reason to inject the provider key at all. One mechanism delivers three things: the non-negotiable rule becomes literally true in v0, the broker pattern ships in its cheapest form, and the only dollar-denominated pain — the stated paid trigger — gets tested four weeks earlier. v1 separated them for no reason.

Scope:
- `npx hobbycloud deploy` — zero-config auto-detect. **Streamlit first-class; Next.js supported.** Gradio and FastAPI deferred. No Dockerfile, no YAML, no config file.
- Live URL <90s. Scale to zero on idle. One microVM per app (D1).
- Private by default: Google Workspace domain. Visibility control: domain / named emails / public.
- Google Workspace SSO only, **for viewers who have no HobbyCloud account.** This is §2.2.1 and it is the whole wedge. **Not** Entra, Okta, or SAML.
- Rollback (last 5 deploys). Crash notification with last 100 log lines.
- **Provider-key broker + enforced ceiling:**
  - Builder stores an OpenAI/Anthropic/Google key in the org vault. It is **never injected into the app.**
  - App receives a HobbyCloud-scoped token; outbound provider calls route through our checkpoint, which resolves the real key at request time.
  - **Built on a third-party gateway** (Vercel AI Gateway, OpenRouter, or LiteLLM) — not from scratch. We inherit 300+ models and fallback for free and write only the part missing everywhere: the ceiling. See §10.
  - Per-app monthly ceiling. Alert at 50%. **Reject at 100% with a readable in-app message, not a crash.**
  - Rotate a key without redeploying. Usage dashboard: spend per app.
  - Every call logged as `(app, actor, capability, target, decision, cost, ts)`.
- Org secret store for non-brokered config, write-only after save. Scoped by the rule at the top of this doc.
- **Deploy-from-agent:** machine-readable CLI output, clear flags, and an MCP tool wrapping `deploy`. The Builder can't write a Dockerfile, so the agent runs the command — **this is distribution, not a feature.** Split out from v1's Stage 3 MCP item; capability *discovery* genuinely does need >1 connector and stays in Stage 3.

Out: connectors, DB brokering, audit UI beyond the spend log, Lead's inventory, org kill switch, custom domains, crons, multi-region.

Exit criteria:
- 25 apps deployed by people you don't know
- 10 paying cards at $25/user/mo
- **≥3 apps with a ceiling actually configured** — the real signal. Deploys prove P1 is interesting; a configured ceiling proves P2 is felt.
- Zero tenant-isolation incidents

### Stage 1 — "The Lead can see and stop it" (weeks 6–9)
**Goal:** the org-tier trigger. **Solves:** P5.

- Lead's inventory table: every app, owner, last opened, visibility, current spend.
- Per-app access log (who opened, when), CSV export, 90-day retention.
- Org-wide kill switch: offline + URL dead + tokens revoked, <10s.
- External pentest of the control plane and proxy before public signup at volume.

Exit criteria: first $250/mo org tier sold. Zero tenant-isolation incidents.

### Stage 2 — "It can touch our data" (weeks 10–17) ← the moat
**Goal:** brokered scoped Postgres. **Solves:** P4. **Gated on validation (§8.2).**

- Admin connects a Postgres instance once at org level. **Read replica is the documented default; primary is opt-in.** v1 treated "replica only" as a pivot branch — it is the modal answer, the engineering is identical, and it makes "passed a security review" achievable at week 17 without SOC 2.
- Admin defines named scopes: table + column allowlist, **read-only**. (Write scoping is a harder policy problem and won't be trusted yet.)
- App declares a required scope; the checkpoint (D2) resolves a short-lived credential at request time. Builder's code receives a handle — no URL, no password, nothing printable or loggable.
- Grant approval flow: app requests scope → Admin approves/denies. Policy type #2 on the existing checkpoint.
- Admin view: which apps hold which scopes. One-click revoke; pooled connections terminated.
- Query audit: every query attributable to app + person + time + scope.

Exit criteria: 3 orgs with production or read-replica Postgres connected. One passed security review.

### Stage 3 — "It's how our company builds" (months 5–9)
**The crossing: human-triggered → agent-triggered.** Stage 0–2 apps always have a person behind the request. This is where they don't, and it is the gate to everything in §10.
- **Non-human principal, fully realised.** Cron/webhook/agent-loop execution with an attributable identity, holding scopes, revocable. `actor = null` becomes a real path, not a reserved shape.
- **Per-action runtime approval.** Approval as a first-class suspended state the process resumes from — not an error, not an out-of-band webhook. (The thing Naïve got right.)
- **MCP capability discovery.** Agents enumerate the org's approved scopes and schemas and generate code against them. Requires >1 connector to be worth anything.
- Connector 2 and 3: Slack (post/read channel), then one CRM (Salesforce or HubSpot) with field-level scoping.
- Write scopes with approval + dry-run diff.
- Entra ID SSO. Okta/SAML/SCIM. Custom domains. Crons and background jobs.

### Stage 4 — "Nobody can rip us out" (months 10–18)
- **Scope recommendation engine** from the accumulated grant corpus: "3 similar apps were approved read-only on 2 columns."
- Snowflake / BigQuery connectors. Internal REST API connector with declarative scoping.
- Connector tail via resale (Composio/Nango) behind our scope layer (§10).
- Self-hosted / VPC deployment for regulated buyers.
- Egress policy and per-app network allowlists (the literal reading of "environment customization").

---

## 5. User stories

### Deploy & share (Stage 0)
- As a Builder, I run one command in my project directory and get a live URL in <90s without writing a Dockerfile, YAML, or config file.
  - *Accept:* zero-config deploy works for Streamlit and Next.js; failure output names the missing thing, not a stack trace.
- As a Builder, my app is private by default so I never accidentally ship something world-readable.
  - *Accept:* new deploys are domain-restricted with no action taken; going public requires explicit confirmation.
- As a Builder, I share the URL in Slack and my teammate just opens it — no invite, no account creation, no seat on my platform.
  - *Accept:* anyone with a matching Workspace domain lands in the app after one Google prompt, having never signed up for HobbyCloud.
- As a Builder using Claude Code/Cursor, my agent deploys for me. *Accept:* MCP tool wraps `deploy`; CLI emits machine-readable status and names failures precisely.
- As a Builder, I roll back to the last working version in one click. *Accept:* last 5 deploys retained; rollback <30s.
- As a Builder, I'm told when my app crashes. *Accept:* owner notified with last 100 log lines.

### Credentials & spend (Stage 0)
- As a Builder, I store my provider API key once and my app calls the model without ever holding that key.
  - *Accept:* key write-only after save; not retrievable via UI or API; **absent from the app's environment, memory, logs, and build output**; app authenticates with a HobbyCloud-issued token.
- As a Builder, I set a $50/month ceiling on my app's LLM spend so a shared link can't run up an unbounded bill.
  - *Accept:* alert at 50%; **at 100% the call is rejected** and the app renders a clear message; no crash, no overage.
- As a Builder, I rotate a compromised key in one click without redeploying. *Accept:* live <30s, no rebuild.

### Oversight (Stage 1 — the org-tier trigger)
- As a Lead, I see every app my team deployed, who owns it, when it was last opened, whether it's public, and what it's spending.
  - *Accept:* one table, sortable by "public", "last active", and "spend".
- As a Lead, I kill an app instantly when someone leaves or something looks wrong.
  - *Accept:* URL returns 410 <10s; tokens revoked; scopes revoked.
- As a Lead, I answer "who accessed this tool?" without asking the Builder. *Accept:* per-app log, CSV export, 90 days.

### Brokered data (Stage 2 — the moat)
- As an Admin, I connect our Postgres read replica once and define what any app may read, so I never hand a database password to a non-engineer.
  - *Accept:* table+column allowlist, read-only; credential never rendered in UI or API response.
- As a Builder, I query approved company data without ever seeing a connection string.
  - *Accept:* code receives a handle; no credential material is printable, loggable, or reachable from app memory.
- As an Admin, I approve or deny each app's scope request before it can read anything.
  - *Accept:* app is blocked with a clear message until approved; decision logged with actor and timestamp.
- As an Admin, I revoke an app's data access instantly. *Accept:* next query fails <10s; existing pooled connections terminated.
- As an Admin, I trace any query back to an app and a person. *Accept:* query log with app ID, actor, timestamp, scope used.

### Agent-native (Stage 3)
- As a Builder, my agent discovers what company data is available and writes code against it without me pasting schemas.
  - *Accept:* MCP server exposes approved scopes and schemas for the authenticated org only.
- As an Admin, a scheduled app acting with no human present is still fully attributable and revocable.
  - *Accept:* every action logs `(app, null, capability, target, decision)`; revoking the app's principal halts it <10s.
- As an Admin, an app pauses for my approval mid-run and resumes when I grant it.
  - *Accept:* approval is a durable suspended state surviving process restart; not an error, not a timeout.

---

## 6. Market size

**No credible third-party TAM figure is cited here on purpose** — I could not verify one from this environment, and an invented number is worse than none. Bottom-up only, assumptions stated so you can attack them.

### 6.1 Bottom-up
Unit = a team that (a) has agent-built internal tools and (b) has a Workspace/Entra tenant.

| Segment | Assumed count | Assumed ACV | Implied |
|---|---|---|---|
| Teams 10–50 people, self-serve seats | assume ~1.5M globally with Workspace/365 | $25/user × ~6 paying seats = ~$1.8k/yr | large but low realistic capture |
| Companies 50–500, org tier + 1 connector | assume ~400k globally | $3k–$12k/yr | **primary target** |
| Companies 500+, multi-connector + SSO + VPC | assume ~60k globally | $25k–$100k/yr | Stage 3–4 expansion |

Serviceable near-term target: **English-speaking, Workspace-first, 50–500 person tech-adjacent companies.** Assume ~80k reachable. At 1% penetration and $6k ACV → **~$4.8M ARR**. At 3% and $9k → **~$21M ARR**. That's the honest 3–5 year band for the core motion without enterprise.

### 6.2 Why the ceiling is higher than the small-software framing suggests
Koomen's premise ("one or a handful of users") caps *per-app* willingness to pay — which is why **we never price per app.** We price connections and seats. App count becomes a usage driver, not a price ceiling. A company with 200 tools on 6 connectors pays for the connections and the people, not the 200.

### 6.3 What would invalidate the sizing
- Builders turn out to be overwhelmingly individuals, not teams → seat model collapses, this becomes a $20/mo prosumer product.
- Companies refuse any brokered access, replicas included → org tier loses its anchor feature, ACV drops to ~$1.5k.
- Agent vendors or Vercel ship governed deploy as a bundled checkbox → capture rate falls sharply regardless of TAM.
- **Vercel moves Passport-equivalent viewer SSO into a self-serve tier** → §2.2.1 evaporates and acquisition gets materially harder. Watch their changelog.

---

## 7. Pricing by stage

| Stage | Tier | Price | Contains | Why |
|---|---|---|---|---|
| 0 | Hobby | Free | 1 private app, sleeps on idle, 1 collaborator, **no ceiling config** | Hard-limited on purpose. Usage-based infra + generous free tier is how bootstrapped companies die |
| 0 | Pro | **$25/user/mo** | Unlimited private apps, **Workspace SSO for non-member viewers**, key vault, **spend ceilings**, usage dashboard | SSO at $25 undercuts an *Enterprise* SKU at Vercel and a "contact sales" at Replit. Ceilings ship here because they are the paid trigger — P2 is the only dollar-denominated pain |
| 1 | Team | **$250/mo flat** | + app inventory, access logs + export, org kill switch | Flat, not per-seat: bought by the Lead, priced under discretionary approval to avoid procurement |
| 2 | Team | $250/mo + **$150/connector/mo** | + brokered Postgres, scopes, grant approval, query audit | Moat feature sits in the paid org tier. Per-connector is the expansion lever |
| 3 | Business | **$1,000–2,500/mo** | + multi-connector, write scopes, non-human principals, Entra/Okta/SAML, custom domains, MCP discovery | Where ACV steps up; still self-serve-able for the low end |
| 4 | Enterprise | **$25k–100k/yr** | + VPC/self-hosted, egress policy, scope recommendations, SLAs, DPA | Only after SOC 2. Do not chase before Stage 3 revenue exists |

### Pricing rules to hold
1. **Never price per app.** Contradicts the premise and caps you.
2. **Charge from day one.** Ten cards from strangers beats a thousand emails — the ~15 dead security scanners all collected emails.
3. **Rent compute (Fly), stay asset-light**, so COGS tracks revenue. Never sell unmetered compute at a flat price.
4. **Zero markup on tokens.** Vercel set this price at zero; matching it costs nothing and removes an objection. We sell the ceiling, not the tokens.
5. **Governance in the $25 tier; brokering in the org tier.** Governance is table stakes and copyable; brokering is the moat and should carry the price.
6. **Free tier never gets a connector, and never gets a ceiling.** Both are the things worth paying for.

---

## 8. Validation plan

### 8.1 Before writing code (week 0, ~1 week) — run both halves
v1 ran only the card test. A card test for an unbuilt deploy tool mostly measures whether you have an audience; with no audience, "stop" fires on a good idea. Pair it with the population you've never talked to.

**(a) Card test.** Landing page: one-sentence pitch + "your teammates sign in with their work Google account" + enforced-ceiling promise. **Ask for a card** for a $25/mo early plan, not an email.
- **Go:** 10 cards from strangers. **Stop:** only emails → you've learned what the 15 dead scanners learned, in a week.

**(b) Ten Builder conversations.** Ops/analytics/PM people shipping agent-built tools. This is research gap #1 and the cheapest week in the plan. Ask what they built, where it runs now, who opened it, and what it cost them. Do not pitch.
- **Signal to look for:** they already shipped something and are uneasy about who can reach it. **Anti-signal:** they're building for themselves alone → seat model collapses (§6.3).

### 8.2 Before building the broker (by week 9 — gates 8 weeks of work and the whole moat)
Ask 5 Admins one question: *would you let a non-engineer's tool query your Postgres if they never held a credential and you controlled which columns?*
- **Go:** ≥2 yes on **replica** → build Stage 2 as specified. Replica is the default ask, not the fallback.
- **Escalate:** yes on primary → same product, better story for §9.3.
- **Stop:** universal no, replicas included → the moat isn't data brokering; fall back to egress/runtime policy as the "environment customization" play.

### 8.3 Open research gaps
1. **Builder persona is unvalidated.** All evidence is HN, which skews senior-technical — the opposite of the target Builder. §8.1(b) fixes this.
2. No evidence anyone *pays* for P1. Interest is well-evidenced (Onu 160pts); payment is not.
3. **Confirm Vercel has no per-project hard cap.** Absence in the usage docs is strong but not conclusive — prepaid credits act as an account-level ceiling and per-key limits may be documented elsewhere. Spend ten minutes in a real AI Gateway dashboard **before** making enforced ceilings a headline. If they ship per-project rejection, §2.2.2 is gone and the pitch rests on 1, 3, and 4.
4. Val Town's actual auth/secrets/team capabilities are undocumented on their landing page — read their docs before finalizing differentiation.
5. Read Naïve's "What is served today" page. Their primitives page shows declared-but-refused APIs; this separates competitor from cautionary tale.
6. OpenRun's license and any commercial offering unverified.

---

## 9. Keeping the moat

Three layers, increasing durability. Only the third is defensible long-term.

1. **Packaging (weak, months).** Zero-config + non-member SSO at $25 + enforced ceilings. Vercel can close the SSO gap with a pricing-page edit; anyone can add a ceiling form field. Real head start — Vercel closing it cannibalises Enterprise — but never the moat.
2. **Per-account accumulation (medium, years).** Each connection an Admin configures makes the next app cheaper; by app 20, building is nearly free. Switching = re-establish every connection, re-grant every scope, migrate every running tool people depend on daily. Within-account compounding — how Datadog and Snowflake actually won. Protects accounts you've won; does not help win the next.
3. **Scoping engineering + policy corpus (strong).** Per-system scoped brokering (Postgres row/column, then Salesforce field-level, then Snowflake, then bespoke internal auth) is a grind nobody wants — the same shape of moat that made AgentSync defensible (NIPR integration + 302 tracked state changes, not AI). Every grant approved or denied teaches you a real access policy; at volume that becomes automatic scope proposals a new entrant can't cold-start.

### Rules that preserve it
- **No brokered credential in an app environment, ever** — including v0, per the top of this doc. Shipping the provider key as an env var to save a week in Stage 0 forfeits everything: you become a deploy tool competing with Vercel on DX, and §10 is unreachable.
- **The checkpoint stays general (D2).** An OpenAI-specific meter makes Stage 2 a rewrite.
- **`actor` is nullable from commit one.** The agent-triggered corpus can't be backfilled.
- **Agent-neutral, always.** Companies run Claude Code, Cursor, Copilot, and Lovable simultaneously. Be the runtime all of them deploy into; never fight the build step.
- **Log every grant decision from day one**, before there's any use for the data.
- **Resist connector breadth** until one connector passes real security reviews. Naïve froze six primitives learning this.

### Honest risks
- Per-account moat doesn't help acquisition.
- Vercel and Naïve both already ship the deploy + key-broker surface. Our Stage 0 differentiation is one gated feature (viewer SSO) and one form field (rejection), which is thin until Stage 2 lands.
- Retool and Power Platform hold adjacent ground with distribution; Power Platform already owns identity.
- Agent vendors could ship "governed deploy" as a checkbox and compress the category.
- Building infrastructure before having users is the classic way to spend a year on something nobody adopts — which is why Stage 0 ships in 5 weeks and Stage 2 is gated on §8.2.

---

## 10. Reaching platform breadth without becoming Naïve

Naïve's 50 primitives are the visible version of "agent infrastructure". We reach ~60% of that surface by resale, skip the rest permanently, and go deeper than they can on the one layer that compounds. **Two assets make the whole list cheap to acquire, and neither can be bought:**

1. **Being the runtime every agent-built tool deploys into** (Stages 0–1). Once we are the process, enforcement is by construction, not cooperation. This is what makes each later capability sellable as *governed* rather than as resale.
2. **The per-account capability graph** (Stages 2–3). Naïve provisions greenfield resources, so nothing accumulates. We accumulate a map of a real company's systems and who may touch what.

| Category | Decision | How | When |
|---|---|---|---|
| Governance: policy, ceilings, approvals, audit | **Build** | The spine. D2's checkpoint, one policy type at a time | Stages 0–2 |
| Model routing (300+ models, fallback) | **Resell** | Sit on a gateway; own only the enforced ceiling | Stage 0 |
| Scoped access to existing systems | **Build — this is the business** | Postgres → Slack → CRM, depth-first | Stages 2–3 |
| Managed Postgres / storage / queues | **Resell** | Neon/Supabase when customers ask. Scoping an existing DB is strictly harder than provisioning a fresh one, so this is the easy half | Stage 3 |
| Integration breadth (100+) | **Resell behind our scope layer** | 3 deep connectors + Composio/Nango for the tail | Stage 4 |
| Card issuing / spend at authorization | **Buy, late** | Lithic or Stripe Issuing. Stage 0–1 proxy logs are the underwriting evidence and the policy corpus; enforcement moves from our proxy to card auth — same policy engine, stronger enforcement point | Stage 4+ |
| Legal identity: LLC, EIN, KYB | **Never** | Only needed when the customer *is* the agent business. Ours already has an entity, a Workspace tenant, and an Admin with veto power. Zero value, unbounded liability | — |
| Autonomous SaaS signup, brokerage orders, crypto wallets | **Never** | Naïve's AgentBrowser vaults credentials it captured by signing up on the user's behalf; AgentTrade places regulated orders. Neither has an answer to who holds liability. Not our risk appetite | — |

**Why the sequencing works — each stage produces the next one's input:**
- Stage 0–1 → distribution and spend data. Naïve has neither; card programs and BaaS resale are easy *given* users and impossible without.
- Stage 2 → the trust to hold credentials. Once an Admin lets us broker their Postgres, holding a card is a smaller ask, not a bigger one. The reverse order is why Naïve's liability questions are unanswerable.
- Stage 3 → the non-human principal and MCP surface. The crossing from deploy tool to agent platform.
- Stage 4 → breadth as resale, priced as governance rather than compute.

**"Naïve level" is not the goal.** Their breadth is a symptom of not knowing which piece is the business. We have a thesis about which piece it is, and it is the one piece none of them cover.

---

## 11. Changelog v1 → v2

| # | Change | Reason |
|---|---|---|
| 1 | Non-negotiable rule rescoped to "systems HobbyCloud did not issue", with explicit covered/not-covered lists | v1 stated it three incompatible ways (lines 9, 98, 261) and broke it in Stage 0. An unenforceable rule stops constraining |
| 2 | Spend ceilings + provider-key broker moved Stage 1 → **Stage 0** | Metering requires sitting in the path; once there, don't inject the key. Makes the rule true in v0, ships the broker pattern cheaply, tests the paid trigger 4 weeks earlier |
| 3 | Proxy built **on a third-party gateway**, not from scratch | Vercel AI Gateway ships routing/BYOK/fallback at zero markup. Build only the missing ceiling |
| 4 | **D1: buy isolation** (Fly Machines/Firecracker); sandbox hardening moved to Stage 0 | v1 implied both buy and build; and Stage 0's exit criterion already *is* the untrusted multi-tenant condition. Keeps Stage 0 at 5 weeks instead of 8–10 |
| 5 | **D2: general policy checkpoint** + nullable `actor` + app-as-principal from commit one | Prevents Stage 2 rewrite; the agent-triggered corpus can't be backfilled |
| 6 | **Vercel added** as closest MVP competitor; AI Gateway and Passport gating documented | Vercel already ships the key-broker pattern; the real gap is non-member viewer identity, which is Enterprise there |
| 7 | **Naïve added**; 50 primitives inventoried, 6 deprecations noted; verified zero coverage of existing-DB scoping | Strongest available evidence that §2.2.3 is genuine whitespace, and a live lesson about breadth |
| 8 | Differentiators re-ranked; **non-member SSO at $25** promoted to the wedge | Was implicit in v1's pricing; it is the actual reason a stranger picks us over Vercel |
| 9 | **Streamlit promoted to first-class, Next.js de-marketed** | Next.js invites a comparison with Vercel we lose. Streamlit is where no competitor plays and where the Builder lives |
| 10 | MCP split: **deploy-from-agent → Stage 0**, capability discovery stays Stage 3 | Agents are the distribution channel for a Builder who can't write a Dockerfile; only *discovery* needs >1 connector |
| 11 | **Read replica is the Stage 2 default**, not the pivot branch | Modal answer, identical engineering, makes a passed security review reachable without SOC 2 |
| 12 | Stage 3 reframed as **human-triggered → agent-triggered crossing** | Names the actual gate to §10 instead of listing features |
| 13 | §8.1 now runs **card test + 10 Builder conversations** | Card test alone measures audience, not idea; and gap #1 was flagged unvalidated but unaddressed |
| 14 | §10 added: build / resell / never table | Answers "how do we reach platform breadth" without repeating Naïve's mistake |
| 15 | Stage 0 exit adds **≥3 configured ceilings**; §6.3 and §8.3 add Vercel-specific invalidators | Deploys prove interest; a configured ceiling proves felt pain |
