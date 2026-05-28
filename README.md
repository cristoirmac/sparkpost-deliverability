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

### As a single-skill plugin (simplest)

```bash
mkdir -p ~/.claude/skills/sparkpost-deliverability
curl -L https://raw.githubusercontent.com/cristoirmac/sparkpost-deliverability/main/SKILL.md \
  -o ~/.claude/skills/sparkpost-deliverability/SKILL.md
```

Then restart Claude Code. Invoke with `/sparkpost-deliverability <domain | pool-name | "account">` or just ask in natural language ("is acme.org's deliverability OK?", "how is our SparkPost sending overall?", "why is X on the shared pool?").

### As a Claude Code plugin marketplace

If you want to install via the plugin system (and pull updates more cleanly):

```bash
claude plugins marketplace add github:cristoirmac/sparkpost-deliverability
claude plugins install sparkpost-deliverability
```

Then `claude plugins update sparkpost-deliverability` to pull future revisions.

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

Extracted and sanitized from an internal Claude Code skill built and refined against several million email sends per week on a SaaS platform that sends through SparkPost on behalf of customers. The generic version drops product names, customer names, internal docs page IDs, and platform-specific tier-routing details; the first-run customization re-introduces those as the operator's own configuration.

## License

MIT.
