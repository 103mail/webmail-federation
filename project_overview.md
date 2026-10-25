---
name: webmail-federation project overview
description: What the webmail-federation repo is for and how it relates to its sibling repos (Go server, React client, e2e stories, PHP legacy)
type: project
---

webmail-federation holds the protocol and design for federating independently
operated Webmail Domains, so domains like 103mail.com and example.com can share
one client bundle while still exchanging mail. It contains the RFC
(`project_rfc.md`, normative for anything that crosses a Domain boundary), the
design and migration narrative (`project_design.md`) and the plan with current
status (`project_plan.md`). Status as of 2026-10-25.

**Why:** 103mail.com production still runs on the PHP monolith, which is
single-domain and will never federate. The Go backend replaces it and
implements the Basic tier of the RFC, so 103mail.com can federate once it is
cut over, and later become the first Registrar.

**How to apply:** When the user asks about federation, multi-domain, or "103mail
and other domains", think across the repos under `~/projects/webmail/`, not
this one in isolation:

- `webmail-server`: Go backend (ADR 2 supersedes PHP). PostgreSQL in
  production, SQLite under test (ADR 19). Implements the Basic tier (ADR 17)
  and advertises `registrar: false`. Federation code is in
  `internal/federation`, the mail side of it in `internal/email`, and the
  chat side in `internal/chat` (RFC §14, ADR 23). The OpenAPI
  spec of record is in `docs/openapi/v1/`, tested against the routes.
- `webmail-client`: React/MUI SPA, one bundle for all Domains. Real API client,
  per-Domain `/config.json` and `DomainContext`, chat over WebSocket. No
  federation-specific UI yet.
- `webmail-e2e`: end-to-end stories per milestone, run with godog against
  several real Domains through the public API only. A milestone is done when
  its stories are green.
- `webmail-legacy`: the PHP monolith running 103mail.com today. Reference for
  behaviour and schema only; must not be extended.
- `webmail-website`: Hugo marketing site for 103mail.com.
- `webmail-deploy`: the local compose stack of federated Domains, moved out
  of `webmail-server/deploy`.
- Tools outside this folder: `../dbml-tools` (schema in DBML, SQL and
  migrations) and `ws2api` (WebSocket proxy for chat).

**Key architectural facts to remember:**

- No message content ever travels by SMTP (ADR 4). A message is written once as
  canonical `g_thread`/`g_message` rows with per-user projections (ADR 3);
  outbound email carries only metadata and a secure link, via Amazon SES (ADR
  15/16).
- Between Domains, delivery is a signed HTTPS envelope: discovery at
  `webmail.<domain>/.well-known/webmail-federation`, TOFU-pinned Ed25519 keys,
  an outbox worker with a 72-hour retry budget and local bounces, and a
  `federation_id` (UUIDv7) on every thread and message.
- Each Domain speaks only for its own addresses. Unknown addresses on a local
  Domain are refused, not auto-onboarded (the old `{7digit}@103mail.com` path
  is gone in Go), and a non-Registrar answers `400 not_registrar` for
  recipients on any other domain. Peer-refused recipients are bounced and the
  thread is told about them. Only members write to a thread: a peer cannot add
  someone by listing them, and gets `425 not_a_member` (RFC §5.6).
- Every non-Registrar Domain must have a Default Registrar (RFC §8.2). Reaching
  people without a federated home through a Registrar is specified but not
  built yet.
- `webmail-deploy` runs four Domains locally (103mail.com,
  example.com, mail.acme.test and its unrelated parent acme.test), each with
  its own database, API process and hostname, plus the scripted test peer
  peer.test, a fault proxy and a clock the stories move forward. Federation
  works there, and M1 stories 01-09 (inbound rate limiting included) and the
  M7 chat stories are green; nothing federates in staging or production yet.
- Chat and calendar federate as optional extensions on protocol version
  `0.3`, each advertised by its own descriptor field: chat in RFC §14
  (`chat`, `chat_read` envelopes) and calendar in RFC §15 (`meeting`,
  `meeting_rsvp`). Decided 2026-10-25: calendar is not iCalendar, iTIP or
  iMIP. A meeting is sent whole with a rising `revision`, only by the
  organizer's Domain, which alone invites and removes attendees; each
  attendee's own Domain sends their answer to every attendee's Domain;
  recurrence is JSON modelled on RRULE. Nothing about a meeting goes by
  email, so External addresses cannot be invited. Both are implemented in
  `webmail-server` (ADR 23 for chat, ADR 22 for calendar); the M7 chat
  stories are green, and the calendar stories are still to be written.
