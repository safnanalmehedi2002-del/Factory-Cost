# FERO FASHION HOUSE LTD. — Garments ERP

React + Tailwind (Vite) client, Express + Prisma + PostgreSQL API, Google Sheets export, Apps Script automation.

## Run locally
```bash
docker compose up -d                       # Postgres on :5432
cp server/.env.example server/.env
npm install
npm run db:migrate                         # applies prisma/migrations/0001_init
npm run db:seed                            # optional demo data
npm run dev                                # API :4000, web :5173 (proxies /api)
npm test                                   # calc engine unit tests
```
Production: `npm run build`, then `npm run start -w server` and serve `client/dist` from any static host / reverse proxy (route `/api` and `/uploads` to the server).

## Layout
- `server/prisma/schema.prisma`, `migrations/0001_init/migration.sql` — 4 tables with FKs (cascade delete)
- `server/src/lib/calc.ts` — calculation engine (CM↔INC, yield, WIP, reconciliation, status). **`client/src/lib/calc.ts` is a copy; keep them identical** (`cp server/src/lib/calc.ts client/src/lib/calc.ts`)
- `server/src/lib/summary.ts` — cumulative totals, WIP, buyer reconciliation, dashboard KPIs
- `server/src/routes/*` — CRUD: `/api/orders`, `/api/production`, `/api/fabric` (+`/convert`, `/:id/snapshot`, `/reconciliation`), `/api/trims` (+`/alerts`), `/api/dashboard`, `/api/calc/yield`, `POST /api/export/sheets`
- `client/src/components/*` — 4 tabs, yield calculator modal, export dialog
- `apps-script/Code.gs` — `onOpen`, `checkShipmentDeadlines`, `auditFabricShortage`, `sendDailyProductionDigest`, `installTriggers`

## Business rules (decisions worth knowing)
- **Order status is derived** and written back to `master_orders.status` on read: Completed (packed ≥ qty) → Delayed (past deadline) → Urgent (≤ 7 days) → In production (cut > 0) → Not started.
- **Cumulative cut/sew/pack** come from `daily_production`.
- **Fabric received** = sum of `cutting_fabric_grid.fabric_received_kg` when the style has grid rows (challan-backed), otherwise the sum of the daily-log `fabric_received_kg`. Required = order qty × `unit_consumption_kg`.
- Buyer-wise garment impact uses the buyer's qty-weighted average kg/pc; shortage pcs round up, excess pcs round down.
- Critical alerts = delayed + urgent orders + buyers with fabric shortage + trims below buffer. Trim "low" means `received − used < min_buffer`.
- Unit toggle converts stored dimensions (4 dp) and leaves consumption unchanged. The formula adds 10 cm / 5 cm allowances, so inches are converted to cm first.

## Google Sheets export
Create a Google Cloud service account, enable the Sheets + Drive APIs, put its JSON in `GOOGLE_SERVICE_ACCOUNT_JSON`. The header "Export to Sheets" button (or `POST /api/export/sheets {shareWith?, spreadsheetId?}`) creates/refreshes the workbook. The service account owns the file, so pass `shareWith` to give people access. Then paste `apps-script/Code.gs` into Extensions → Apps Script, set `ALERT_EMAILS`, run `installTriggers`.

## Not included
No authentication/roles (put it behind your SSO/VPN or add auth middleware before real use). Uploaded snapshots are stored on local disk in `server/uploads` — mount a volume or switch to object storage in production.
