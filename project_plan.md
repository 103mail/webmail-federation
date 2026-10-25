# Webmail Federation: Project Plan

Status as of 2026-10-25. Covers the seven repos under `~/projects/webmail/`. The
RFC (`project_rfc.md`) is the normative spec for anything that crosses a Domain
boundary, and the design doc (`project_design.md`) is the migration narrative.
This file only adds the order of work and where each part currently stands.

## 1. Where things stand

| Repo                 | Role                                         | State                                                                                                                                                                                                                                                                                                                                                                 |
| -------------------- | -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `webmail-federation` | RFC + design + plan + overview               | RFC revision 0.3, tagged `rfc-0.3` (protocol version 0.2 on the wire), complete for the Basic and Registrar tiers. Since 0.3: replies belong to the thread (§5.5), messages from non-members of a thread are refused with `425 not_a_member` (§5.6, §6.3), the chat extension (§14) defines protocol version `0.3`, and the calendar extension (§15) is drafted on it, with `kind` general to `0.3` and unknown kinds refused with `400 unsupported_kind` (§10). Overview and design are current. |
| `webmail-server`     | Go backend, replaces the PHP side            | Implements the **Basic tier** (ADR 17, hardened in ADR 21): discovery + TOFU, Ed25519 signing, envelopes, inline attachments, outbox worker with 72h retry, backfill once a peer is back, and local bounces carrying the peer's error code. Attachment limits on both sides, non-members refused, own-key rotation with a 48h window, Appendix B vector tested, inbound rate limiting per peer Domain. PostgreSQL in production, SQLite under test (ADR 19). Advertises `registrar: false`. The OpenAPI spec of record now lives here (`docs/openapi/v1/`, moved from the legacy repo and extended with the federation, chat-session, SES webhook and thread move/leave routes), and `openapi_test.go` fails when a served route is missing from it or the reverse. Implements the chat extension (RFC §14, ADR 23): advertises `chat: true` and versions `0.2` and `0.3`, chat and `chat_read` envelopes through the outbox on a 1-second loop of their own with a 1-hour budget, receiving, history for new members, read receipts, and bounces in the chat. Implements the calendar extension (RFC §15, ADR 22): advertises `calendar: true`, stores a meeting as a stack of revisions with answers per occurrence, sends `meeting` and `meeting_rsvp` envelopes through the mail outbox with a later revision superseding a queued one, receives both, and tells the organizer in their inbox who cannot be reached. Organizer-only changes through `PUT /calendar/meeting/{id}` and `DELETE /calendar/attendee/{id}`. |
| `webmail-client`     | React/MUI SPA, one bundle for all Domains    | Real API client, `/config.json` + `DomainContext` per Domain, chat over WebSocket. No federation-specific UI yet (remote recipient marking, bounce rendering).                                                                                                                                                                                                        |
| `webmail-legacy`     | PHP monolith, current 103mail.com production | Reference for behaviour and schema only (ADR 2). Single-domain, `{7digit}@103mail.com` auto-onboarding. Must not be extended. No longer holds the OpenAPI spec.                                                                                                                                                                                                       |
| `webmail-deploy`     | Local compose stack (moved out of `webmail-server/deploy`) | Four federated Domains plus the scripted peer `peer.test`, with a fault proxy, mail sink, per-story config overrides, a movable clock and read-only controls (ADR 20). `docker compose up` alone: a setup service makes certificates and keys, and each Domain is at `http://localhost:8001` to `8004`, so no `/etc/hosts` entries or trusted CA. Federation still runs over HTTPS between the real hostnames on the federation network. |
| `webmail-website`    | Hugo marketing site for 103mail.com          | Says nothing about federation, own domains, or operators.                                                                                                                                                                                                                                                                                                             |
| `webmail-e2e`        | End-to-end stories per milestone (godog)     | Stories for M1-M6 written, and M7 chat stories (`m7-beyond-mail/01-chat-across-domains.feature`, five stories for RFC §14) green. M1 stories 01-09 green on a virtual clock (about 4 minutes for the suite), the rate-limit scenarios of stories 04 and 08 included. The suite runs in a container on the stack's network, so no `/etc/hosts` entries are needed. Committed: it brings up its own stack (`stack/`, from four images, no host ports). Uncommitted work points it at `../webmail-deploy` instead; which of the two stays is open. |

Design migration steps (`project_design.md`, "Migration steps") mapped to
reality:

