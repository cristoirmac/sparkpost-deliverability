---
name: sparkpost-deliverability
description: Use when investigating email deliverability for a SaaS product that sends through SparkPost — overall account health, a single customer's sending domain, or a single SparkPost IP pool. Pulls live SparkPost metrics, runs DNS authentication checks (SPF/DKIM/DMARC), and produces an advisor-style brief with fact-vs-hypothesis labelling and ranges-not-point-estimates. Use when a CSM, AM, or Support teammate asks "is acme.org's deliverability OK?", "why is X on the shared pool?", or "how is our sending overall?" Also use when investigating DKIM/SPF/DMARC posture, ESP commingling at apex, or sending-domain verification status.
---

# SparkPost Deliverability Investigator

A skill for SaaS senders that send mail on behalf of customers through SparkPost. Investigates deliverability questions, produces an evidence-graded brief, and (optionally) publishes a persistent per-customer readout to your docs system.

## When to use
- A CSM/AM/Support teammate asks about deliverability for one of your customers (overall, single domain, or single pool)
- You need to interpret SparkPost metrics in plain language and decide what's actionable
- You need to check a customer's DNS authentication posture (SPF, DKIM, DMARC, ESP commingling at apex)
- A specific symptom needs a root cause: bounce spike, Block storm, complaint surge, throttling

## When NOT to use
- Active outage / incident → use your platform's incident runbook and escalate per your ESP support procedures
- You need to *fix* a deliverability problem — this skill investigates and advises; remediation is the human's job
- The domain isn't in your SparkPost account — the API key is scoped to your subaccount, you'll get no data anyway

## First-run customization — offer to tailor this skill to your platform

This skill is written generically. Real platforms have specific tooling, naming conventions, and escalation paths that the skill should know about so it stops asking the same questions every run.

**On first use, before doing any investigation, ask the user:**

> Want me to customize this skill to your platform first? I'll ask a handful of questions (logging tool, docs system, bounce subdomain, escalation contact, tier-routing model) and rewrite the placeholders in this file with your specifics. About 2 minutes. Otherwise I'll ask the questions ad-hoc as they come up.
> (yes, customize / no, just answer my question)

If yes, walk through these questions and **edit this SKILL.md file in place** to replace each placeholder with the user's answer. Make the edits via the Edit tool against this file, then continue the investigation against the customized version.

### Customization questions

1. **Platform name** — what's the product/company name to use in chat references? (e.g. "Acme Mail", "Initech Mailer"). Replace any `your platform` / `your SaaS` references with this name.

2. **Logging tool** — where do application-side errors (transmission failures, dispatcher logs) live?
   - Datadog (default — adapt log query syntax for Datadog)
   - Splunk
   - CloudWatch Logs
   - Google Cloud Logging
   - Honeycomb
   - Loki / Grafana
   - Something else (user names it)

   Record the choice in the "Logging system" section below. Subsequent log queries use that tool's DSL.

3. **Docs / publishing system** — where do persistent customer readouts and the FAQ live?
   - Confluence (default — `confluence_create_page` / `confluence_update_page` / `confluence_upload_attachment` tools)
   - Google Docs (Drive API or `mcp__google-drive__*` tools if available)
   - Notion
   - GitHub Wiki / markdown in a repo
   - Internal CMS (user names it)
   - None — just produce the brief in chat

   For each readout-publishing reference below, replace "Confluence" with the chosen system and adapt the tool calls accordingly. If "None", strip the §11 "Offer to post a persistent readout" section entirely on customization.

4. **Readouts location** — once the publishing system is chosen, what's the parent location for per-customer readout pages? (e.g. Confluence space key + parent page ID, Google Drive folder URL, Notion database ID, repo path)

5. **Customer-facing canonical FAQ** (optional) — is there a single canonical "how deliverability works at <your platform>" page the skill should cite? URL + a one-line description. Many platforms don't have one yet — that's fine, leave it blank.

6. **Bounce subdomain convention** — does your platform CNAME a customer subdomain to `sparkpostmail.com` for the SMTP return-path?
   - Yes — what's the convention? (e.g. `mail-bounces.<customer-domain>`, `sp.<customer-domain>`, `bounces.<customer-domain>`). Used in DNS-check step 5.
   - No, we use SparkPost's default return-path for everyone — drop the bounce-CNAME check from DNS step 5.
   - Mixed — some customers have a custom subdomain, some use the default. Check `is_default_bounce_domain` per-customer.

