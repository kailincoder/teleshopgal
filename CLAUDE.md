# Tele Gal

Telegram-first selling tool. A seller creates a **batch** (drop) of items. The bot posts the items to a Telegram **channel** spread across a time window. Buyers claim an item by commenting on its post in the linked discussion group, and the first commenter wins. The admin dashboard then turns the winners into per-buyer orders, bills them, tracks payments, closes the batch and exports it to Excel.

Full design: [docs/PLAN.md](docs/PLAN.md). Progress checklist: [TODO.md](TODO.md).

## Stack
- **pnpm workspaces**, TypeScript everywhere
- `apps/web`: Next.js 15 (App Router) + Tailwind; admin UI and API routes
- `apps/bot`: grammY worker (long polling); scheduler loop and comment listener
- `packages/db`: **raw SQL** with the `postgres` (porsager) driver. **No ORM.**
- `packages/core`: pure business logic (schedule, winners, orders, invoice, excel), unit-tested
- PostgreSQL, S3 / Cloudflare R2 for photos (MinIO locally)

## Commands
```
docker compose up -d      # postgres + minio
pnpm install
pnpm db:migrate           # apply packages/db/migrations/*.sql
pnpm dev                  # web + bot
pnpm test                 # unit tests (packages/core)
pnpm lint
```

## Database conventions
- Migrations are numbered plain SQL files: `packages/db/migrations/NNN_name.sql`. **Never edit an applied migration**; add a new one. The runner records applied files in `schema_migrations`.
- Queries live in `packages/db/src/queries/<table>.ts` as tagged templates (`` sql`select ... where id = ${id}` ``). Never build SQL by string concatenation.
- Tables and columns are snake_case. TS row types in `packages/db/src/types.ts` are written by hand and kept in sync with the migrations.
- Money is `numeric(10,2)`, so treat it as a string or decimal in TS and never as a float for totals. Telegram ids are `bigint`, so treat them as `string`/`bigint` in TS.
- Multi-step writes (generate orders, close batch) go in `sql.begin(async sql => ...)`.

## Domain rules
- Item codes: `B001`, `B002`, … auto-assigned per batch in creation order.
- Scheduling: N items spread evenly over the window with ±20% jitter and a minimum gap of 60s (`core/schedule.ts`).
- Claims: **any comment** on the post counts. The order is by Telegram group `message_id` (strictly increasing), not timestamp. Only a user's first comment per item counts. Comments from admins are ignored.
- Winners: the first `qty` non-skipped claimers. The seller can **skip** a claimer, which promotes the next one.
- Orders: one per buyer per batch, built by grouping winners. Regenerating is idempotent and **keeps per-item `paid` flags**.
- Close batch: items with a winner become `sold`, the rest become `unsold`, and the batch is locked (read-only).
- Excel sheets: Items, Claims (code, then ordered claimers), Buyers (buyer → items, total, paid), Payments, Unsold.

## Telegram gotchas
- The bot must be **admin in both** the channel (to post) and the linked discussion group (to read all messages).
- Each channel post is auto-forwarded into the discussion group (`is_automatic_forward`). Store those discussion `message_id`s on the item, because comments are replies to them. Albums produce one forwarded message per photo, so store all of them.
- The Bot API cannot fetch history. **Claims are only captured while the bot is running.** Keep the worker up during drops.
- Albums are `sendMediaGroup`, max 10 photos, with the caption on the first photo only.
- Users may have no username. Key on `telegram_user_id` and link with `tg://user?id=<id>`.

## S3 layout
`drops/{yyyy-mm}/{batch_id}/{item_code}/{n}.jpg`. Uploads go browser → S3 via a presigned PUT from `/api/uploads/presign`.

## Env
See `.env.example`: `DATABASE_URL`, `BOT_TOKEN`, `CHANNEL_ID`, `DISCUSSION_GROUP_ID`, `S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `S3_PUBLIC_BASE_URL`, `ADMIN_PASSWORD`, `SESSION_SECRET`.

## Working rules
- Keep business logic in `packages/core` (pure, testable). Routes and bot handlers should stay thin.
- When a TODO.md item is finished, tick it in the same change.
