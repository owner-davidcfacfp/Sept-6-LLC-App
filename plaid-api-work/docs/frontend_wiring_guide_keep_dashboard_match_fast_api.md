# Frontend Wiring Guide — Keep Dashboard, Match FastAPI

Target: existing dashboard UI stays the same. Add a thin client to talk to the FastAPI backend with `X-API-Key`. Implement Plaid Link flow and balances fetch. No JWT, no cookies, no Node server.

## Constraints
- Do not change backend routes or auth.
- Every request includes header `X-API-Key: <value>`.
- Use only these endpoints:
  - `POST /plaid/link-token` → `{ link_token }`
  - `POST /plaid/exchange` with body `{ public_token, institution?: string | null }`
  - `POST /plaid/refresh-balances` → `{ accounts: Account[] }`
  - Optional: `GET/PUT /state`
- Keep dashboard HTML/CSS/UX as-is.
- Plain JS. No build step. Tailwind via CDN allowed.

---

## File to add
Create **`llc-finance-app/js/backend-glue.js`**

```js
// Minimal client for FastAPI. No cookies. Sends X-API-Key on each call.
// Reads apiBase and apiKey from sessionStorage or from URL params (?apiBase=&apiKey=)
(function(){
  function param(k){ return new URLSearchParams(location.search).get(k) || ''; }
  const apiBase = sessionStorage.getItem('apiBase') || param('apiBase') || '';
  const apiKey  = sessionStorage.getItem('apiKey')  || param('apiKey')  || '';
  if (param('apiBase')) sessionStorage.setItem('apiBase', param('apiBase').replace(/\/+$/,''));
  if (param('apiKey'))  sessionStorage.setItem('apiKey', param('apiKey'));

  function getBase(){ const b = sessionStorage.getItem('apiBase') || apiBase; if(!b) throw new Error('API base not set'); return b; }
  function getKey(){  const k = sessionStorage.getItem('apiKey')  || apiKey;  if(!k) throw new Error('API key not set');  return k; }

  async function api(path, init={}){
    const res = await fetch(getBase() + path, {
      ...init,
      headers: { 'Content-Type':'application/json','X-API-Key': getKey(), ...(init.headers||{}) }
    });
    if(!res.ok){ throw new Error(`${res.status} ${await res.text().catch(()=>res.statusText)}`); }
    const text = await res.text(); return text ? JSON.parse(text) : {};
  }

  // Backend routes (exact names)
  async function getState(){ return api('/state'); }
  async function putState(state){ return api('/state',{method:'PUT',body:JSON.stringify({state})}); }
  async function createLinkToken(){ return api('/plaid/link-token',{method:'POST'}); }
  async function exchangePublicToken(public_token,institution){
    return api('/plaid/exchange',{method:'POST',body:JSON.stringify({ public_token, institution: institution||null })});
  }
  async function refreshBalances(){ return api('/plaid/refresh-balances',{method:'POST'}); }

  window.Backend = { api, getState, putState, createLinkToken, exchangePublicToken, refreshBalances };
})();
```

---

## HTML change
In your dashboard HTML, include the glue **before** your existing app script.

```html
<script src="js/backend-glue.js"></script>
<script src="js/app.js"></script>
```

No other HTML changes required.

---

## `app.js` integration
Append this to the end of your existing `js/app.js`. Do not remove your current UI code.