7. **Tier-routing model** — does your platform run a tier-rotation layer over SparkPost's IP pools?
   - Yes — what are the pool names (good / purgatory / shared / customer-private / etc.)? What rolling-window does the tier cron evaluate? What are the per-category thresholds for demotion?
   - No — strip the "Pool-tier rotation" section.

8. **SparkPost CSM contact** (optional) — who's the human at SparkPost to escalate to when the docs aren't enough? Name + email or just "ask <internal-team-name>".

9. **ESP-at-apex commingling concerns** — do you have a prior-ESP migration where stale DKIM records at customer apex domains are a known problem (SendGrid, Mailchimp, etc.)? If yes, note which ESP and how to identify migration residue vs. active third-party use.

10. **Common customer-side patterns** — any platform-specific "this almost always means X" patterns worth pre-loading? (e.g. "INTERNAL_HEAVY senders always have one mailbox provider")

### What to write back into the skill

For each answered question, edit the relevant section of this SKILL.md file:

| Question | Section(s) to edit |
|---|---|
| 1. Platform name | Anywhere "your platform" / "your SaaS" appears |
| 2. Logging tool | "Logging system" section + step 3c |
| 3. Docs system | "Standing behavior: ask before publishing" + step 11 + tool references |
| 4. Readouts location | Step 11 placeholders |
| 5. Canonical FAQ | Add reference under "What this skill is NOT" or at the top |
| 6. Bounce subdomain | DNS check step 5 |
| 7. Tier-routing | "Pool-tier rotation" section + step 3 references to 30-day window |
| 8. SparkPost CSM | "Standing behavior: search SparkPost support docs" escalation paragraph |
| 9. ESP-at-apex | DNS check step 6 |
| 10. Custom patterns | Step 3 "Always cross-check" — add as a sub-bullet |

After editing, confirm to the user: "Customized. Saved <N> changes to SKILL.md. Want to run an investigation now?"

If the user later wants to re-customize, they can invoke: "re-customize the deliverability skill". The skill should then re-ask the questions and update.

## Three investigation modes
Ask the user (if unclear) which mode applies:

