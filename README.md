# M4C Email Ramp-Up Engine

**What it is:** A self-governing bulk political email/SMS sending system with budget-aware throttling and automatic pause/resume.
**Why it existed:** A shoestring-budget campaign needed to reach 100K+ people before a 12-day deadline, without a dedicated ops team or a five-figure managed sending tool.
**What I owned:** The full architecture — shared-budget ramp, reputation-based auto-pause, delivery verification, and a self-hosted email validator built specifically to avoid uploading voter data to a third party.
**Stack:** PostgreSQL, n8n, Listmonk, AWS SES.
**Production status:** Deployed and exercised against real sends at scale (tens of thousands of recipients).

---

Documentation for a self-hosted, abuse-resistant bulk political email sending
system, built for the Friends of Melissa Chaudhry for Congress campaign to
reach a 109,000+ person voter list ahead of a 12-day ballot deadline.

This repo is **documentation of a working n8n automation**, not an exported
workflow — the implementation lives in n8n (a visual workflow tool) wired to
PostgreSQL, Listmonk (self-hosted email), and AWS SES. No credentials, voter
data, or infrastructure hostnames are included.

## Why self-hosted

Third-party email-validation services (ZeroBounce, NeverBounce, BriteVerify,
etc.) all require uploading the full voter list to a third-party cloud
service — a direct conflict with keeping sensitive campaign/voter data
on-server only. A self-hosted syntax + MX + SMTP-probe validator was built
instead, with zero data ever leaving the campaign's own infrastructure.

## Architecture

**Shared daily budget, not per-list caps.** A single ramp schedule governs
total daily send volume across whatever content is queued, growing
geometrically (`starting_batch_size` × `daily_growth_multiplier` per day,
with a safety buffer applied once per calendar day) rather than a flat rate —
chosen because the fixed deadline meant a slower start would compress the
necessary peak volume into less remaining time if anything ever paused.

**Auto-pause on real delivery signals, not estimates.** Every run checks
trailing 48-hour bounce and complaint rates against Listmonk's own data
before sending anything. If either threshold is breached, the ramp pauses
itself, alerts the team, and waits for a human — it does not silently keep
sending into a reputation problem.

**Queue, not a single checkbox.** Campaign lists are queued in Airtable with
an explicit order; the workflow always resolves to exactly one active list
at a time, finishes it, marks it complete, and only then picks up the next
one on its next scheduled run — deliberately not same-run chaining, so each
execution does exactly one understandable thing.

**Delivery is verified, not assumed.** A send is only logged as delivered
after polling the actual campaign status and confirming `sent == to_send` —
early versions of systems like this often log success the moment the send
*API call* returns, which does not mean the email actually left the
building. This system treats that gap as a correctness bug, not an edge
case: a batch that doesn't fully deliver within a bounded retry window is
left unmarked and retried, never silently counted as reached.

## Email validation pipeline (self-hosted)

Two tiers, run entirely on campaign-owned infrastructure:

1. **Syntax + MX** — fast, zero network risk beyond DNS lookups.
2. **SMTP probe** — connects to each domain's real mail server and checks
   mailbox existence without sending. Empirically, this is highly reliable
   against Gmail (consistent hard-rejects for nonexistent mailboxes, ~46% of
   a typical voter-file list), unreliable against Microsoft domains, and
   gives no signal at all against Yahoo (which refuses the probe connection
   outright) — worth knowing before assuming SMTP validation covers a list
   evenly.

Confirmed-invalid addresses are blocklisted at the platform level (not
per-list), so the protection applies automatically to any future list or
campaign without additional wiring.

## Real failure modes found and fixed building this

- **Zero-row branches silently drop in workflow automation tools** unless
  explicitly configured to still emit output — a "no bounces in the last
  48h" result (the *good* case) was, without that fix, silently killing the
  entire rest of the run.
- **A config fix on disk doesn't apply to an already-running process** — a
  long-running sync/send job with the old code loaded in memory can keep
  reproducing an already-fixed bug for hours until it's explicitly
  restarted.
- **A platform's own "success" response can lie about actual delivery** —
  see the delivery-verification section above. This was found by cross-
  checking server-side logs against what the automation had been reporting,
  not by trusting a clean run.
- **A full read-modify-write on a settings object that contains masked
  secret fields (e.g. password values returned as bullet-placeholder
  strings) can silently overwrite the real secret with the placeholder** —
  caused a same-day SMTP outage. Fix: never round-trip a masked value back
  into a write; always substitute the real known value or avoid touching
  that key.

## Status

Deployed and exercised against real sends (tens of thousands of recipients).
Deliberately built and kept inactive until a human explicitly decided to
flip the first production send live — this kind of system should never
auto-activate itself as a side effect of being built.
