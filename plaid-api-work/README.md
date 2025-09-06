# Plaid API Integration Project

LLC Finance API (Minimal, Single-User)

Purpose
- Persist the frontend dashboard state as a single JSON document.
- Optionally fetch bank balances via Plaid (TD/Chase, etc.).
- **NEW**: Fetch and sync bank transactions via Plaid with efficient incremental syncing.
- Single user, simple API key auth, no renters/invoices, no splits, no reconciliation.

Stack
- Python FastAPI + Uvicorn
- Postgres via SQLAlchemy + psycopg
- cryptography (Fernet) for encrypting Plaid access tokens
- plaid-python (optional paths)

API
- GET `/state` → returns the saved JSON state (document)
- PUT `/state` → replaces the saved JSON state
- POST `/plaid/link-token` → returns a Plaid Link token (if Plaid env configured)
- POST `/plaid/exchange` → exchange `public_token` → stores encrypted `access_token`
- POST `/plaid/refresh-balances` → fetches balances, upserts cache, returns balances
- **NEW**: POST `/plaid/refresh-transactions` → syncs transactions, returns new/modified transactions

Auth
- All endpoints require header `X-API-Key: <API_KEY>`

Environment Variables
- `DATABASE_URL` (required): e.g. postgres://user:pass@host:5432/dbname
- `API_KEY` (required): shared secret for single-user access
- `FERNET_KEY` (required for Plaid): 32-byte base64 key (run `python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`)
- `PLAID_CLIENT_ID` (optional): for Plaid endpoints
- `PLAID_SECRET` (optional): for Plaid endpoints
- `PLAID_ENV` (optional): sandbox|development|production (default: sandbox)

Quick Start (local)
1) Create and activate a virtualenv
   - `python -m venv .venv && source .venv/bin/activate`
2) Install deps
   - `pip install -r requirements.txt`
3) Set env vars (see .env.example)
4) Run
   - `uvicorn app.main:app --reload --port 8000`

Endpoints (curl examples)
- Save state:
  - `curl -X PUT http://localhost:8000/state -H "X-API-Key: $API_KEY" -H 'Content-Type: application/json' -d '{"accountsData": {}}'`
- Load state:
  - `curl http://localhost:8000/state -H "X-API-Key: $API_KEY"`

Plaid (optional)
- Link token:
  - `curl -X POST http://localhost:8000/plaid/link-token -H "X-API-Key: $API_KEY"`
- Exchange public token:
  - `curl -X POST http://localhost:8000/plaid/exchange -H "X-API-Key: $API_KEY" -H 'Content-Type: application/json' -d '{"public_token":"PUBLIC-..."}'`
- Refresh balances:
  - `curl -X POST http://localhost:8000/plaid/refresh-balances -H "X-API-Key: $API_KEY"`
- **NEW**: Refresh transactions:
  - `curl -X POST http://localhost:8000/plaid/refresh-transactions -H "X-API-Key: $API_KEY"`

## Recent Updates - Transaction Integration

### New Features Added (v0.2.0)
- **Transaction Syncing**: Full integration with Plaid's transaction API
- **Efficient Syncing**: Uses Plaid's `transactions_sync` endpoint for incremental updates
- **Database Schema**: New `plaid_transactions` table with proper indexing and constraints
- **Sync Cursors**: Tracks sync state to avoid duplicate data fetching

### Database Changes
The following new table has been added to the database schema:

```sql
-- New table: plaid_transactions
CREATE TABLE plaid_transactions (
    id SERIAL PRIMARY KEY,
    plaid_account_id TEXT NOT NULL,
    plaid_transaction_id TEXT NOT NULL UNIQUE,
    plaid_item_id TEXT REFERENCES plaid_items(item_id),
    date DATE NOT NULL,
    name TEXT NOT NULL,
    amount_cents BIGINT NOT NULL,  -- Stored in cents for precision
    currency TEXT(3) NOT NULL,
    category TEXT,
    pending BOOLEAN DEFAULT FALSE,
    raw JSONB,  -- Stores complete Plaid response
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- New column added to plaid_items
ALTER TABLE plaid_items ADD COLUMN sync_cursor TEXT;
```

### Transaction Data Structure
Transactions are returned in the following format:

```json
{
  "transactions": [
    {
      "plaid_transaction_id": "unique_plaid_id",
      "plaid_account_id": "account_reference",
      "date": "2024-01-15",
      "name": "STARBUCKS COFFEE",
      "amount": 5.75,
      "currency": "USD",
      "category": "Food and Drink",
      "pending": false
    }
  ]
}
```

### How Transaction Syncing Works
1. **Initial Sync**: First call fetches all available transactions
2. **Incremental Updates**: Subsequent calls only fetch changes since last sync
3. **Efficient Storage**: Uses sync cursors to track sync state per Plaid item
4. **Data Integrity**: Handles added, modified, and removed transactions
5. **Duplicate Prevention**: Unique constraints prevent duplicate transaction storage

### Plaid Product Scope Update
The link token creation now includes both `balance` and `transactions` products, allowing your frontend to request transaction access during Plaid Link setup.

Render Deployment
- Create a new Web Service
- Start command: `uvicorn app.main:app --host 0.0.0.0 --port 10000`
- Set env vars (DATABASE_URL, API_KEY, FERNET_KEY, and Plaid vars if needed)

Blueprint via Render CLI
- Install CLI: `npm i -g @renderinc/cli` or follow Render docs
- Login: `render login` (opens a browser; complete auth)
- From this folder (`llc-finance-api/`), deploy:
  - `render blueprint deploy`  (uses `render.yaml`)
  - This creates both the Postgres database and the web service.
- Set secrets (if not already set during deploy):
  - `render secrets set llc-finance-api API_KEY=... FERNET_KEY=...`
  - Or via dashboard: define secrets `llc_finance_api_key`, `llc_finance_fernet_key`, `plaid_client_id`, `plaid_secret` used in render.yaml
- After deploy, fetch the service URL from the CLI or dashboard.

Notes
- The service listens on port 10000 in Docker, which Render expects.
- Health check is `/healthz`.
- DATABASE_URL is sourced from the managed Postgres defined in `render.yaml`. If you prefer an existing DB, remove the `databases:` block and set `DATABASE_URL` as an env var instead.
- **NEW**: After deploying, the database will automatically create the new `plaid_transactions` table and add the `sync_cursor` column to `plaid_items`.
- **NEW**: Transaction syncing requires the `transactions` product scope in your Plaid Link setup.
