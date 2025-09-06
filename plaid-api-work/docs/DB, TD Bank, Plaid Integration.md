DB, TD Bank, Plaid Integration

Balances + transactions from TD via Plaid OAuth give you daily visibility with zero password storage. TD and Plaid have a data‑access agreement that uses tokenized APIs (no credential sharing). 
TD Stories

Transactions Sync adds little complexity and gives you deltas in seconds, not full re-downloads. 
Plaid

1) TD‑only service (Node/TS) — balances and transactions

What it does

Plaid Link OAuth → TD login → token exchange

Balances: /accounts/balance/get (fresh, real‑time) 
Plaid

Transactions: /transactions/sync with a saved cursor (idempotent catch‑up) 
Plaid

Minimal DB schema + nightly cron

Assumption: You’ll test in Sandbox first, then request Production + OAuth access in Plaid to link real TD accounts. (Checklist below.) 
Plaid
+1

Folder layout
td-only/
  package.json
  tsconfig.json
  .env.example
  migrations/
    001_init.sql
  src/
    index.ts
    lib/
      db.ts
      plaidClient.ts
    jobs/
      refreshTd.ts
    routes/
      plaid.ts
    scripts/
      refreshTd.ts

.env.example
PLAID_CLIENT_ID=xxxxxxxx
PLAID_SECRET=xxxxxxxx
PLAID_ENV=sandbox    # sandbox | development | production
PLAID_REDIRECT_URI=https://yourapp.example.com/plaid/oauth-return

DATABASE_URL=postgres://<user>:<pass>@<host>:5432/<db>?sslmode=require
PORT=8080

migrations/001_init.sql
-- Plaid items you’ve linked (TD only in this app)
create table if not exists linked_items (
  id bigserial primary key,
  provider text not null default 'plaid',
  item_id text unique not null,
  access_token text not null,
  institution_id text,
  institution_name text,
  status text not null default 'active',
  created_at timestamptz not null default now()
);

-- Snapshot balances for simple rollups
create table if not exists balance_snapshots (
  id bigserial primary key,
  as_of timestamptz not null default now(),
  provider text not null,                -- 'plaid'
  institution text not null,             -- e.g., 'TD Bank'
  item_id text not null,
  account_id text not null,
  account_name text,
  account_type text,
  currency text,
  current_balance numeric,
  available_balance numeric,
  raw jsonb
);
create index if not exists idx_balances_asof on balance_snapshots(as_of);
create index if not exists idx_balances_item on balance_snapshots(item_id);

-- Transactions store (bank transactions)
create table if not exists bank_transactions (
  id bigserial primary key,
  as_of timestamptz not null default now(),
  institution text not null,
  provider text not null default 'plaid',
  item_id text not null,
  account_id text not null,
  transaction_id text not null unique,
  name text,
  merchant_name text,
  date date,
  authorized_date date,
  amount numeric,
  currency text,
  category text[],
  pending boolean,
  is_removed boolean not null default false,
  raw jsonb
);
create index if not exists idx_tx_date on bank_transactions(date);
create index if not exists idx_tx_account on bank_transactions(account_id);

-- Cursor for Transactions Sync per item
create table if not exists plaid_tx_cursors (
  item_id text primary key,
  cursor text,
  updated_at timestamptz not null default now()
);

package.json
{
  "name": "td-only",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "refresh": "node dist/scripts/refreshTd.js"
  },
  "dependencies": {
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "pg": "^8.11.3",
    "plaid": "^24.0.0",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "tsx": "^4.7.0",
    "typescript": "^5.5.4"
  }
}

tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ES2020",
    "moduleResolution": "node",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"]
}

src/lib/db.ts
import { Pool } from 'pg';

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: { rejectUnauthorized: false }
});

export async function query<T = any>(text: string, params?: any[]) {
  const res = await pool.query<T>(text, params);
  return res.rows;
}

src/lib/plaidClient.ts
import { Configuration, PlaidApi, PlaidEnvironments } from 'plaid';

const env = (process.env.PLAID_ENV || 'sandbox') as 'sandbox'|'development'|'production';

const config = new Configuration({
  basePath: PlaidEnvironments[env],
  baseOptions: {
    headers: {
      'PLAID-CLIENT-ID': process.env.PLAID_CLIENT_ID!,
      'PLAID-SECRET': process.env.PLAID_SECRET!
    }
  }
});

export const plaid = new PlaidApi(config);

src/jobs/refreshTd.ts
import { plaid } from '../lib/plaidClient.js';
import { query } from '../lib/db.js';

type ItemRow = {
  item_id: string;
  access_token: string;
  institution_name: string | null;
};

