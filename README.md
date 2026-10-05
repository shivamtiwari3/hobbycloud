# HobbyCloud

**Push a directory, get a private URL your team opens with their work login. The app can query and spend against your real systems through connections the builder never holds a credential for.**

This repo is the product spec, written in the open. Read the full [PRD](PRD.md).

## The bet

Agents made small software trivial to build and left it hard to deploy safely. Deploying is getting commoditised: Vercel, Val Town and Naïve all do it. Giving an internal app scoped access to a company's real systems without handing a human the keys is still unsolved.

So the plan is to ship the easy surface first to get adoption, and build the brokered-credential layer underneath to keep it.

**The one rule:** no credential to a system HobbyCloud did not issue ever lands in an app's runtime. The app gets a scoped HobbyCloud token. We hold the real key and swap it in at the boundary.

## What's in the PRD

| Section | What it covers |
|---|---|
| Problem | Who has it, the evidence (with sources), and the trap to avoid |
| Competitive position | Vercel, Replit, Val Town, Naïve and others. What we can't claim, and what is actually ours |
| Roadmap | Five stages, from "it deploys, it's private, and it can't bankrupt you" to "nobody can rip us out" |
| User stories | Deploy and share, credentials and spend, oversight, brokered data, agent-native |
| Market size | Bottom-up sizing, and what would invalidate it |
| Pricing | Pricing for each stage, and the rules to hold |
| Validation plan | What to prove before writing code, and before building the broker |
| Keeping the moat | Rules that protect it, and the honest risks |

## Roadmap at a glance

| Stage | Promise | Proof it worked |
|---|---|---|
| 0 | It deploys, it's private, and it can't bankrupt you | 25 apps from strangers, 10 paying cards, 3+ apps with a spend ceiling set |
| 1 | The Lead can see and stop it | First $250/mo org tier sold |
| 2 | It can touch our data (the moat) | 3 orgs with Postgres connected, one through a security review |
| 3 | It's how our company builds | Agent-triggered apps with their own identity, per-action approval, MCP discovery |
| 4 | Nobody can rip us out | Scope recommendations, warehouse connectors, self-hosted/VPC |

## Status

Draft v2 (2026-08-09). Spec only, no code yet. Feedback welcome: open an issue.