1. **Account-wide** — overall sending health across all customer domains
2. **Single domain** — one customer's sending domain (e.g. `acme.org`)
3. **Single pool** — one SparkPost IP pool (e.g. `shared`, a private customer pool, or your platform's tier-specific pools)

## Setup (one-time)

SparkPost API key lives in macOS Keychain. To check whether it's stored:

```bash
security find-generic-password -s sparkpost-api-key -w 2>/dev/null | head -c 8
```

If empty, prompt the user: "Save your SparkPost API key?" then run (replacing `<key>`):

```bash
security add-generic-password -s sparkpost-api-key -a "$USER" -w <key>
```

**Never echo the key back to chat.** Always pipe `security find-generic-password ... -w` directly into the API call.

The API endpoint is `https://api.sparkpost.com/api/v1/...` with header `Authorization: <key>`.

## Logging system — default to Datadog, ask otherwise

The skill assumes **Datadog** for application-side logs (transmission errors, dispatcher logs, ESP-side rejections that don't show up in SparkPost's UI). Many deliverability symptoms are explained by app-side logs, not SparkPost metrics — e.g. SparkPost returning `400 Unverified Sending Domain` on injection won't appear in SparkPost's deliverability reports at all; you have to read it from your app logs.

**On first use in a session**, confirm:
> Default logging tool is Datadog. If you use something else (Splunk, CloudWatch, GCP Logging, Honeycomb, etc.), tell me which one and I'll adapt the queries. Otherwise I'll search Datadog.

Adapt query syntax to the tool the user names. The *pattern* — search for the SparkPost error string, the customer's sending domain, or the message IDs / compose IDs — is the same; only the query DSL changes.

## Standing behavior: ask before publishing to docs

The skill can write to your docs system in a few places — per-customer deliverability readouts, the canonical FAQ, the README that hosts the release zip. **Always ask before writing.** Investigations are sometimes exploratory; doc updates are sometimes drafty. The user should choose what becomes persistent.

Pattern: after producing the content in chat, end with a one-line prompt naming what would change and where:

> Want me to publish this as a readout under `<your-readouts-parent>`? (yes / no)

Wait for explicit confirmation before calling any `create_page` / `update_page` / `upload_attachment` tool. Reading pages + commenting on existing pages is fine without asking.

## Standing behavior: search SparkPost support docs when in doubt

When you encounter an unfamiliar bounce/delay reason, an unrecognized API field, an ambiguous error code, or any other SparkPost-specific behavior — **search the support docs first**. They're the authoritative source.

Starter URLs (curate your own list over time):
- SparkPost API reference: <https://developers.sparkpost.com/api/>
- SparkPost classification codes: <https://support.sparkpost.com/docs/deliverability/bounce-classification-codes>
- SparkPost deliverability docs: <https://www.sparkpost.com/docs/deliverability/>

Pattern:
1. Use WebFetch against the specific support doc URL with a narrow question
2. Cite the URL inline in the brief — `[FACT, per SparkPost docs <URL>]`
3. Document the finding so future investigations don't re-research

Escalate to your SparkPost CSM when: a bounce reason isn't in their documented classification codes, behavior contradicts the docs, or you suspect a SparkPost-side issue (sending IP suspended, sub-account misconfiguration, listing on a SparkPost-shared IP).

## Investigation protocol (all modes)

1. **Confirm time window.** Default last 30 days. Ask if anything else.

2. **Pull the primary metric set** for the mode (all paths under `/api/v1/metrics/...`):
   - **Account-wide:** `/metrics/deliverability?from=...&to=...` for the top line; `/metrics/deliverability/sending-domain?from=...&to=...&order_by=count_targeted&limit=20` for top domains; `/metrics/deliverability/ip-pool?from=...&to=...` for pool-level breakdown
   - **Single domain:** **First** `GET /api/v1/sending-domains/<domain>` (note: under `/api/v1/sending-domains/`, *not* `/metrics/`) to capture the configured DKIM selector + auth status (`dkim_status`, `spf_status`, `ownership_verified`, `is_default_bounce_domain`). Then `/metrics/deliverability?from=...&to=...&sending_domains=<domain>` for the top line, `/metrics/deliverability/mailbox-provider?from=...&to=...&sending_domains=<domain>` for the per-provider slice, and `/metrics/deliverability/time-series?from=...&to=...&sending_domains=<domain>&precision=day` to separate single-campaign events from trends.
   - **Single pool:** `/metrics/deliverability/ip-pool?from=...&to=...&ip_pools=<pool>` + `/metrics/deliverability/sending-domain?from=...&to=...&ip_pools=<pool>&order_by=count_targeted` to enumerate who's on it

   **Endpoint + filter gotchas:**
   - There is **no `/metrics/deliverability/aggregate`** — that path 404s. Use `/metrics/deliverability` for the account-wide aggregate.
   - **`domains=<domain>` filters by RECIPIENT domain** (mail sent TO `@acme.org`), not sender. Almost never what you want for a customer brief. Use **`sending_domains=<domain>`** for "how is acme.org's outbound doing?" — this is the single most common analytical bug.
   - `mailbox_providers=<Name>` (URL-encode spaces) works on `/metrics/deliverability`, `/metrics/deliverability/mailbox-provider`, and `/metrics/deliverability/time-series`. Provider names match what `/metrics/deliverability/mailbox-provider` returns: `Gmail`, `Yahoo`, `Hotmail / Outlook`, `Office 365`, `Apple`, `Proofpoint`, `Comcast`, `Gsuite`, `Other`, etc.
   - `/metrics/deliverability/bounce-classification` and `/metrics/deliverability/bounce-reason` may return an **empty `results` array** when filtered by `sending_domains=` alone (observed for high-volume senders). If you need bounce-reason text for one sender's traffic and these endpoints come back empty, fall back to `/metrics/deliverability/bounce-reason/domain` (recipient-domain breakdown) or the message-events search (`GET /api/v1/events/message`).

3. **Always cross-check with at least 2 secondary slices.** Single numbers lie.
   - For a domain: by mailbox provider + by bounce classification + by top recipients (if a single bad address dominates)
   - For a pool: by domain (who's on it) + by mailbox provider
   - For the account: by domain (top 20) + by pool
   - **When Hard / Invalid Recipient (code 10) dominates:** ALSO pull the per-day Hard-vs-Admin time-series — if Hard stays high while Admin stays near 0%, this is a list-acquisition problem (fresh addresses), NOT a list-maintenance one.
   - **When the mailbox-provider breakdown shows fewer than 3 providers with ≥1K targeted in the window**, the mailbox-provider dimension carries no diagnostic signal — pivot to the per-recipient-domain breakdown via `/metrics/deliverability/domain`. Typical for internal-heavy corporate senders: 99%+ of volume routes through their own corporate filter (Proofpoint, Mimecast, etc.), so "mailbox provider" collapses to 1. Recipient-domain dimension reveals typo variants (e.g. `acmme.com`, `acmecorp.co`), defunct subsidiaries, personal-address contamination, and partner-domain engagement quality that are invisible at the aggregate level.

3a. **Always run multi-window comparison (7d / 30d / 90d)** for any non-trivial investigation. The 30d view is what most tier/pool-routing logic typically evaluates; 7d shows current behavior; 90d shows reputation trajectory. Three windows together reveal recovery / collapse / steady-state stories that the 30d view alone hides.

3b. **When bounce rates are elevated (>2% Hard or >5% Block), drill into recipient + reason-text patterns using the proper metrics endpoints.** Pull:
   - **`/metrics/deliverability/bounce-reason`** — server-aggregated bounce reasons with classification labels (6-month retention; preferred over events API parsing).
   - **`/metrics/deliverability/bounce-reason/domain`** — bounce reasons broken down by recipient domain in ONE query.
   - **`/metrics/deliverability/rejection-reason`** — block-side rejections (separate from bounces).
   - **`/metrics/deliverability/delay-reason`** AND **`/metrics/deliverability/delay-reason/domain`** — delay reasons aggregated. Critical when bounces show receiver-throttling signals (Soft Transient code 70 dominant, Gmail `4.7.x` codes, Comcast/Microsoft rate-limit references). Surfaces signals that never appear in bounce data because messages eventually deliver, plus SparkPost-internal IP suspension messages (`451 4.3.0 [internal] Sending IP temporarily suspended`) that ONLY appear in delay data.
   - **`/metrics/deliverability/attempt`** — delivery attempt distribution (1 / 2 / 3+). If >20% require 3+ attempts, that's a heavy receiver-throttling signature even when bounce rate looks healthy.
   - **`/api/v1/events/message` events API** — only for spot-checks of specific recipients / messages, NOT for aggregate analysis. Events API is capped at 10K/page and retained 10 days; the metrics endpoints don't have those limits and aggregate server-side. The Gmail "very low reputation" diagnostic in delay reason text is the closest thing to a Postmaster Tools reading you can get from SparkPost.

3c. **When the symptom is "mail isn't arriving / isn't in SparkPost at all"** — that's almost always SparkPost rejecting at injection. Read **your app-side logs** (Datadog by default — see Logging section above) for the `EmailSender` / `Transmissions` / your platform's mail-dispatch error string. Common causes:
   - **`Unverified Sending Domain <X> Code: 7001`** — the customer's sending domain is registered in SparkPost but DNS verification isn't complete (`dkim_status: unverified`). Reject happens BEFORE the message hits SparkPost's deliverability reports, so SparkPost UI shows nothing. Pull `/api/v1/sending-domains/<X>` to confirm status; fix is publishing DKIM TXT at the selector SparkPost generated.
   - **Per-domain rate limit / IP suspension** — surface in delay-reason data, not bounces.
   - **Customer-side suppression hit** — message dropped before transmission; in your app logs, not SparkPost.

3d. **When Block bounces are present or reason-text references blocklists, run reputation probes.** DNS-based checks against Spamhaus DBL/ZEN, SURBL, URIBL, SpamCop, Barracuda, SORBS — free, fast, well-documented. Listing-driven blocks are catastrophic and need a different escalation path than reputation-driven rejection. **Skip these checks unless one of the trigger conditions is met** — they're conditional, not default.

3e. **When a customer-specific signal looks anomalous and the rest of the diagnostic doesn't explain it, run cohort comparison.** Pull the same metric (e.g. block rate at Gmail, soft-bounce rate at Microsoft) across the account-top-50 sending domains to determine whether the issue is customer-specific or shared infrastructure. **Trigger conditions:** one ISP showing >50% block on this customer; Block >5% with no blocklist references; sudden Hard spike without obvious campaign; Soft Transient >10% at multiple providers; pool placement doesn't match per-category bounce rates. **Skip if no trigger fires** — most investigations don't need this.

4. **Look at the full metric set, not just bounce rate.** Always include:
   - Accepted rate, hard bounce %, soft bounce %, block bounce %, admin bounce %
   - Spam complaint rate (with the caveat in §6 below)
   - Real (non-prefetched) open rate, CTR
   - Volume (must be ≥10K over the window to draw any conclusions; <1K = noise; 1K–10K = directional only)

5. **For single-domain mode, also run DNS checks.** Order matters:
   1. `GET /api/v1/sending-domains/<domain>` for the SparkPost-configured DKIM selector + auth status. **This is ground truth — do not guess DKIM selectors by digging a fixed list.** SparkPost's `scph<MMDD>` selectors are minted per-sending-domain at provisioning; the actual value is in the API response.
   2. `dig TXT <selector>._domainkey.<domain> +short` to confirm DNS publication of the selector the API named.
   3. `dig TXT <domain> +short | grep spf` — SPF at apex. Note: `spf_status: invalid` from the SparkPost API is **misleading for Valimail-fronted customers** (apex SPF uses `_spf.vali.email` macros) — SparkPost can't evaluate the dynamic macro. Real-world delivery is fine if `dkim_status: valid` and accept rate is healthy.
   4. `dig TXT _dmarc.<domain> +short` — DMARC posture. `p=reject` is strict; alignment must be carried by DKIM if SPF goes through SparkPost's own bounce subdomain.
   5. `dig CNAME <your-bounce-subdomain>.<domain> +short` — does the customer's domain have a bounce CNAME pointing to `sparkpostmail.com`? **Absence is NOT automatically a problem.** Many customers (especially Valimail-fronted ones) use SparkPost's default return-path (`is_default_bounce_domain: false`), and bounces still reach the platform via SparkPost's events stream.
   6. Detect ESPs commingled at apex by checking known selectors from other ESPs (SendGrid `s1/s2/em*`, Mailchimp `k1/k2/k3`, Google Workspace `google`, O365 `selector1/selector2`, HubSpot `hs1-X/hs2-X`, Constant Contact `ctct1/ctct2`). Two or more non-SparkPost ESP selectors at apex = commingled reputation = anti-pattern.

6. **Label every conclusion as fact or hypothesis.**
   - **Fact** = direct from the data (e.g. "Hard bounce rate is 4.2%")
   - **Hypothesis** = your inference (e.g. "Likely caused by a dormant list because the dominant bounce classification is code 22 Mailbox Full")
   - Mark them inline: `[FACT]` / `[HYPOTHESIS]`

7. **Spam complaints caveat.** Hotmail/Outlook complaint volumes can be inflated by their "this is junk" toolbar (a single human click + machine-learning amplification). Gmail rarely surfaces complaint feedback at all. **Don't treat the SparkPost complaint count as ground truth.** Treat it as a directional signal and corroborate against bounce + open patterns.

8. **Provide ranges, not point estimates** when interpreting future state. E.g.:
   - "Recovery likely 60–90 days if engagement suppression starts this week; 90–120 if Hotmail block is severe."
   - "If the customer publishes the DKIM record today, sends should start succeeding within 1 hour of `dkim_status` flipping to `valid`."

9. **Explain what would shift the conclusion.** E.g.:
   - "If the Hard bounces are concentrated on dormants pulled from a pre-2024 list, suppression alone fixes it. If they're hitting current opt-ins, the signup flow is the problem."
   - "If the top recipient address dominates 30%+ of bounces, this isn't a list-quality problem — it's a corrupted contact record."

10. **End with two distinct sections:**
    - **Findings & follow-ups** — what we know (FACT), what we suspect (HYPOTHESIS), and what to investigate next
    - **Doc-drift suggestions** — if the investigation surfaces information that contradicts or extends an existing doc, flag the doc + section + rationale

11. **Offer to post a persistent readout** to your docs system for delta tracking across runs. Suggested convention:
    - **Title:** `<customer-apex-domain> — Deliverability` (domain-first, no product-name prefix)
    - **Page emoji** (page-tree icon): one of 🟢 / 🟡 / 🟠 / 🔴 / ⚪ per the status rubric below
    - **BLUF block** at top — 1–3 sentence executive summary with the headline number bolded, trajectory direction, and the leverage action
    - **Status pill row** under the title — `**Status:** <emoji> <name> | **Last reviewed:** YYYY-MM-DD | **Next review:** YYYY-MM-DD`
    - **One page per customer** — search for the existing page first by title. If found, prepend new dated section + update the Status Change Report table at top + rewrite the BLUF + update the page emoji if status changed. If not, create new page under your "Domain Analysis Reports" parent with the exact title + emoji + labels (suggested: `deliverability` + `cs-readout`).

    **Status emoji rubric:**
    - 🟢 **Healthy** — Accept ≥ 95%; all per-category rates below purgatory thresholds; quarterly review
    - 🟡 **Watch — recovering or borderline** — Accept 90–95% OR one rate in purgatory range OR at threshold OR recovering from prior BAD; weekly review
    - 🟠 **At-risk — stuck or trending wrong** — Multiple rates in purgatory range OR one rate above BAD threshold but improving OR one ISP >50% block; weekly + customer outreach
    - 🔴 **Critical** — BAD pool with no recovery / worsening / active customer complaint / Spamhaus listing / Block storm in progress; daily + immediate outreach
    - ⚪ **No read** — Volume < 10K in 30d window

    Tie-breaking: pick the less rosy of two candidates; explain the split in the BLUF.

    **Ask the user first.** After producing the brief in chat, end with: "Want me to publish this as a readout under `<your-readouts-parent>`? (status: <emoji>) (yes / no)" Do NOT auto-publish. Wait for explicit confirmation. If they say yes, write the page and return the URL. If they say no or don't respond, skip — the brief in chat stands on its own.

    **Why we ask:** investigations are sometimes exploratory ("what do the numbers look like?") and shouldn't pollute the persistent record. The persistent page is for investigations the user wants kept for delta-tracking later. Make it the user's explicit choice.

## Critical pitfall: same-day block counts mislead

SparkPost block events for bulk sends **arrive over days-to-weeks, not on send day**. A campaign showing 0.5% block rate on the day-of can roll up to 20%+ once delayed events arrive. When interpreting a customer's recent campaign performance:

- ❌ **Don't use same-day block-event counts** as evidence of "their list is clean now"
- ✅ **Use the rolling 30-day rate** (what tier-routing logic typically evaluates)
- ✅ **Wait 7+ days after a bulk send** before drawing conclusions about that campaign's quality

This is the single most common interpretation error in deliverability investigations.

## Pool-tier rotation (if your platform has one)

Many SaaS senders build a tier-routing layer over SparkPost's IP pools: customers with healthy 30-day metrics route through "good" pools (warm, reputation-built); customers with elevated bounces route through "purgatory" / "shared" / "bad" pools to protect the rest of the cohort. If your platform has such a layer:

- A customer's *current* pool placement reflects the most recent tier-cron run, which usually evaluates a rolling 30-day bounce signal
- Auto-promotion typically happens once all per-category rates drop below thresholds (commonly Hard <1%, Block <2%, Soft <5%). Expect ~4 weeks in practice because the 30-day window keeps prior high-bounce campaigns in the average.
- Don't tell a customer "the cron will auto-promote you in 30 days" if the cron is disabled or you haven't confirmed it ran recently. Check your platform's cron-status docs first.
- The SparkPost-side `ip_pool` field on a message reflects what was *requested* at injection. Whether SparkPost actually delivered through that pool depends on pool capacity and dispatch logic.

## Output formats

Ask the user which they want before producing it:

- **Markdown brief** (default for single-domain / single-pool) — paste-friendly for your docs system or a Slack thread
- **Excel workbook** (`.xlsx`) — only for multi-domain investigations; useful when comparing 10+ domains side-by-side

## What this skill is NOT
- A verdict machine. Every conclusion is fact-vs-hypothesis labelled. Recovery times are ranges.
- A customer-facing report generator. CSM/AM rewrites the brief in customer-safe language before sending; the skill produces internal-language analysis.
- An incident-response tool. Active outages go to your incident runbook.
- An autonomous doc editor. Doc-drift suggestions are recommendations; the human decides whether to update.

## Chat protocol

1. Greet briefly. Confirm mode + target + time window if not clear. On first session use, confirm the logging tool (Datadog default).
2. Run the protocol (steps 1–11 above). Don't dump raw API output into chat — summarize.
3. Produce the brief in the requested format.
4. Offer one or two specific follow-up investigations the user could run next.

Be tight. Anyone asking a deliverability question is usually under time pressure to respond to a client.
