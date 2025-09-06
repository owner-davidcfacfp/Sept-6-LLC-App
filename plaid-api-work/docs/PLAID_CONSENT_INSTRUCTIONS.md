
# Plaid Consent UX + Integration — Hand‑off for Dev

> **Project-Specific Implementation Note:** The code examples in this document use a **Node.js/TypeScript/React** stack. This project's actual implementation uses a **Python/FastAPI** backend and **vanilla JavaScript** on the frontend. While the programming languages and libraries are different, the core concepts, security measures, and API endpoints described here have been implemented in the project's codebase. This document should be used as a conceptual guide and for its detailed consent and review requirements.

> Purpose: Implement a reviewer‑friendly consent experience and a minimal, compliant Plaid Link flow (TD Bank via OAuth), with copy and code you can drop in. Replace placeholders like **[COMPANY]** and **[APP_URL]**.

---

## A) General Requirements (What Plaid reviewers expect)

1. **Explicit consent near the action**: Visible text that states we use Plaid, what data we access (balances + transactions), and that we **do not** collect bank credentials.
2. **Links to policies**: Our **Privacy Policy** and **Plaid’s End User Privacy Policy** linked at the point of consent.
3. **OAuth‑capable Link**: Use `redirect_uri` and `products: ["transactions"]` (balances come from `/accounts/balance/get`).
4. **Auth’d user flow**: Show that the user is logged in, then launches Link from a protected dashboard.
5. **Token security**: Exchange `public_token` → `access_token`, store access token **encrypted** at rest.
6. **Proof for review**: Provide screenshots of (1) Sign‑up, (2) Login, (3) Dashboard with consent text, (4) Plaid Link (TD OAuth), (5) Return to app, (6) Balances/transactions visible.

---

## B) Consent Copy (Place above the button)

**Headline (visible):**  
> **We use Plaid for secure bank connections.**