```js
document.addEventListener('DOMContentLoaded', () => {
  // Optional: redirect to an access page if storing values elsewhere
  if (!sessionStorage.getItem('apiBase') || !sessionStorage.getItem('apiKey')) {
    // If you prefer a dedicated access page, uncomment:
    // location.href = 'access.html';
    // Otherwise, pass ?apiBase=&apiKey= once and they will be cached for the session.
  }

  const connectBtn = document.getElementById('connectBankBtn');
  const refreshBtn = document.getElementById('refreshBtn');

  async function pullAndRenderBalances() {
    const data = await Backend.refreshBalances(); // { accounts: [...] }
    // Expose to your existing render code if it expects a global
    window.accountsDataFromBackend = data.accounts || data || [];

    // If you already have a renderer, hook it here:
    if (typeof window.updateDashboardBalancesFromBackend === 'function') {
      window.updateDashboardBalancesFromBackend(window.accountsDataFromBackend);
      return;
    }

    // Fallback render if no custom renderer exists
    const list = document.getElementById('accountsList');
    const empty = document.getElementById('emptyMsg');
    if (!list) return;
    list.innerHTML = '';
    const accs = window.accountsDataFromBackend;
    if (!accs.length && empty) empty.classList.remove('hidden');
    if (accs.length && empty) empty.classList.add('hidden');
    accs.forEach(a => {
      const li = document.createElement('li');
      li.className = 'py-2 flex items-center justify-between';
      const name = a.name || a.official_name || a.plaid_account_id || 'Account';
      const amt  = a.current ?? a.balances?.current ?? 0;
      const cur  = a.currency || a.balances?.iso_currency_code || '';
      li.innerHTML = `<span class="text-slate-800">${name}</span>
                      <span class="font-mono text-slate-700">${amt} ${cur}</span>`;
      list.appendChild(li);
    });
  }

  async function startPlaidLink() {
    if (!connectBtn) return;
    const original = connectBtn.textContent;
    connectBtn.disabled = true; connectBtn.textContent = 'Connecting…';
    try {
      const { link_token } = await Backend.createLinkToken();
      const handler = Plaid.create({
        token: link_token,
        onSuccess: async (public_token, metadata) => {
          await Backend.exchangePublicToken(public_token, metadata?.institution?.name || null);
          await pullAndRenderBalances();
        },
        onExit: () => {}
      });
      handler.open();
    } catch (e) {
      alert('Plaid Link failed: ' + e.message);
    } finally {
      connectBtn.disabled = false; connectBtn.textContent = original;
    }
  }

  if (connectBtn) connectBtn.addEventListener('click', startPlaidLink);
  if (refreshBtn) refreshBtn.addEventListener('click', pullAndRenderBalances);

  // Initial attempt to load cached or freshly fetched balances
  pullAndRenderBalances().catch(()=>{});
});
```

---

## One-time setup for local testing
- Ensure backend is running with env vars set: `API_KEY`, `FERNET_KEY`, `PLAID_CLIENT_ID`, `PLAID_SECRET`, `PLAID_ENV`, `DATABASE_URL`.
- Open the dashboard with URL params once to seed the session:
  - `index.html?apiBase=http://localhost:8000&apiKey=YOUR_KEY`
  - The script saves both to `sessionStorage` for the session.

Optional: build a small `access.html` to input and store these two values. Not required if you use URL params once per session.

---

## Acceptance tests
1) **Health**: backend `/healthz` returns 200.
2) **Link token**: `POST /plaid/link-token` with header `X-API-Key` returns `{ link_token }`.
3) **Plaid Link**: clicking **Connect bank** launches Plaid, completes sandbox flow without app cookies.
4) **Exchange**: on success, frontend posts to `/plaid/exchange` with `{ public_token, institution }`.
5) **Balances**: frontend calls `POST /plaid/refresh-balances` and renders a non-empty accounts list when available.
6) **Refresh**: clicking **Refresh balances** updates amounts.
7) **Key handling**: `sessionStorage` contains `apiBase` and `apiKey`, and no cookies are created by the app.

---

## Troubleshooting
- 401/403: wrong or missing `X-API-Key`. Re-seed via `?apiBase=&apiKey=`.
- CORS: we do not use cookies, so credentialed CORS is not required. If hosting on a different origin, ensure backend allows the origin.
- Plaid Link fails to open: missing or expired `link_token`. Retry `createLinkToken()`.
- Empty accounts: bank not linked yet or balances cache empty. Click **Refresh balances** after successful exchange.

---

## Security notes
- Never commit the API key. Provide it via URL param once per session or via an access page that writes to `sessionStorage`.
- Rotate the API key on compromise.
- Keep `/state` payloads small if you later use it for UI metadata.