async function upsertBalance(row: {
  institution: string; item_id: string; account_id: string;
  account_name?: string|null; account_type?: string|null;
  currency?: string|null; current_balance?: number|null; available_balance?: number|null; raw?: any;
}) {
  await query(
    `insert into balance_snapshots
      (provider, institution, item_id, account_id, account_name, account_type, currency, current_balance, available_balance, raw)
     values ('plaid', $1,$2,$3,$4,$5,$6,$7,$8,$9)`,
    [
      row.institution, row.item_id, row.account_id,
      row.account_name ?? null, row.account_type ?? null,
      row.currency ?? null, row.current_balance ?? null, row.available_balance ?? null,
      row.raw ?? null
    ]
  );
}

async function upsertTransaction(t: any, institution: string, item_id: string) {
  await query(
    `insert into bank_transactions
      (institution, provider, item_id, account_id, transaction_id, name, merchant_name, date, authorized_date, amount, currency, category, pending, is_removed, raw)
     values ($1,'plaid',$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,false,$13)
     on conflict (transaction_id) do update
       set name = excluded.name,
           merchant_name = excluded.merchant_name,
           date = excluded.date,
           authorized_date = excluded.authorized_date,
           amount = excluded.amount,
           currency = excluded.currency,
           category = excluded.category,
           pending = excluded.pending,
           is_removed = false,
           raw = excluded.raw`,
    [
      institution, item_id, t.account_id, t.transaction_id,
      t.name || null, t.merchant_name || null, t.date ? new Date(t.date) : null,
      t.authorized_date ? new Date(t.authorized_date) : null,
      t.amount ?? null, t.iso_currency_code || t.unofficial_currency_code || null,
      t.category || null, t.pending ?? false, t
    ]
  );
}

async function markRemoved(ids: string[]) {
  if (!ids.length) return;
  await query(`update bank_transactions set is_removed=true where transaction_id = any($1)`, [ids]);
}

export async function refreshBalancesAndTransactionsTD() {
  // Get all Plaid items linked in this app (TD focused, but will refresh whatever you linked)
  const items = await query<ItemRow>(
    `select item_id, access_token, institution_name from linked_items where provider='plaid' and status='active'`
  );

  for (const it of items) {
    const instName = it.institution_name || (await getInstitutionName(it.access_token)) || 'Unknown';

    // --- Balances (real-time) ---
    const bal = await plaid.accountsBalanceGet({ access_token: it.access_token });
    for (const acct of bal.data.accounts) {
      await upsertBalance({
        institution: instName,
        item_id: it.item_id,
        account_id: acct.account_id,
        account_name: acct.name || acct.official_name || acct.mask || acct.account_id,
        account_type: [acct.type, acct.subtype].filter(Boolean).join(':'),
        currency: acct.balances.iso_currency_code || acct.balances.unofficial_currency_code || null,
        current_balance: acct.balances.current ?? null,
        available_balance: acct.balances.available ?? null,
        raw: acct
      });
    }

    // --- Transactions via /transactions/sync ---
    let cursorRow = await query<{ cursor: string }>(`select cursor from plaid_tx_cursors where item_id=$1`, [it.item_id]);
    let cursor = cursorRow[0]?.cursor || null;

    // Loop until has_more = false
    while (true) {
      const resp = await plaid.transactionsSync({
        access_token: it.access_token,
        cursor: cursor || undefined,
        count: 500
      });

      for (const tx of resp.data.added) await upsertTransaction(tx, instName, it.item_id);
      for (const tx of resp.data.modified) await upsertTransaction(tx, instName, it.item_id);
      const removedIds = resp.data.removed.map(r => r.transaction_id).filter(Boolean);
      if (removedIds.length) await markRemoved(removedIds);

      cursor = resp.data.next_cursor;
      await query(
        `insert into plaid_tx_cursors(item_id, cursor) values ($1,$2)
         on conflict (item_id) do update set cursor=excluded.cursor, updated_at=now()`,
        [it.item_id, cursor]
      );

      if (!resp.data.has_more) break;
    }
  }
}

async function getInstitutionName(access_token: string) {
  const item = await plaid.itemGet({ access_token });
  const instId = item.data.item.institution_id;
  if (!instId) return null;
  const inst = await plaid.institutionsGetById({ institution_id: instId, country_codes: ['US'] });
  return inst.data.institution.name || null;
}

src/routes/plaid.ts
import { Router } from 'express';
import { z } from 'zod';
import { plaid } from '../lib/plaidClient.js';
import { query } from '../lib/db.js';

export const plaidRouter = Router();

