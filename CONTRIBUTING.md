# Contributing

PRs welcome. This skill is opinionated — the opinions come from running SparkPost in production at SaaS-scale on both sides of the API — but the surface keeps evolving and feedback from operators with different setups is genuinely useful.

## What's in scope

- **Corrections to factual claims** in the skill (API paths, filter behavior, defaults, retention windows, etc.). If SparkPost changes something, please flag it.
- **New diagnostic patterns** you've found durable across multiple investigations. The bar is "I've used this pattern more than once and it held up" — not "this might be useful someday."
- **Adapter notes for non-Datadog logging tools** (Splunk, CloudWatch, GCP Logging, Honeycomb, Loki). Each adapter is short.
- **Adapter notes for non-Confluence docs systems** (Google Docs, Notion, GitHub wiki). Same idea.
- **Sanitization gaps** — if you spot a customer name, internal page ID, or vendor-specific phrase that slipped through, open an issue or PR.
- **Typo / clarity fixes** in the skill prose.

## What's out of scope

- Wholesale rewrites of the investigation protocol. The current order (ground-truth domain config → metrics → secondary slices → DNS → reputation probes when triggered) reflects a specific opinion about how to avoid the common interpretation errors. Propose changes here as an issue first so we can discuss the rationale before code.
- Adding non-SparkPost ESP support inside this skill. A separate skill (`sendgrid-deliverability`, `postmark-deliverability`, etc.) is the right shape for that. Happy to link related skills from this README.
- Customer-facing report templates. Internal-language briefs only; the human handles customer-facing rewrites.

## How to propose a change

1. **Small fixes** (typos, broken links, wrong API path) — open a PR directly against `main`. No issue needed.
2. **New diagnostic patterns / new sections** — open an issue first describing the pattern and what triggers it. We'll discuss whether it belongs in the core skill or in a reference file before you spend time on the PR.
3. **Adapter for a new logging or docs tool** — PR directly. Keep it under ~40 lines if possible; the skill is meant to be read end-to-end.

## PR mechanics

- Fork → branch off `main` → PR back to `main`.
- One change per PR. Easier to discuss and merge.
- Keep the prose tight. The skill is read top-to-bottom by Claude on every invocation; verbosity costs tokens and dilutes the load-bearing parts.
- If you're changing the SKILL.md investigation protocol, also update the matching section in `README.md` if the public-facing description drifts.
- I aim to review within a few days. If a PR sits longer than a week, ping me on the PR thread.

## Style notes

- Use `[FACT]` and `[HYPOTHESIS]` consistently when adding investigative guidance.
- Ranges, not point estimates, for any "recovery time" or "expected metric" language.
- Cite SparkPost docs inline with the URL when documenting API behavior — these things change, and the citation lets a future reader re-verify.
- No customer names. No internal page IDs. No employee names. The skill is generic.

## Reporting security issues

If you find something that looks like a real security issue (e.g. the skill leaking credentials, an injection-attack vector via the customization interview), please email rather than opening a public issue. My contact is in my GitHub profile.

## License

By contributing, you agree your contributions will be licensed under the project's [MIT License](./LICENSE).
