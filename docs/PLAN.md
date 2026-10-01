# Tele Gal: MVP Build Plan

## Context
The seller runs Telegram "drops", which are batches of about 20 items posted to a **Telegram channel**. Buyers claim an item by commenting on its post in the linked discussion group. The comment text can be anything; what counts is the order. Today, finding out who claimed first, grouping items by buyer, billing and tracking payment are all done by hand. Tele Gal automates the whole loop: create a batch, spread the posts over a time window, collect comments as ordered claims, assign the first commenter as winner, build per-buyer bills, record payments by hand, close the batch (sold/unsold) and export to Excel.

The repo is empty (no commits), so this is a new build. Decisions so far: Next.js full-stack, S3/R2 from day one, and `CLAUDE.md` and `TODO.md` committed to the repo.

## Architecture
```
Next.js app (admin UI + API routes)  ──┐
                                        ├── PostgreSQL (raw SQL)
Bot worker (Node, grammY)  ────────────┤
                                        └── S3 / R2 (photos)
Bot worker ⇄ Telegram: posts to channel, listens to discussion group
```
- **One repo, two processes**: `apps/web` (Next.js 15, Tailwind) and `apps/bot` (grammY, long-polling). They share `packages/db` (raw SQL: the `postgres` driver from porsager with tagged-template queries, numbered `.sql` migration files and a ~40-line migration runner that tracks applied files in a `schema_migrations` table, plus hand-written TS row types and one query module per table) and `packages/core` (pure logic: scheduling, winners, billing text, Excel).
- pnpm workspaces. TypeScript everywhere.
- The bot worker runs the **scheduler loop**: every 30s it posts items whose `scheduled_at <= now` and `status = scheduled`. Because state lives in the database, a restart is safe.

## Telegram mechanics (channel and comments)
- The bot is an admin of the **channel** (to post) and of the **linked discussion group** (to read every message).
- Post per item: `sendMediaGroup` (up to 10 photos, caption on the first). Caption:
  `[B001]\nFloral Dress\n$20\nQty: 1\n\nComment to claim 💬`
- Telegram auto-forwards each channel post into the discussion group (`is_automatic_forward = true`, with `forward_origin.message_id` = the channel message id). The bot stores the **discussion message ids** on the item. For an album it stores every message in the media group.
- A comment is a group message whose `reply_to_message` (or thread root `message_thread_id`) matches a stored discussion message id. It becomes a **claim** with `telegram_user_id`, `username`, `first_name`, text, Telegram `date` and a monotonic `position`. Ordering uses Telegram message id, which is strictly increasing in the group, so it doesn't depend on timestamp ties.
- Ignored: comments from the seller or admins (configurable), and repeat claims by the same user on the same item. Only their first one counts.
- Users without a username are stored by id, and the UI shows a `tg://user?id=` link.
- **Constraint:** claims can only be captured while the bot is running. The Bot API has no history fetch. CLAUDE.md will document this.

## Data model (raw SQL, `packages/db/migrations/001_init.sql`)
snake_case tables. Postgres enums for statuses, `numeric(10,2)` for money, `bigint` for Telegram ids, `int[]` for message ids, and FKs with `on delete cascade` from item to photos and claims. Queries live in `packages/db/src/queries/*.ts` (e.g. `items.ts`, `claims.ts`, `orders.ts`). Multi-step writes (generate orders, close batch) use `sql.begin(...)` transactions.
- `Batch`: id, name, status (`draft|scheduled|posting|live|closed`), windowStart, windowEnd, createdAt, closedAt
- `Item`: id, batchId, code (`B001`, unique per batch), title, description, price (Decimal), qty, status (`draft|scheduled|posted|claimed|sold|unsold`), scheduledAt, postedAt, channelMessageIds Int[], discussionMessageIds Int[], winnerUserId?
- `ItemPhoto`: id, itemId, s3Key, url, sortOrder
- `Claim`: id, itemId, telegramUserId (BigInt), username?, firstName?, text, tgMessageId, claimedAt, position, `skipped` bool (lets the seller pass over a no-show and promote the next claimer)
- `TgUser`: telegramUserId PK, username, firstName, lastName
- `Order`: id, batchId, telegramUserId, status (`pending|invoiced|paid`), total, invoicedAt, paidAt, unique(batchId, telegramUserId)
- `OrderItem`: id, orderId, itemId, price, `paid` bool (the per-item checkbox the seller asked for)

Qty > 1: the first N non-skipped claimers win one unit each. Winners go in a computed `winners` view/helper rather than a single `winnerUserId`. That field stays for the qty = 1 fast path.

