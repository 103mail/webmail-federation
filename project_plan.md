# Webmail Federation: Project Plan

Status as of 2026-10-21. Covers the six repos under `~/projects/webmail/`. The
RFC (`project_rfc.md`) is the normative spec for anything that crosses a Domain
boundary, and the design doc (`project_design.md`) is the migration narrative.
This file only adds the order of work and where each part currently stands.

## 1. Where things stand

| Repo                 | Role                                         | State                                                                                                                                                                                                                                                                                                                                                                 |
| -------------------- | -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `webmail-federation` | RFC v0.2 + design + overview                 | RFC complete for the Basic and Registrar tiers. Last change (2026-04-25) added §4.1.1 subdomain Domains, which is missing from the Appendix C change log. `project_overview.md` rewritten 2026-10-21.                                                                                                                                                                 |
| `webmail-server`     | Go backend, replaces the PHP side            | Implements the **Basic tier** (ADR 17): discovery + TOFU, Ed25519 signing, envelopes, inline attachments, outbox worker with 72h retry and local bounces, `federation_id` on `g_*` rows. Advertises `registrar: false`. Compose stack in `deploy/` federates four Domains locally (103mail.com, example.com, mail.acme.test, acme.test). Since 2026-10-20: PostgreSQL in production and SQLite under test (ADR 19), unknown addresses on own Domains refused instead of onboarded, `not_registrar` for recipients on other domains, peer-refused recipients bounced, one message order on every Domain, read-only controls for the e2e suite (ADR 20). `CLAUDE.md` updated 2026-10-21.                          |
| `webmail-client`     | React/MUI SPA, one bundle for all Domains    | Real API client, `/config.json` + `DomainContext` per Domain, chat over WebSocket. No federation-specific UI yet (remote recipient marking, bounce rendering).                                                                                                                                                                                                        |
| `webmail-legacy`     | PHP monolith, current 103mail.com production | Reference for behaviour and schema only (ADR 2). Single-domain, `{7digit}@103mail.com` auto-onboarding. Must not be extended.                                                                                                                                                                                                                                         |
| `webmail-website`    | Hugo marketing site for 103mail.com          | Says nothing about federation, own domains, or operators.                                                                                                                                                                                                                                                                                                             |
| `webmail-e2e`        | End-to-end stories per milestone (godog)     | Stories for M1-M6 written. M1 story 01 green, story 05 green except its outage scenario; the rest `@wip`.                                                                                                                                                                                                                                                             |

Design migration steps (`project_design.md`, "Migration steps") mapped to
reality:

| Step  | Topic                                                    | State                                               |
| ----- | -------------------------------------------------------- | --------------------------------------------------- |
| 1     | Client `/config.json`, no hardcoded domain               | Done                                                |
| 2-5   | Domain config, discovery, fan-out split, outbox, inbound | Done in Go (not in PHP, by design)                  |
| 6     | Two-domain round-trip                                    | Done in local compose stack (four Domains), not in staging  |
| 7     | Harden: rate limits, caps, bounces                       | Partly: caps and bounces done, rate limiting not    |
| 8-10  | Real-email addressing on all Domains                     | Not started                                         |
| 11-14 | Registrar: shadows, claim, `claim-by-real-email`         | Not started (schema has `is_registrar` column only) |
| 15-16 | Multi-registrar lookup and consolidation                 | Not started                                         |

Known gaps recorded in ADR 17:

- ~~`g_subscription` lacks a unique `(g_thread_id, email)` index~~ Done
  (2026-10-20, index `thread_email`). Production data still has duplicates to
  clean up during the M2 migration.
- No inbound rate limiting (RFC §9.10).
- Own-key rotation is a flag day: we follow peers' `previous_public_keys` but
  cannot publish our own.
- `in_reply_to` points at the thread, not the message, so branching replies are
  not expressible.
- Chat and calendar are not federated at all; the RFC only covers mail.

## 2. Goal and milestones

**Goal:** 103mail.com runs on the Go backend in production as a conformant Basic
Webmail-Federation Domain, then becomes the first Registrar, so that any
independently operated Domain can exchange mail with it and reach non-federated
recipients through it.