| Step  | Topic                                                    | State                                               |
| ----- | -------------------------------------------------------- | --------------------------------------------------- |
| 1     | Client `/config.json`, no hardcoded domain               | Done                                                |
| 2-5   | Domain config, discovery, fan-out split, outbox, inbound | Done in Go (not in PHP, by design)                  |
| 6     | Two-domain round-trip                                    | Done in the local compose stack (`webmail-deploy`, four Domains and peer.test), not in staging |
| 7     | Harden: rate limits, caps, bounces                       | Done: caps, bounces and inbound rate limiting       |
| 8-10  | Real-email addressing on all Domains                     | Not started                                         |
| 11-14 | Registrar: shadows, claim, `claim-by-real-email`         | Not started (schema has `is_registrar` column only) |
| 15-16 | Multi-registrar lookup and consolidation                 | Not started                                         |

Known gaps recorded in ADR 17:

- ~~`g_subscription` lacks a unique `(g_thread_id, email)` index~~ Done
  (2026-10-20, index `thread_email`). Production data still has duplicates to
  clean up during the M2 migration.
- ~~No inbound rate limiting (RFC §9.10)~~ Done (2026-10-21): per peer Domain,
  per minute and per day, answered with `429` and `Retry-After`.
- ~~Own-key rotation is a flag day~~ Done (2026-10-21, ADR 21): retired keys
  are published for 48 hours, with a runbook.
- `in_reply_to` points at the thread, not the message. Decided (2026-10-21):
  that is intended, not a gap. A reply belongs to the thread, which is read in
  one order (RFC §5.5); there are no branching replies.
- Chat and calendar are not federated yet. Decided (2026-10-25): both
  federate, as RFC §14 and §15 (M7).

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
| M7 Beyond mail                  | Chat and calendar federate between Domains                       | RFC §14 and §15 implemented; M7 chat and calendar stories green                                                      |

## 3. Work breakdown

Sizes are rough: S under a week, M one to three weeks, L more.

### M0: Docs in sync (S)

- Rewrite `project_overview.md` to describe the current trio: Go server, React
  client, this repo. Done (2026-10-21).
- Add §4.1.1 to RFC Appendix C as 0.3 and bump the version header. Done
  (2026-10-21): the document is revision 0.3, with §4.1.1 and everything since
  in its change log, and still specifies protocol version `0.2` on the wire,
  since every change is optional or a rule about behaviour (§10).
- Update `webmail-server/CLAUDE.md` (no longer greenfield, client no longer on
  mocks, sibling path is `../webmail-legacy`). Done (2026-10-21).
- Point `project_design.md` "Context" at the Go implementation and mark steps
  1-6 as done. Done (2026-10-21); step 7 is marked done since inbound rate
  limiting landed.

### M1: Basic tier, production grade (M)

Server:

- Clean up duplicate `g_subscription` rows, then add the unique index (migration
  via `dbml-tools migrate`). Partly done: the index is in the schema
  (2026-10-20); cleaning production duplicates moves to the M2 migration.
- Inbound rate limiting per peer Domain, answering `429` with `Retry-After` (RFC
  §9.10). Done (2026-10-21): a peer's accepted envelopes are counted over the
  last minute and the last day (120 and 20000 by default, `rateLimit` in
  `federation.json`), and past either limit it is answered `429 rate_limited`
  with `Retry-After` until the oldest falls out of the window. Replays are
  answered whatever the rate. Stories 04 and 08 cover both sides.
- Publish `previous_public_keys` for our own Domains: support a list of keys per
  Domain in `federation.json`, sign with the newest, keep old ones in the
  descriptor for the 48h window (RFC §9.5). Write a runbook for rotation. Done
  (2026-10-21): `previousKeys` with `retiredAt`, story 07, and
  `webmail-server/docs/runbooks/key-rotation.md`.
- Refuse a message on a thread from someone who is not a member of it, and
  bounce with the peer's error code; enforce attachment limits on both sides.
  Done (2026-10-21, ADR 21), proven by stories 03 and 06.
- Subdomain Domains (RFC §4.1.1): add tests for `mail.example.com`-style
  Domains. Decide whether refusing a Domain literally named `webmail.*`
  (`internal/federation/federation.go:151`) is intended, and say so in the RFC
  if it is. Partly done: `mail.acme.test` is a third Domain in the compose
  stack with unit tests (2026-10-20), and story 09 is green (2026-10-21). The
  `webmail.*` question is open.
- `accept_from: any | allowlist` is already `allowedPeers`; document it as the
  private-deployment knob. Proven by story 08 (2026-10-21); the operator
  documentation is still to write.

Conformance and interop:

- Turn RFC Appendix B into a Go test that runs the published vectors byte for
  byte. Done (2026-10-21, `TestAppendixBVector`, checked against OpenSSL).
- Scripted interop suite on the compose stack: send, reply, attachment, bounce
  on unknown recipient, peer down then up (retry), key rotation on one side,
  replayed envelope (dedup), stale `Date`. Done on the local stack
  (2026-10-21): M1 stories 01-09 are green, rate limits included. Running them against a second, independently operated stack is
  still open and belongs to M3.
