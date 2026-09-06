---
name: accordacff-load-accord
description: Load a single Accord (deal or onboarding workspace) from the Accord GraphQL API with its stages, steps, stakeholders, resources and summaries, in one request.
api: Accord Developer API (GraphQL)
endpoint: https://api2.inaccord.com/graphql
operations:
  - accord
  - accords
  - accordsConnection
  - stages
  - steps
  - accordMembers
  - resources
  - summaries
generated: '2026-09-06'
method: generated
source: https://developers.inaccord.com/quickstarts/accords
---

# Load an Accord

Accord has no REST API. Everything is one POST to `https://api2.inaccord.com/graphql`.

## Before you start

- You need a workspace API key. A workspace **admin** creates it in the Accord app under
  Settings → Workspace → API Keys, and it is shown only once.
- The key is workspace-scoped and obeys the same row-level security as an in-app session. If you get
  `extensions.code: "42501"` you are not blocked by a bug — the key's role cannot see that row.
- API access requires an Accord licence that includes it; Accord's pricing lists "API Support" under Enterprise.

## Steps

1. **Send the key on every request.**

   ```
   Authorization: Bearer YOUR_API_KEY
   Content-Type: application/json
   ```

2. **Hydrate one Accord.** This is Accord's own published quickstart shape — the `accord` query with nested
   `stages`, `steps`, `accordMembers`, `resources`, `summaries` and `accordDomains`. Order nested collections
   explicitly with `orderBy: PRIMARY_KEY_ASC`, and use `condition` to hide internal-only steps from a
   buyer-facing view.

   ```graphql
   query LoadAccord($accordId: String!) {
     accord(id: $accordId) {
       id
       workspaceId
       accountName
       opportunityName
       opportunityAmount
       closeDate
       isDraft
       isTemplate
       archivedAt
       createdAt
       updatedAt
       stages(orderBy: PRIMARY_KEY_ASC) {
         id
         title
         order
         description
         steps(condition: { internal: false }, orderBy: PRIMARY_KEY_ASC) {
           id
           title
           order
         }
       }
       accordMembers {
         id
         designation
         invitationStatus
         internal
         contactRole
         workspaceAccount {
           id
           contactEmail
           role
           profile { firstName lastName jobTitle }
         }
       }
       resources { id name }
       summaries { id }
       accordDomains { id domain }
     }
   }
   ```

   Variables: `{ "accordId": "ac_01H…" }`.

3. **List many Accords with a cursor, not an offset.** Every collection is published twice — a plain list field
   and a Relay connection field. Use the connection when you need to page.

   ```graphql
   query ListAccords($after: Cursor) {
     accordsConnection(first: 50, after: $after, orderBy: PRIMARY_KEY_ASC) {
       edges { cursor node { id accountName opportunityName closeDate archivedAt } }
       pageInfo { hasNextPage endCursor }
       totalCount
     }
   }
   ```

   Accord's own guidance is to prefer narrower filters and page sizes of 25–100 over pulling everything and
   filtering client-side.

4. **Filter server-side.** `condition` is exact-match and ANDs every field together. `filter` gives you
   `equalTo`, `notEqualTo`, `in`, `notIn`, `isNull`; `like`, `likeInsensitive`, `startsWith`, `endsWith`,
   `includes` on strings; `greaterThan`/`lessThan` families on numbers, dates and UUIDs; `contains`,
   `containedBy`, `overlaps` on lists — combined with `and`, `or`, `not`.

5. **Skip archived rows unless you want them.** Objects carry `archivedAt`; a non-null value means the row was
   soft-deleted and can be brought back with the matching `recover*` mutation.

## Reading the response

- Accord returns **HTTP 200 even for errors**. Always check both `data` and `errors` — a response can carry both.
- The stable code is `extensions.code`, a PostgreSQL SQLSTATE. Do not pattern-match on `message`; Accord says it
  can change.
- `42501` permission denied · `23505` unique violation · `23503` foreign key · `23502` not null ·
  `22P02` invalid UUID/enum/date · `P0001` business-rule rejection.
- Retry only on transient HTTP `5xx` with exponential backoff. GraphQL errors are mostly deterministic; retrying
  them will not help.

## Limits and unknowns

Accord publishes no rate limits and no rate-limit response headers, so back off on your own signal. It publishes
no idempotency mechanism either — that only matters for the write skill, not this one.