**One‑liner consent (visible):**  
> *By clicking “Connect Bank,” you authorize [COMPANY] to connect to your financial institution via Plaid to securely access your balances and transactions for display in your dashboard. We do not collect or store your banking credentials. See our [Privacy Policy]([APP_URL]/privacy) and Plaid’s [End User Privacy Policy](https://plaid.com/legal/#end-user-privacy-policy).*

**Short “tooltip‑style” text (optional, for a cleaner UI):**  
> *When you connect an account, [COMPANY] uses Plaid to securely access your balances and transactions. We never see or store your banking credentials. Data is handled under our [Privacy Policy]([APP_URL]/privacy) and Plaid’s [End User Privacy Policy](https://plaid.com/legal/#end-user-privacy-policy).*

> **Reviewer tip**: During review, keep the **one‑liner visible on the page**. After approval, you may rely on the tooltip to declutter the UI, but keep the consent accessible.

---

## C) Frontend (React) — Button + Descriptive Consent Toggle

Install: `react-plaid-link`

```tsx
import { useState, useEffect, useCallback } from 'react';
import { usePlaidLink } from 'react-plaid-link';

function ConsentTooltip() {
  return (
    <div className="bg-white border border-gray-300 p-3 rounded-lg shadow-md w-80 text-sm text-gray-700">
      <p>
        When you connect an account, <strong>[COMPANY]</strong> uses Plaid to securely access
        your balances and transactions. We never see or store your banking credentials.
      </p>
      <p className="mt-2">
        See our{" "}
        <a href="/privacy" target="_blank" className="text-blue-600 underline">
          Privacy Policy
        </a>{" "}
        and{" "}
        <a
          href="https://plaid.com/legal/#end-user-privacy-policy"
          target="_blank"
          className="text-blue-600 underline"
        >
          Plaid’s End User Privacy Policy
        </a>
        .
      </p>
    </div>
  );
}

export function ConnectBankSection() {
  const [linkToken, setLinkToken] = useState<string>();
  const [showTooltip, setShowTooltip] = useState(false);

  useEffect(() => {
    (async () => {
      const r = await fetch('/plaid/create_link_token', { method: 'POST', credentials: 'include' });
      const j = await r.json();
      setLinkToken(j.link_token);
    })();
  }, []);

  const onSuccess = useCallback(async (public_token: string) => {
    await fetch('/plaid/exchange_public_token', {
      method: 'POST',
      credentials: 'include',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ public_token }),
    });
    window.location.reload();
  }, []);

  const { open, ready } = usePlaidLink({ token: linkToken, onSuccess });

  return (
    <section className="space-y-2">
      <h2 className="text-lg font-semibold">We use Plaid for secure bank connections.</h2>
      <p className="text-sm text-gray-700">
        By clicking “Connect Bank,” you authorize [COMPANY] to connect to your financial institution via Plaid to
        securely access your balances and transactions for display in your dashboard. We do not collect or store your
        banking credentials. See our{" "}
        <a href="/privacy" className="text-blue-600 underline" target="_blank">Privacy Policy</a> and{" "}
        <a href="https://plaid.com/legal/#end-user-privacy-policy" className="text-blue-600 underline" target="_blank">
          Plaid’s End User Privacy Policy
        </a>.
      </p>

      <div className="relative inline-block">
        <button
          disabled={!ready}
          onClick={() => open()}
          className="px-4 py-2 rounded-lg bg-green-600 text-white font-medium hover:bg-green-700"
        >
          Connect your bank securely with Plaid
        </button>
        <button
          type="button"
          onClick={() => setShowTooltip(!showTooltip)}
          className="ml-3 text-sm text-blue-600 underline"
          aria-expanded={showTooltip}
          aria-controls="consent-tooltip"
        >
          Why we ask you to connect
        </button>
        {showTooltip && (
          <div id="consent-tooltip" className="absolute mt-2 z-10">
            <ConsentTooltip />
          </div>
        )}
      </div>
    </section>
  );
}
```

---

## D) Backend (Express/TS) — Minimal Auth + Plaid Link Endpoints

**Assumptions**: You already have a login session (cookie or header). Plaid routes require auth. Tokens stored encrypted at rest.

Env:
```
PLAID_CLIENT_ID=...
PLAID_SECRET=...
PLAID_ENV=sandbox           # later: production
PLAID_REDIRECT_URI=https://[APP_URL]/plaid/oauth-return
DATABASE_URL=postgres://...
ENCRYPTION_KEY=<32 bytes base64>  # openssl rand -base64 32
```

DB tables (simplified):
```sql
create table if not exists plaid_items (
  user_id bigint not null,
  item_id text primary key,
  access_token_enc text not null,
  institution_id text,
  institution_name text,
  created_at timestamptz not null default now()
);
```

Routes:
```ts
import { Configuration, PlaidApi, PlaidEnvironments } from 'plaid';
import { Pool } from 'pg';
import crypto from 'crypto';

const db = new Pool({ connectionString: process.env.DATABASE_URL, ssl:{rejectUnauthorized:false} });

// AES-256-GCM helpers
const ENC_KEY = Buffer.from(process.env.ENCRYPTION_KEY!, 'base64');
function enc(plain:string){ const iv = crypto.randomBytes(12);
  const c = crypto.createCipheriv('aes-256-gcm', ENC_KEY, iv);
  const ct = Buffer.concat([c.update(plain,'utf8'), c.final()]); const tag=c.getAuthTag();
  return Buffer.concat([iv, tag, ct]).toString('base64');
}

const cfg = new Configuration({
  basePath: PlaidEnvironments[process.env.PLAID_ENV as 'sandbox'|'development'|'production'],
  baseOptions: { headers: { 'PLAID-CLIENT-ID': process.env.PLAID_CLIENT_ID!, 'PLAID-SECRET': process.env.PLAID_SECRET! } }
});
const plaid = new PlaidApi(cfg);

// POST /plaid/create_link_token  (auth required)
export async function createLinkToken(req:any, res:any){
  const userId = req.user.uid; // set by your auth middleware
  const resp = await plaid.linkTokenCreate({
    user: { client_user_id: String(userId) },
    client_name: '[COMPANY] Personal Finance',
    products: ['transactions'],
    country_codes: ['US'],
    language: 'en',
    redirect_uri: process.env.PLAID_REDIRECT_URI
  });
  res.json({ link_token: resp.data.link_token });
}

// POST /plaid/exchange_public_token  (auth required)
export async function exchangePublicToken(req:any, res:any){
  const userId = req.user.uid;
  const exchanged = await plaid.itemPublicTokenExchange({ public_token: req.body.public_token });
  const access_token = exchanged.data.access_token;
  const item = await plaid.itemGet({ access_token });
  const instId = item.data.item.institution_id || null;
  let instName = 'Unknown';
  if (instId) {
    const inst = await plaid.institutionsGetById({ institution_id: instId, country_codes: ['US'] });
    instName = inst.data.institution.name || 'Unknown';
  }
  await db.query(
    `insert into plaid_items(user_id, item_id, access_token_enc, institution_id, institution_name)
     values ($1,$2,$3,$4,$5)
     on conflict (item_id) do update set access_token_enc=excluded.access_token_enc,
                                       institution_id=excluded.institution_id,
                                       institution_name=excluded.institution_name`,
    [userId, exchanged.data.item_id, enc(access_token), instId, instName]
  );
  res.json({ ok:true, item_id: exchanged.data.item_id, institution: instName });
}
```

---

## E) Screenshot Checklist (attach to Plaid email)

1. **Sign‑up** screen (before Link appears).  
2. **Login** screen.  
3. **Dashboard** with:  
   - Headline: “We use Plaid for secure bank connections.”  
   - One‑liner consent text (visible).  
   - Button: “Connect your bank securely with Plaid”.  
   - Link: “Why we ask you to connect” (show a capture with tooltip expanded).  
4. **Plaid Link** OAuth window for TD Bank.  
5. **Return to app** showing a “Connected” state.  
6. **Balances/Transactions** view populated.

---

## F) Reviewer Notes (add to your Plaid email)

- Only the end‑user connects their own accounts via **Plaid Link**; **no credential collection** by [COMPANY].  
- Access tokens are stored **encrypted** (AES‑256‑GCM) and never exposed to the frontend.  
- Data is encrypted **in transit** (TLS 1.2+) and **at rest** by our managed database.  
- We do **not** sell consumer data; usage is limited to balance and transaction display for the account owner.

---

## G) Post‑approval: Minimal Balances Fetch (optional)

```ts
// GET /balances  (auth required)
export async function getBalances(req:any, res:any){
  const userId = req.user.uid;
  const items = await db.query(`select access_token_enc from plaid_items where user_id=$1`, [userId]);
  const accounts:any[] = [];
  for (const row of items.rows) {
    const access_token = '*** decrypt here if you add a dec() helper ***';
    const r = await plaid.accountsBalanceGet({ access_token });
    accounts.push(...r.data.accounts);
  }
  res.json({ accounts });
}
```

---

### Done. Replace placeholders, wire routes into your server, and capture the screenshots for Plaid.