| Milestone                       | Outcome                                                          | Exit criterion                                                                                                       |
| ------------------------------- | ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| M0 Docs in sync                 | Every repo describes what is actually built                      | Overview, server `CLAUDE.md`, RFC change log updated; RFC tagged 0.3                                                 |
| M1 Basic tier, production grade | ADR 17 gaps closed, conformance suite green                      | Appendix B vectors pass, interop suite passes against a second independent stack                                     |
| M2 103mail.com on Go            | PHP retired for 103mail.com                                      | Production traffic served by `webmail-server`; legacy read-only                                                      |
| M3 Federation live              | 103mail.com federates with at least one external Domain          | `webmail.103mail.com/.well-known/webmail-federation` public; real round-trip with a peer in staging, then production |
| M4 Real-email addressing        | Every user has a verified `real_email`                           | Signup double opt-in; notifications go to `real_email`                                                               |
| M5 103mail.com as Registrar     | Non-federated recipients get shadow mailboxes and can claim them | RFC §8 complete; advertises `registrar: true`                                                                        |
| M6 Multi-registrar              | Two registrars converge on one mailbox                           | Design step 16 staging scenarios (a), (b), (c) pass                                                                  |
| M7 Beyond mail (optional)       | Decision and, if yes, spec for chat/calendar federation          | RFC extension draft or a written "no"                                                                                |

## 3. Work breakdown

Sizes are rough: S under a week, M one to three weeks, L more.

### M0: Docs in sync (S)

- Rewrite `project_overview.md` to describe the current trio: Go server, React
  client, this repo. Done (2026-10-21).
- Add §4.1.1 to RFC Appendix C as 0.3 and bump the version header. Still open:
  Appendix C now has an "Unreleased" section (§5.5, §5.6, `thread.members`,
  `thread.refused`, `not_registrar`, Default Registrar), but §4.1.1 is not in
  it and the header still says 0.2.
- Update `webmail-server/CLAUDE.md` (no longer greenfield, client no longer on
  mocks, sibling path is `../webmail-legacy`). Done (2026-10-21).
- Point `project_design.md` "Context" at the Go implementation and mark steps
  1-6 as done.

### M1: Basic tier, production grade (M)

Server:

- Clean up duplicate `g_subscription` rows, then add the unique index (migration
  via `dbml-tools migrate`). Partly done: the index is in the schema
  (2026-10-20); cleaning production duplicates moves to the M2 migration.
- Inbound rate limiting per peer Domain, answering `429` with `Retry-After` (RFC
  §9.10).
- Publish `previous_public_keys` for our own Domains: support a list of keys per
  Domain in `federation.json`, sign with the newest, keep old ones in the
  descriptor for the 48h window (RFC §9.5). Write a runbook for rotation.