## Core flows
1. **Create batch** → add items (form: photos, title, price, description, qty). Photos are uploaded directly to S3/R2 via a presigned PUT URL (`/api/uploads/presign`). The key is `drops/{yyyy-mm}/{batchId}/{code}/{n}.jpg`. Item codes are auto-assigned in order (B001…).
2. **Schedule**: pick a window start and duration (default 60 min). `core/schedule.ts` spreads N items evenly with ±20% jitter, never fewer than 60s apart. The queue is shown for review and can be edited, then Confirm sets the items to `scheduled`.
3. **Post**: the bot loop posts, records message ids and sets the item to `posted`.
4. **Claims**: collected live. The Claims screen lists each item code, with claimers in order under it and the winner highlighted. Each claim has a "skip" button that promotes the next claimer.
5. **Orders (main screen)**: "Generate orders" groups winners by buyer into one `Order` per buyer (idempotent regeneration keeps the `paid` flags). Each buyer card shows the items, total, a **Copy invoice** button (filled from a template with PayNow number from Settings) and an optional **Send via bot DM**, which only works if the buyer has /started the bot. Each item has a paid checkbox plus a "Mark all paid" button.
6. **Close batch**: items with a winner become `sold` and items without one become `unsold`. The batch is locked. Close is still allowed with unpaid orders, after a confirmation that shows the count.
7. **Excel export** (`exceljs`, `GET /api/batches/:id/export`) → `Drop_<name>.xlsx` with these sheets:
   - **Items**: code, title, price, qty, status, winner
   - **Claims**: code, then the ordered list of claimers under each code, matching the requested layout
   - **Buyers**: buyer → their items, prices, total, paid (the billing view)
   - **Payments**: buyer, total, paid amount, status
   - **Unsold**

## Admin UI (Next.js App Router)
`/batches` list → `/batches/[id]` with tabs: **Items · Schedule · Claims · Orders · Export**, plus `/settings` (PayNow number, invoice template, channel/group ids). Auth for the MVP is a single admin password in env with a signed cookie, enforced by middleware.

## Repo layout
```
apps/web/        Next.js admin + API routes
apps/bot/        grammY worker: scheduler loop + comment listener
packages/db/     migrations/*.sql, migrate.ts runner, client.ts, queries/*.ts, types.ts
packages/core/   schedule.ts, winners.ts, orders.ts, invoice.ts, excel.ts (pure, unit-tested)
CLAUDE.md        architecture, commands, Telegram gotchas, conventions
TODO.md          phased checklist (below)
.env.example     DATABASE_URL, BOT_TOKEN, CHANNEL_ID, DISCUSSION_GROUP_ID, S3_* , ADMIN_PASSWORD
docker-compose.yml  postgres (+ minio for local S3-compatible testing)
```

## TODO.md (phased; becomes the repo checklist)
**Phase 0: Scaffold**
- [ ] pnpm workspace, TS config, ESLint/Prettier
- [ ] docker-compose: Postgres + MinIO
- [ ] `001_init.sql`, migration runner (`pnpm db:migrate`), postgres client and row types
- [ ] CLAUDE.md and .env.example

**Phase 1: Batches and items**
- [ ] Admin password auth (middleware)
- [ ] Batch CRUD
- [ ] Item form with presigned S3 photo upload, reorder and delete photos
- [ ] Auto item codes

**Phase 2: Scheduling and posting**
- [ ] `core/schedule.ts` + tests (even spread, jitter, minimum gap)
- [ ] Schedule tab: preview and edit queue, confirm
- [ ] Bot scheduler loop: sendMediaGroup from S3 URLs, store channel ids
- [ ] Handle the auto-forward and store discussion message ids

**Phase 3: Claims**
- [ ] Comment listener → Claim rows (dedupe, ignore admins, order by msg id)
- [ ] `core/winners.ts` + tests (qty > 1, skip/promote)
- [ ] Claims tab: code → ordered user list, winner, skip button, live refresh

**Phase 4: Orders and billing**
- [ ] `core/orders.ts` + tests (group by buyer, idempotent regenerate)
- [ ] Orders tab: buyer cards, per-item paid checkbox, mark all paid
- [ ] Invoice template + copy button; optional bot DM
- [ ] Settings page

**Phase 5: Close and export**
- [ ] Close batch (sold/unsold, lock)
- [ ] `core/excel.ts` + Export endpoint (Items, Claims, Buyers, Payments, Unsold)

**Phase 6: Deploy**
- [ ] Dockerfiles for web and bot; deploy notes (e.g. Railway/Fly + Neon + R2)

## Verification
- `pnpm test`: unit tests for schedule, winners, orders, invoice and excel in `packages/core`.
- Local end-to-end: `docker compose up`, `pnpm db:migrate`, then `pnpm dev` (web + bot) against a **test channel with a linked test group** and a test bot token.
  1. Create a batch with 3 items and photos. Check the objects land in MinIO/R2.
  2. Schedule a 3-minute window. Check the posts appear spaced out, with albums.
  3. Comment from 2 or 3 Telegram accounts. Check the Claims tab order and winners.
  4. Generate orders, copy an invoice, tick items paid.
  5. Close the batch and download the xlsx. Check every sheet.