/** Create a Link token for TD (Transactions product for balances + txns). */
plaidRouter.post('/create_link_token', async (req, res, next) => {
  try {
    const body = z.object({
      products: z.array(z.string()).optional() // optional override
    }).parse(req.body);
    const products = (body.products ?? ['transactions']) as any;

    const resp = await plaid.linkTokenCreate({
      user: { client_user_id: 'david-td' },   // replace with your user id
      client_name: 'TD Balances',
      country_codes: ['US'],
      language: 'en',
      products,
      redirect_uri: process.env.PLAID_REDIRECT_URI || undefined  // required for OAuth institutions
    });
    res.json({ link_token: resp.data.link_token });
  } catch (err) { next(err); }
});

/** Exchange public_token -> access_token; store item & institution. */
plaidRouter.post('/exchange_public_token', async (req, res, next) => {
  try {
    const { public_token } = z.object({ public_token: z.string().min(1) }).parse(req.body);
    const exchanged = await plaid.itemPublicTokenExchange({ public_token });
    const access_token = exchanged.data.access_token;
    const item_id = exchanged.data.item_id;

    const item = await plaid.itemGet({ access_token });
    const institution_id = item.data.item.institution_id || null;
    let institution_name: string | null = null;
    if (institution_id) {
      const inst = await plaid.institutionsGetById({ institution_id, country_codes: ['US'] });
      institution_name = inst.data.institution.name || null;
    }

    await query(
      `insert into linked_items (provider, item_id, access_token, institution_id, institution_name)
       values ('plaid',$1,$2,$3,$4)
       on conflict (item_id) do update set access_token=excluded.access_token,
                                         institution_id=excluded.institution_id,
                                         institution_name=excluded.institution_name`,
      [item_id, access_token, institution_id, institution_name]
    );

    res.json({ ok: true, item_id, institution_id, institution_name });
  } catch (err) { next(err); }
});

src/index.ts
import 'dotenv/config';
import express from 'express';
import { plaidRouter } from './routes/plaid.js';
import { refreshBalancesAndTransactionsTD } from './jobs/refreshTd.js';
import { query } from './lib/db.js';

const app = express();
app.use(express.json({ limit: '1mb' }));

app.use('/plaid', plaidRouter);

/** Manual refresh endpoint */
app.post('/td/refresh', async (_req, res, next) => {
  try {
    await refreshBalancesAndTransactionsTD();
    res.json({ ok: true });
  } catch (e) { next(e); }
});

/** Simple rollups */
app.get('/td/balances/recent', async (req, res, next) => {
  try {
    const days = Number(req.query.days ?? 30);
    const rows = await query(
      `
      select date_trunc('day', as_of) as day,
             institution,
             sum(coalesce(current_balance,0)) as total
      from balance_snapshots
      where as_of >= now() - ($1 || ' days')::interval
      group by 1,2
      order by 1 desc, 2 asc
      `,
      [days]
    );
    res.json({ days, data: rows });
  } catch (e) { next(e); }
});

app.get('/td/transactions/recent', async (req, res, next) => {
  try {
    const days = Number(req.query.days ?? 30);
    const rows = await query(
      `
      select date, institution, name, merchant_name, amount, currency, pending
      from bank_transactions
      where date >= current_date - $1::int and is_removed=false
      order by date desc
      `,
      [days]
    );
    res.json({ days, data: rows });
  } catch (e) { next(e); }
});

app.use((err: any, _req: any, res: any, _next: any) => {
  console.error(err);
  res.status(500).json({ ok: false, error: err.message || 'Internal error' });
});

const PORT = Number(process.env.PORT || 8080);
app.listen(PORT, () => console.log(`TD-only service on :${PORT}`));

src/scripts/refreshTd.ts
import 'dotenv/config';
import { refreshBalancesAndTransactionsTD } from '../jobs/refreshTd.js';

(async () => {
  await refreshBalancesAndTransactionsTD();
  console.log('TD refresh done');
})();

How to run (step‑by‑step)

Create a Postgres DB; run migrations/001_init.sql.

npm i → npm run dev (or build+start).

Create Link token (TD via Transactions product):

curl -X POST "http://localhost:8080/plaid/create_link_token" \
  -H "Content-Type: application/json" -d '{}'


Open Plaid Link in your frontend with that link_token → pick TD Bank (US) → complete OAuth → your app receives a public_token.

Exchange it:

curl -X POST "http://localhost:8080/plaid/exchange_public_token" \
  -H "Content-Type: application/json" \
  -d '{"public_token":"PUBLIC_TOKEN"}'


Refresh:

curl -X POST "http://localhost:8080/td/refresh"


Read data:

curl "http://localhost:8080/td/balances/recent?days=30"
curl "http://localhost:8080/td/transactions/recent?days=14"


Balances come from /accounts/balance/get (forces a real‑time refresh); transactions use /transactions/sync with a saved cursor for incremental updates. 
Plaid
+1