- Message-level `in_reply_to` (carry the replied-to message's `federation_id`),
  keeping thread correlation as fallback.
- Subdomain Domains (RFC §4.1.1): add tests for `mail.example.com`-style
  Domains. Decide whether refusing a Domain literally named `webmail.*`
  (`internal/federation/federation.go:151`) is intended, and say so in the RFC
  if it is. Partly done: `mail.acme.test` is a third Domain in the compose
  stack with unit tests (2026-10-20); e2e story 09 is still `@wip` and the
  `webmail.*` question is open.
- `accept_from: any | allowlist` is already `allowedPeers`; document it as the
  private-deployment knob.

Conformance and interop:

- Turn RFC Appendix B into a Go test that runs the published vectors byte for
  byte.
- Scripted interop suite on the compose stack: send, reply, attachment, bounce
  on unknown recipient, peer down then up (retry), key rotation on one side,
  replayed envelope (dedup), stale `Date`. Partly done: `webmail-e2e` holds the
  stories and a godog harness (2026-10-20); story 01 (project thread) is green,
  story 05 (mixed delivery and bounces) is green except the outage scenario.
- Observability: outbox depth and age per `peer_domain`, inbound accept/reject
  counts by error code, bounce count. Alert on outbox age over a few hours.

### M2: 103mail.com on Go (L)

This is the precondition for federation in production, since PHP will never
federate.

- Finish the parity check from ADR 14 (byte-identical JSON on shared endpoints;
  calendar is the known exception).
- Data migration rehearsal on a production dump with `dbml-tools migrate`,
  including the `g_attendee.will_attend` type change.
- Cut over: Go API + worker, `webmail-client` bundle, ws2api for chat, SES/SNS
  for mail in and out (ADR 15/16).
- Keep PHP read-only for a rollback window, then retire.
- Remove the `{7digit}@103mail.com` auto-onboarding path for remote domains once
  traffic is on Go. Done in Go (2026-10-20): unknown addresses are refused, not
  onboarded, and recipients on other domains get `not_registrar`.

### M3: Federation live (M)

Ops:

- DNS and TLS for `webmail.103mail.com`; certificate must cover that exact name
  (RFC §9.2).
- Generate and store the production Ed25519 key (secret store, backed up,
  access-logged).
- Staging peer: a second Domain operated separately (e.g. `example.com` or a
  partner), on its own infrastructure, not the compose stack.
- Production enable, first with `allowedPeers` set to known partners, then open.

Client:

- Mark remote recipients in compose (design §9), so users see when a message
  leaves the Domain.
- Render local bounces clearly, with the reason from the outbox.
- Per-Domain branding from `/config.json` (logo, name), so one bundle serves
  every Domain cleanly.

Spec:

- Submit the `webmail-federation` well-known suffix to IANA (RFC §12) once the
  RFC is stable enough to publish.

### M4: Real-email addressing (M)

Design steps 8-10, on all Domains:

- Schema: `user.real_email`, `real_email_verified_at`, unique
  `(domain_id, real_email)`.
- Signup requires `real_email` with double opt-in; activation gated on
  verification. Backfill for existing 103mail users (prompt on next login).
- `internal/notify` sends to `real_email`.

### M5: 103mail.com as Registrar (L)

Design steps 11-14, RFC §8:

- Schema: `alias.real_email`, `forward_canonical`, `state`, `issued_at`,
  `expires_at`; shadow record (design §14.8).
- Non-registrar side: `default_registrar` (required; in the RFC §8.2 since
  2026-10-20, not yet in code) and `fallback_registrars` config; fan-out branch that federates non-federated recipients to the
  registrar (design §15.4). Boot fails without `default_registrar` on a
  non-registrar Domain.
- Registrar side: local match, peer match, tentative shadow + SMTP invite +
  claim page (design §15.5).
- `POST /api/v1/federation/claim-by-real-email` and the offer/claim/backfill
  protocol (design §14.4).
- Privacy review of `/registrar/lookup` and `/pending-shadows` (RFC §9.9) before
  advertising `registrar: true`.

### M6: Multi-registrar (M)

Design steps 15-16, RFC §8.3-8.6:

- `/api/v1/registrar/lookup`, `/api/v1/registrar/pending-shadows`, curated
  `registrar_peers` config.
- Cross-registrar query on lookup miss, consolidation on claim, 60s cache purge
  on peer removal.
- Staging with two registrars and one non-registrar; run the three scenarios
  from design step 16.

### M7: Beyond mail (decision first)

The seed data and TESTING.md already show the gap: a chat with a peer on the
other Domain has history but cannot go live. Decide whether chat and calendar
invites federate. If yes, draft an RFC extension (envelope kinds for chat
messages and meeting invites, presence excluded), and add an ADR in the server.
If no, record it as a non-goal in RFC §2.2 and hide cross-Domain chat in the
client.

### Website (S, runs alongside M3)

- Add a federation page to `webmailwebsite`: what it means for users (mail stays
  in Europe, on the Domains involved), and an operator section linking to the
  RFC.
- Business/Enterprise plans: offer "your own Domain" once multi-domain hosting
  is in place.

## 4. Dependencies

```
M0 ──> M1 ──> M3 ──> M5 ──> M6
        │      ^      ^
        └─> M2 ┘      │
               M4 ────┘
M7 is independent; decide any time after M3.
```

M1 and M2 can run in parallel. M4 can start once M2 is done, since it changes
signup on the production backend.

## 5. Risks

- **Cutover risk (M2)** outweighs everything federation-specific. Rehearse the
  migration on real data and keep a rollback window.
- **Silent mail loss** if outbox monitoring is missing. The bounce path exists;
  alerting on outbox age is what makes it visible to operators.
- **Spam over federation.** TOFU trusts any Domain with DNS and a certificate.
  Launch with an allowlist, add rate limiting (M1) before opening up.
- **Registrar privacy.** Lookup by real-email hash is an enumeration oracle if
  rate limits and peer curation are weak (RFC §9.9).
- **Spec drift.** The RFC, design doc and ADR 17 already disagree in one place
  (re-signing on retry). Every such resolution should land back in the RFC, not
  just in an ADR.
- **Single maintainer.** Interop is only meaningful against a stack someone else
  operates; find a second operator early.

## 6. Open decisions

- Who operates the first external peer Domain for M3?
- Does 103mail.com become the Default Registrar for the network, and under what
  terms for other operators?
- Chat and calendar federation: in scope or a recorded non-goal (M7)?
- Multi-domain hosting on one Server (design §13): needed for the
  Business/Enterprise plans before or after M5?
- Account portability between Domains stays "no" for v1 (design open questions);
  confirm.