- Observability: outbox depth and age per `peer_domain`, inbound accept/reject
  counts by error code, bounce count. Alert on outbox age over a few hours.

### M2: 103mail.com on Go (L)

This is the precondition for federation in production, since PHP will never
federate.

- Finish the parity check from ADR 14 (byte-identical JSON on shared endpoints;
  calendar is the known exception). The OpenAPI spec now lives in
  `webmail-server` and is tested against its routes (2026-10-21), so parity is
  checked against that spec rather than the copy in `webmail-legacy`.
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

### M7: Beyond mail (M for chat, L for calendar)

Decided (2026-10-25): chat and calendar both federate, as optional extensions
on protocol version `0.3`. Calendar does not follow iCalendar, iTIP or iMIP: a
meeting travels as an envelope between Domains, like chat and mail, never as an
invitation carried by mail.

Spec:

- Chat extension, RFC §14: direct and group chats, joining and leaving with
  history, read receipts, a shorter retry budget. Done (2026-10-25).
- Calendar extension, RFC §15. Drafted (2026-10-25): a `meeting` envelope
  carries the whole meeting with a `revision` that only goes up, sent only by
  the organizer's Domain; only the organizer invites and removes attendees; a
  `meeting_rsvp` from each attendee's own Domain reaches every attendee's
  Domain, for the whole meeting or one occurrence; recurrence is JSON
  modelled on RRULE, with overrides, and an unknown member in it is refused;
  documents travel with every revision. External addresses cannot be invited,
  since that would put the meeting in an email.

Server, chat:

- Implement RFC §14. Done (2026-10-25, ADR 23), unknown kinds answered
  `400 unsupported_kind` included.
- Tell the user at once when an added address is on a Domain without chat
  (§14.1); today they hear it from `mailer-daemon` when the first envelope is
  refused.
- Throttle read receipts to one per chat, reader and Destination every few
  seconds, and rate-limit chat envelopes apart from mail (§14.7, §14.8).

Server, calendar:

Done (2026-10-25), ADR 22 accepted:

- Store meetings as a stack of revisions (ADR 22):
  `g_meeting` as the meeting, append-only `g_meeting_revision`,
  `g_meeting_attendee` per revision, `g_meeting_rsvp` per answer and
  `g_meeting_document` per revision, nothing canonical edited or deleted,
  and deleting only in the `meeting` projection. Occurrences are expanded at
  read time from the §15.6 JSON, so `g_schedule` and `schedule` go. Add an
  update route that writes a revision.
- Make adding and removing attendees organizer-only (`AddAttendee` lets any
  attendee invite today), and stop recording an unreachable attendee without
  telling the organizer.
- Send `meeting` and `meeting_rsvp` through the outbox, receive both, and send
  existing answers to the Domains of newly invited attendees.
- Advertise `calendar: true` once all of §15 is in.

Client:

- Show who cannot be reached when a chat member or meeting attendee is on a
  Domain without the extension.
- Show every attendee's answer, per occurrence, and mark an answer given to an
  earlier revision.

End-to-end:

- Make the five M7 chat stories green and drop `@wip`. Done (2026-10-25).
- Write calendar stories: invite across three Domains, move and cancel a
  meeting, answers reaching every Domain, removing an attendee, a recurring
  meeting with one occurrence moved and one cancelled, and an attendee on a
  Domain without `calendar`.

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
M7 is independent of M2-M6 and can run alongside them.
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
- **Calendar schema rework (M7).** Stacking revisions (ADR 22) replaces the
  calendar tables and moves recurrence expansion to read time, and existing
  meetings have to be migrated into it, which lands in the same window as the
  M2 migration.
- **Single maintainer.** Interop is only meaningful against a stack someone else
  operates; find a second operator early.

## 6. Open decisions

- One local stack or two: should `webmail-e2e` keep its own image-only stack
  (runs beside a development stack) or use `webmail-deploy` (one place to
  maintain)? The working tree of `webmail-e2e` currently does the latter.

- Who operates the first external peer Domain for M3?
- Does 103mail.com become the Default Registrar for the network, and under what
  terms for other operators?
- ~~Chat and calendar federation: in scope or a recorded non-goal (M7)?~~
  Decided (2026-10-25): both in scope, RFC §14 and §15.
- Meetings have no location in the schema, so RFC §15 carries none. Add one
  to both?
- Multi-domain hosting on one Server (design §13): needed for the
  Business/Enterprise plans before or after M5?
- Account portability between Domains stays "no" for v1 (design open questions);
  confirm.
