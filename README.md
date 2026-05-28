# sparkpost-deliverability

A Claude Code skill for investigating email deliverability on a SaaS sending platform that uses [SparkPost](https://www.sparkpost.com/).

It pulls live SparkPost metrics, runs DNS authentication checks (SPF / DKIM / DMARC), correlates with your application logs, and produces an advisor-style brief — with every conclusion graded `[FACT]` or `[HYPOTHESIS]`, recovery times as ranges, and recommended next probes.

## What it does

- **Three investigation modes:** account-wide, single sending domain, single SparkPost IP pool
- **Multi-window comparison** (7d / 30d / 90d) to separate trends from one-off campaigns
- **DNS authentication audit** for any customer sending domain, with the SparkPost API as ground truth for the configured DKIM selector (no selector guessing)
- **App-log correlation** for symptoms where SparkPost itself shows nothing (e.g. injection rejections like `Unverified Sending Domain Code: 7001`)
- **Reputation probes** against the major blocklists (Spamhaus / SURBL / URIBL / SpamCop / Barracuda / SORBS) when symptoms warrant
- **Cohort comparison** to determine whether an anomaly is customer-specific or shared infrastructure
- **Optional persistent readout** in your docs system (Confluence / Google Docs / Notion / GitHub wiki — chosen during customization) with a status emoji and BLUF block, for delta-tracking across runs

## First-run customization

On first use, the skill offers a ~2-minute interview (logging tool, docs system, bounce-subdomain convention, tier-routing model, escalation contacts, etc.) and rewrites its own SKILL.md in place to swap generic placeholders for your platform's specifics. Subsequent investigations run against the customized version. You can re-customize later by asking the skill to.

If you skip the interview, the skill asks the same questions ad-hoc as they come up.

## Install

The repo is a Claude Code plugin marketplace — install via the standard `claude plugins` flow:

```bash
claude plugins marketplace add github:cristoirmac/sparkpost-deliverability
claude plugins install sparkpost-deliverability@cristoirmac-sparkpost-deliverability
claude plugins list   # verify
```

Restart Claude Code to load the skill. Pull future revisions with `claude plugins update sparkpost-deliverability`.

### Invoke

Either use the slash command:

```
/sparkpost-deliverability <domain | pool-name | "account">
```

…or ask in natural language: "is acme.org's deliverability OK?", "how is our SparkPost sending overall?", "why is X on the shared pool?". The skill auto-triggers from those phrasings.

### Quick install without the plugin system

Copy `SKILL.md` directly into your user-level skills directory:

```bash
mkdir -p ~/.claude/skills/sparkpost-deliverability
curl -L https://raw.githubusercontent.com/cristoirmac/sparkpost-deliverability/main/skills/sparkpost-deliverability/SKILL.md \
  -o ~/.claude/skills/sparkpost-deliverability/SKILL.md
```

Restart Claude Code. Same invocation patterns as above.

## Requirements

- Claude Code installed: <https://docs.claude.com/en/docs/claude-code/quickstart>
- A SparkPost API key with `Metrics: Read` and `Sending Domains: Read/Write` scopes
- macOS Keychain (the skill stores the SparkPost API key via `security add-generic-password -s sparkpost-api-key`). On Linux / Windows the storage line in SKILL.md needs adjusting — easy edit.
- Access to your platform's application logs — Datadog by default; the skill's first-run interview asks which tool you use and adapts queries accordingly.

## What this skill is NOT

- A verdict machine — every conclusion is fact-vs-hypothesis labelled
- A customer-facing report generator — your CSM/AM rewrites the brief in customer-safe language before sending; this skill produces internal-language analysis
- An incident-response tool — active outages should go to your incident runbook
- An autonomous doc editor — doc-drift suggestions are recommendations; the human decides whether to update

## Background

This skill was built and refined inside a SaaS platform sending several million emails per week through SparkPost on behalf of customers. The published version is the sanitized core — product names, customer names, and internal docs page IDs stripped; the first-run customization re-introduces those as the operator's own configuration.

Author: Chris McFadden ([@cristoirmac](https://github.com/cristoirmac)). Currently CTO at Quorum, where the team operates this skill against production SparkPost traffic. Previously VP of Engineering at Message Systems / SparkPost (2014–2021) and Director of Engineering (2012–2014) — led SparkPost's transition from on-prem enterprise software to a cloud-native, API-first SaaS platform processing over one billion messages per day at peak. One of the original authors of the SparkPost metrics and events API endpoints (`/api/v1/metrics/deliverability/*`, `/api/v1/events/message`) that this skill uses. Continued as VP of Engineering after MessageBird's $600M acquisition of SparkPost in 2021, leading the SparkPost engineering integration.

The defensive notes in the skill — `/metrics/deliverability` vs the nonexistent `/aggregate`, `sending_domains=` vs `domains=`, looking up the DKIM selector via the API before digging DNS, the 30-day rolling rate vs same-day block counts — come from running SparkPost in production over that period.

## Contributing

PRs welcome — corrections, new durable diagnostic patterns, adapter notes for non-Datadog logging tools or non-Confluence docs systems, and sanitization fixes are the highest-value contributions. See [CONTRIBUTING.md](./CONTRIBUTING.md) for scope and mechanics.

## License

[MIT](./LICENSE).
