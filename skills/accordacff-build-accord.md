---
name: accordacff-build-accord
description: Create an Accord and populate it with stages, steps, stakeholders and resources through the Accord GraphQL API, including how to undo a mistake.
api: Accord Developer API (GraphQL)
endpoint: https://api2.inaccord.com/graphql
operations:
  - createAccord
  - autoCreateAccord
  - createStage
  - createStep
  - createAccordMember
  - createAccordContact
  - createResource
  - updateAccord
  - deleteAccord
  - recoverAccord
generated: '2026-09-06'
method: generated
source: https://developers.inaccord.com/reference
---

# Build an Accord

Writing to Accord is a sequence of mutations against `https://api2.inaccord.com/graphql`, authenticated with
`Authorization: Bearer YOUR_API_KEY`. Every operation named here exists in Accord's published API reference; look
each one up at `https://developers.inaccord.com/reference/operations/mutations/<kebab-name>` for its exact input
type before you send it — Accord publishes no SDL, so the input shapes are only readable there.

## Order of work

1. **Create the Accord.** `createAccord` for a plain create, or `autoCreateAccord` when you want Accord to
   populate it from a playbook/template rather than building the plan yourself. Keep the returned `id`
   (a prefixed string, e.g. `ac_01H…`).
2. **Add the plan.** `createStage` per phase, then `createStep` per action inside each stage. Both carry an
   `order` field — set it explicitly rather than relying on insertion order.
3. **Add the people.** `createAccordContact` records the buyer-side contact; `createAccordMember` grants
   membership of the Accord and carries `designation`, `contactRole`, `internal` and
   `shouldReceiveNotifications`.
4. **Attach content.** `createResource`, then bind resources to steps.
5. **Amend rather than recreate.** `updateAccord` (and the matching `update*` mutation for each child type)
   changes a row in place. There are 182 `update*` and 52 `upsert*` mutations in the reference; an `upsert*`
   keyed on a natural key is the closest thing to a safe re-run Accord offers.

## Before you write: two things Accord does not give you

**There is no idempotency mechanism.** No `Idempotency-Key` header, no client-supplied request id, no documented
replay protection anywhere in Accord's developer documentation. If a `create*` mutation times out you cannot
safely resend it — you may create a second row. Handle it yourself:

- Prefer an `upsert*` mutation where one exists for the type you are writing.
- Otherwise, on a timeout, **query first** to find out whether the row landed, then decide. `23505`
  (unique violation) coming back on a retry is the signal that your first attempt did succeed.
- Do not blind-retry mutations on GraphQL `errors[]`. Accord's own guidance is to retry only transient HTTP
  `5xx`, with exponential backoff.

**There is no dry run.** No mutation documents a preview or validate-only mode. Write to a non-production
workspace if you need to rehearse.

## Undoing a write

Deletes are soft and reversible. Every core type has a `delete*` mutation paired with a `recover*` mutation —
`deleteAccord` / `recoverAccord`, `deleteStage` / `recoverStage`, `deleteStep` / `recoverStep`,
`deleteAccordMember` / `recoverAccordMember`, `deleteResource` / `recoverResource`, `deleteComment` /
`recoverComment`, and 19 more. Objects carry `archivedAt` so you can tell an archived row from a live one.

**Accord does not publish how long recovery stays available.** There is no stated retention window anywhere in
its documentation. Treat recovery as best-effort and act promptly; do not tell a user a deletion is reversible
for any particular number of days, because Accord has not said so.

There is no undo for `update*` — the previous values are not exposed. Read the row before you change it if you
need to be able to put it back. The one exception is library resource versions, which have
`revertLibraryResourceVersion`.

## Errors you will actually hit

| `extensions.code` | What it means | What to do |
|---|---|---|
| `42501` | Permission denied on the relation | The key's workspace/row-level-security role cannot write this row |
| `23505` | Unique violation | The row already exists — look it up, or use the `upsert*` variant |
| `23503` | Foreign-key violation | A referenced id is missing or was deleted; confirm the parent first |
| `23502` | Not-null violation | A required input field is missing |
| `22P02` | Invalid input syntax | Malformed UUID, enum value or date |
| `P0001` | Application error | A business rule rejected it; read `message`, do not retry |

HTTP-level: `400` malformed request or missing query · `401` missing or invalid key · `403` unauthorized key ·
`5xx` retry with backoff. Everything else, including all of the above codes, arrives as HTTP `200`.
