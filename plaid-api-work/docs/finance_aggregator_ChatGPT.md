Perfect—let’s lock this to **TD Bank (US retail)**, **Chase**, **Fidelity**, and **Gemini** with the least friction, using **OAuth + first‑party APIs** where possible, and a **CSV fallback** only where needed.

---

## Why this approach (quick reminder)

* **Secure & durable:** OAuth + tokenized APIs = no stored passwords, fewer breakages. Chase and TD both support API‑based connections; Chase in particular enforces OAuth for Plaid connections. ([Chase][1], [Chase Media Center][2], [Stack Overflow][3])
* **Balances are easy:** Your daily roll‑up can be one API call per connector. Plaid’s `/accounts/balance/get` gives real‑time balances. Gemini exposes balances directly via its private REST API. ([Plaid][4], [Gemini Docs][5])
* **Fidelity reality check:** Fidelity pushes traffic to **Akoya** and has tightened down on scraping; aggregator coverage for Fidelity can vary. Keep a CSV fallback ready. ([Fidelity Newsroom][6], [Akoya][7])

---

## Recommended connector per institution

| Institution                         | Primary (best today)                                                                                                | Fallback                                                    | Notes                                                                                                                                                                                                                |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **TD Bank (US)**                    | **Plaid** via OAuth (Link → `/accounts/balance/get`)                                                                | Teller (dev‑friendly bank data)                             | TD announced a North American **data‑access agreement** with Plaid for API connections. Plaid lists a TD Bank institution page. ([TD Stories][8], [Plaid][9])                                                        |
| **Chase**                           | **Plaid OAuth** (Link → app‑to‑app Chase login → token)                                                             | Teller                                                      | Chase provides a **secure API** and requires OAuth for Plaid apps; expect an app‑to‑app redirect during linking. ([Chase][1], [Stack Overflow][3])                                                                   |
| **Fidelity (brokerage/retirement)** | **Aggregator via Akoya** (where supported) → **Plaid Investments** endpoints for balances/holdings **if available** | **Manual CSV export** (Balances/Positions) ingested nightly | Fidelity is actively pushing API‑based data sharing via **Akoya**; some aggregators’ direct coverage is limited—use the CSV fallback if your aggregator path isn’t enabled yet. ([Fidelity Newsroom][6], [Akoya][7]) |
| **Gemini (crypto)**                 | **Gemini REST (private)**: `/v1/balances` with API key (read‑only)                                                  | —                                                           | Official API returns available & current balances; authenticate with headers + signed payload. ([Gemini Docs][5])                                                                                                    |

> **Why this mix?** TD/Chase bank balances are cleanest through Plaid OAuth. Fidelity’s access is improving but uneven because of Akoya policies—so keep CSV export in your back pocket. Gemini’s own API is straightforward and reliable.

---

## What each connector returns (balances‑only path)

* **Plaid (banks):** call `/accounts/balance/get` → `[{account_id, name, type, subtype, balances: {current, available, iso_currency_code}}]`. Use once or twice daily. ([Plaid][4])
* **Plaid (investments, for Fidelity if available):** use `/investments/holdings/get` when you want positions + market value; for a pure **balance**, you can still read the account’s `balances.current`. ([Plaid][10])
* **Gemini:** call **`/v1/balances`** (private) → `[{currency, amount, available, availableForWithdrawal, ...}]`. ([Gemini Docs][5])
* **Fidelity CSV fallback:** from **Balances** or **Positions** pages, click **Download** → CSV; ingest and sum. (Fidelity documents downloading balances/positions as CSV.) ([Fidelity Investments][11])

---

## Step‑by‑step build (Render + Node + Postgres)

### 0) One schema to rule them all (simple & extensible)

Create a **`balance_snapshots`** table:

```
id (pk) • as_of (timestamptz) • provider (text) • institution (text)
account_id (text) • account_name (text) • account_type (text)
currency (text) • current_balance (numeric) • available_balance (numeric)
raw (jsonb)
```

Rationale: fast to group/sum by date; `raw` keeps vendor‑specific fields for debugging/extension.

### 1) Banks (TD + Chase) via Plaid

1. Enable **OAuth** in Plaid and add your Redirect URI(s). Chase specifically requires OAuth for Plaid apps. ([Plaid][12], [Stack Overflow][3])
2. Use **Link** in your frontend → exchange `public_token` → `access_token`.
3. Nightly job: call `/accounts/balance/get` for each `access_token` and upsert rows. ([Plaid][4])

> **Why Plaid over Teller here?** You’re already targeting Fidelity investments later; staying within one SDK simplifies your stack. Teller is still a fine option if you want a lighter‑weight, bank‑only path (per‑enrollment pricing, dev‑friendly). ([Teller][13])

### 2) Fidelity (primary: aggregator via Akoya; fallback: CSV)

* **Preferred:** If your aggregator supports Fidelity through **Akoya**, link via OAuth and ingest balances like any other investment account (use Investments endpoints for holdings if/when needed). ([Akoya][7])
* **Fallback (rock‑solid):** Add a tiny uploader in your app. From Fidelity **Balances** or **Positions**, click **Download** (CSV), upload, parse, store totals. It’s fast and ToS‑friendly. ([Fidelity Investments][11])

> **Blunt truth:** Fidelity is deliberately moving traffic off scraping and into Akoya. If your chosen aggregator doesn’t have your specific Fidelity account type wired up yet, the CSV path avoids spinning cycles. ([Fidelity Newsroom][6])

