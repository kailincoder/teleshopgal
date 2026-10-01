# Tele Gal: TODO

## Phase 0: Scaffold
- [ ] pnpm workspace, TS config, ESLint/Prettier
- [ ] docker-compose: Postgres + MinIO
- [ ] `001_init.sql`, migration runner (`pnpm db:migrate`), postgres client, row types
- [x] CLAUDE.md
- [ ] .env.example

## Phase 1: Batches and items
- [ ] Admin password auth (middleware + signed cookie)
- [ ] Batch CRUD
- [ ] Item form: title, description, price, qty
- [ ] Presigned S3 photo upload, reorder and delete photos
- [ ] Auto item codes (B001…)

## Phase 2: Scheduling and posting
- [ ] `core/schedule.ts` + tests (even spread, ±20% jitter, ≥60s gap)
- [ ] Schedule tab: preview and edit queue, confirm
- [ ] Bot scheduler loop: sendMediaGroup from S3 URLs, store channel message ids
- [ ] Handle the auto-forward and store discussion message ids

## Phase 3: Claims
- [ ] Comment listener → claims (dedupe per user, ignore admins, order by message_id)
- [ ] `core/winners.ts` + tests (qty > 1, skip/promote)
- [ ] Claims tab: code → ordered user list, winner highlighted, skip button, live refresh

## Phase 4: Orders and billing
- [ ] `core/orders.ts` + tests (group by buyer, idempotent regenerate keeps paid flags)
- [ ] Orders tab: buyer cards, per-item paid checkbox, mark all paid
- [ ] `core/invoice.ts` template + copy button; optional bot DM
- [ ] Settings page (PayNow number, invoice template)

## Phase 5: Close and export
- [ ] Close batch (sold/unsold, lock, confirm when unpaid)
- [ ] `core/excel.ts` + `GET /api/batches/:id/export` (Items, Claims, Buyers, Payments, Unsold)

## Phase 6: Deploy
- [ ] Dockerfiles for web and bot
- [ ] Deploy notes (e.g. Railway/Fly + Neon + R2)

## End-to-end check
- [ ] Test channel + linked group + test bot: 3 items → 3-min window → comments from 2–3 accounts → orders → paid → close → xlsx