### 3) Gemini (direct)

* Create a **read‑only API key** in Gemini; store in environment secrets.
* POST to **`/v1/balances`** with `X-GEMINI-APIKEY`, `X-GEMINI-PAYLOAD`, `X-GEMINI-SIGNATURE` (HMAC‑SHA384 of the base64 payload). Store results as snapshots. ([Gemini Docs][5])

### 4) Roll‑up card in your dashboard

* Show **Today’s Total** and **by‑institution** totals.
* Add a 7/30‑day **sparkline** using the snapshots table.
* Flag any institution whose data is >48h old (helps catch OAuth disconnects).

---

## “Granular later” (when you want more than balances)

* **Transactions (banks):** Plaid `/transactions/sync`.
* **Positions & market value (Fidelity/other brokerages):** Plaid **Investments** (`/investments/holdings/get`, `/investments/transactions/get`). ([Plaid][14])
* **Crypto details:** Gemini provides additional endpoints for transfers, positions, etc., but balances will cover your current scope. ([Gemini Docs][5])

---

## Security (non‑negotiables)

* **Never store bank usernames/passwords.** Use OAuth (Plaid handles auth with Chase/TD). ([Chase][1])
* Keep all tokens/keys in **Render Secrets**; encrypt at rest in Postgres.
* Scope Gemini keys to **read‑only** and rotate periodically.
* Provide a **“Disconnect”** button that revokes/forgets tokens.

---

## Known frictions & how we avoid them

* **Chase linking breaks** if OAuth prerequisites aren’t set—ensure your Plaid dashboard is enabled for US OAuth institutions. ([Plaid][12])
* **TD connectivity** is generally API‑based through Plaid now (agreement in place); if you ever see odd MFA loops, re‑link via OAuth. ([TD Stories][8])
* **Fidelity coverage varies by aggregator** due to Akoya; use CSV if the OAuth path isn’t available for your specific account type yet. ([Akoya][7])

---

## Concrete next actions (you can do these today)

1. **Plaid setup:** Add Redirect URIs, enable OAuth institutions; wire Link → exchange token → call `/accounts/balance/get`. ([Plaid][4])
2. **Gemini:** Create read‑only key; implement `/v1/balances` call. ([Gemini Docs][5])
3. **Fidelity fallback:** Build a simple CSV uploader & parser; map columns → `balance_snapshots`. (Use Balances/Positions page → **Download**.) ([Fidelity Investments][11])
4. **Cron:** Nightly job on Render to refresh all Plaid items + Gemini; alert if any source is stale >48h.
5. **UI:** Add a “**Total net cash & investments**” tile + per‑institution breakdown.

If you want, I can draft a **starter Node/TS service** (routes + cron + DB upserts) tailored to your Render stack and hand you the exact `curl`/fetch patterns for:

* Plaid `/accounts/balance/get` (TD/Chase) ([Plaid][4])
* Plaid Investments `/investments/holdings/get` (if Fidelity via Akoya is available to your aggregator) ([Plaid][10])
* Gemini `/v1/balances`. ([Gemini Docs][5])

**Bottom line:** For **TD + Chase**, use **Plaid OAuth**; for **Gemini**, use **Gemini’s API**; for **Fidelity**, try aggregator via **Akoya** and keep a **CSV** fallback. That gets you clean **daily balances** with minimal maintenance and zero credential risk.

[1]: https://www.chase.com/digital/data-sharing?utm_source=chatgpt.com "Our secure API"
[2]: https://media.chase.com/news/plaid-signs-data-agreement-with-jpmc?utm_source=chatgpt.com "Plaid Signs Data Agreement with JPMorgan Chase"
[3]: https://stackoverflow.com/questions/74418870/plaid-link-issue-when-connecting-to-chase-you-may-need-to-update-your-app-in-or?utm_source=chatgpt.com "Plaid Link Issue when connecting to Chase \"You may need ..."
[4]: https://plaid.com/docs/api/products/balance/?utm_source=chatgpt.com "Balance - API"
[5]: https://docs.gemini.com/rest/fund-management "Fund Management - REST API - Gemini Crypto Exchange"
[6]: https://newsroom.fidelity.com/pressreleases/fidelity-takes-steps-to-address-screen-scraping/s/2f33bc18-f16d-4b66-9868-626ada9ba32b?utm_source=chatgpt.com "Fidelity Takes Steps to Address Screen Scraping"
[7]: https://akoya.com/?utm_source=chatgpt.com "Open Finance Solutions | Akoya - Secure, Reliable Data Sharing"
[8]: https://stories.td.com/ca/en/news/2023-12-14-td-bank-group-and-plaid-enter-into-north-american-data-acces?utm_source=chatgpt.com "TD Bank Group and Plaid enter into North American data ..."
[9]: https://plaid.com/institutions/td-bank/?utm_source=chatgpt.com "TD Bank API & Data Solutions"
[10]: https://plaid.com/docs/api/products/investments/?utm_source=chatgpt.com "API - Investments"
[11]: https://www.fidelity.com/webcontent/ap002390-mlo-content/19.09/help/learn_balances.shtml?utm_source=chatgpt.com "Fidelity.com Help - Balances"
[12]: https://plaid.com/docs/link/oauth/?utm_source=chatgpt.com "Link - OAuth guide"
[13]: https://teller.io/?utm_source=chatgpt.com "Teller - The API for your bank account"
[14]: https://plaid.com/docs/api/products/transactions/?utm_source=chatgpt.com "API - Transactions"
