Awesome—let’s ship a small, reviewer-friendly **front-end sign-up/login + Plaid Link PoC** you can stand up fast. I’ll give you:

1. A minimal **React (Vite + TS)** front end with:

   * Sign-Up and Login pages (email + password)
   * A dashboard that shows the **consent text + descriptive “Why we ask you to connect” link + Plaid Link button**
   * A simple balances view so you can screenshot “data showing”
2. **Express/Node** backend adds for auth (if you don’t already have them)
3. A **screenshot shot-list** (exact captions to paste under each attachment)

This keeps your Plaid reviewer happy: explicit consent, policies linked, OAuth Link, and no credential collection.

---

# 0) Quick plan (why we’re doing each step)

* Plaid specifically wants to see **user sign-up/login** → **consent text** → **Plaid Link launch** → **successful return** → **data displayed**.
* That’s exactly what the code below gives you—with the **least** moving parts.

---

# 1) Front-end (React + Vite, TypeScript)

## 1.1 Create the app & install deps

```bash
# in your frontend folder
npm create vite@latest plaid-poc -- --template react-ts
cd plaid-poc
npm i react-plaid-link
# if your API runs on a different origin: we’ll use credentials: 'include'
```

## 1.2 `src/api.ts` (tiny fetch helper)

```ts
export async function api(path: string, init: RequestInit = {}) {
  const res = await fetch(path, {
    credentials: 'include',
    headers: { 'Content-Type': 'application/json', ...(init.headers||{}) },
    ...init
  });
  if (!res.ok) throw new Error(`${init.method||'GET'} ${path} failed`);
  return res.json();
}
```

## 1.3 `src/pages/Signup.tsx`

```tsx
import { useState } from 'react';
import { api } from '../api';

export default function Signup() {
  const [email,setEmail] = useState('');
  const [password,setPassword] = useState('');
  const [msg,setMsg] = useState('');

  async function submit() {
    try {
      await api('/auth/signup',{ method:'POST', body: JSON.stringify({ email, password }) });
      window.location.href = '/dashboard';
    } catch (e:any) { setMsg(e.message || 'Sign-up failed'); }
  }

  return (
    <div style={{maxWidth:420,margin:'4rem auto',fontFamily:'system-ui'}}>
      <h1>Create account</h1>
      <input placeholder="Email" value={email} onChange={e=>setEmail(e.target.value)} style={{width:'100%',padding:8,margin:'8px 0'}}/>
      <input placeholder="Password" type="password" value={password} onChange={e=>setPassword(e.target.value)} style={{width:'100%',padding:8,margin:'8px 0'}}/>
      <button onClick={submit} style={{padding:'8px 12px'}}>Sign up</button>
      {msg && <p style={{color:'crimson'}}>{msg}</p>}
      <p style={{marginTop:16}}>Already have an account? <a href="/login">Log in</a></p>
    </div>
  );
}
```

## 1.4 `src/pages/Login.tsx`

```tsx
import { useState } from 'react';
import { api } from '../api';

export default function Login() {
  const [email,setEmail] = useState('');
  const [password,setPassword] = useState('');
  const [msg,setMsg] = useState('');

  async function submit() {
    try {
      await api('/auth/login',{ method:'POST', body: JSON.stringify({ email, password }) });
      window.location.href = '/dashboard';
    } catch (e:any) { setMsg(e.message || 'Login failed'); }
  }

  return (
    <div style={{maxWidth:420,margin:'4rem auto',fontFamily:'system-ui'}}>
      <h1>Log in</h1>
      <input placeholder="Email" value={email} onChange={e=>setEmail(e.target.value)} style={{width:'100%',padding:8,margin:'8px 0'}}/>
      <input placeholder="Password" type="password" value={password} onChange={e=>setPassword(e.target.value)} style={{width:'100%',padding:8,margin:'8px 0'}}/>
      <button onClick={submit} style={{padding:'8px 12px'}}>Log in</button>
      {msg && <p style={{color:'crimson'}}>{msg}</p>}
      <p style={{marginTop:16}}>No account? <a href="/signup">Create one</a></p>
    </div>
  );
}
```

## 1.5 `src/components/ConsentTooltip.tsx`

```tsx
export default function ConsentTooltip() {
  return (
    <div style={{background:'#fff',border:'1px solid #ddd',padding:12,borderRadius:8,boxShadow:'0 4px 12px rgba(0,0,0,0.08)',width:320}}>
      <p>
        When you connect an account, <strong>[COMPANY]</strong> uses Plaid to securely access your balances and
        transactions. We never see or store your banking credentials.
      </p>
      <p style={{marginTop:8}}>
        See our <a href="/privacy" target="_blank">Privacy Policy</a> and{' '}
        <a href="https://plaid.com/legal/#end-user-privacy-policy" target="_blank">Plaid’s End User Privacy Policy</a>.
      </p>
    </div>
  );
}
```

## 1.6 `src/components/ConnectBankSection.tsx`

```tsx
import { useEffect, useState, useCallback } from 'react';
import { usePlaidLink } from 'react-plaid-link';
import ConsentTooltip from './ConsentTooltip';
import { api } from '../api';

export default function ConnectBankSection() {
  const [linkToken, setLinkToken] = useState<string>();
  const [showTip, setShowTip] = useState(false);
  const [accounts, setAccounts] = useState<any[]>([]);

  useEffect(() => {
    (async () => {
      const j = await api('/plaid/create_link_token', { method:'POST' });
      setLinkToken(j.link_token);
    })();
  }, []);

  const onSuccess = useCallback(async (public_token: string) => {
    await api('/plaid/exchange_public_token', {
      method:'POST', body: JSON.stringify({ public_token })
    });
    const data = await api('/balances');
    setAccounts(data.accounts || []);
  }, []);

  const { open, ready } = usePlaidLink({ token: linkToken, onSuccess });

  return (
    <section style={{fontFamily:'system-ui'}}>
      <h2 style={{fontSize:18,fontWeight:600}}>We use Plaid for secure bank connections.</h2>
      <p style={{fontSize:14, color:'#444', maxWidth:720}}>
        By clicking “Connect Bank,” you authorize [COMPANY] to connect to your financial institution via Plaid to securely
        access your balances and transactions for display in your dashboard. We do not collect or store your banking
        credentials. See our <a href="/privacy" target="_blank">Privacy Policy</a> and{' '}
        <a href="https://plaid.com/legal/#end-user-privacy-policy" target="_blank">Plaid’s End User Privacy Policy</a>.
      </p>

      <div style={{position:'relative', display:'inline-block', marginTop:8}}>
        <button disabled={!ready} onClick={() => open()}
          style={{padding:'10px 14px', background:'#16a34a', color:'#fff', borderRadius:8, border:'none', cursor:'pointer'}}>
          Connect your bank securely with Plaid
        </button>
        <button type="button" onClick={() => setShowTip(!showTip)}
          style={{marginLeft:10, background:'transparent', color:'#2563eb', textDecoration:'underline', border:'none', cursor:'pointer', fontSize:13}}>
          Why we ask you to connect
        </button>
        {showTip && (<div style={{position:'absolute', top:'110%', left:0, zIndex:10}}><ConsentTooltip/></div>)}
      </div>

      {accounts.length > 0 && (
        <div style={{marginTop:24}}>
          <h3>Connected accounts</h3>
          <ul>
            {accounts.map((a:any) => (
              <li key={a.account_id}>
                {a.name || a.official_name || a.account_id} — {a.balances?.current} {a.balances?.iso_currency_code || ''}
              </li>
            ))}
          </ul>
        </div>
      )}
    </section>
  );
}
```

## 1.7 `src/pages/Dashboard.tsx`

```tsx
import { useEffect, useState } from 'react';
import { api } from '../api';
import ConnectBankSection from '../components/ConnectBankSection';

export default function Dashboard() {
  const [me, setMe] = useState<any>(null);
  useEffect(() => { (async () => { try { setMe(await api('/auth/me')); } catch { window.location.href='/login'; } })(); }, []);
  if (!me) return null;

  return (
    <div style={{maxWidth:900, margin:'2rem auto', fontFamily:'system-ui'}}>
      <h1>Welcome, {me.email}</h1>
      <ConnectBankSection />
    </div>
  );
}
```

## 1.8 `src/App.tsx`

```tsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import Signup from './pages/Signup';
import Login from './pages/Login';
import Dashboard from './pages/Dashboard';

export default function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Signup/>}/>
        <Route path="/signup" element={<Signup/>}/>
        <Route path="/login" element={<Login/>}/>
        <Route path="/dashboard" element={<Dashboard/>}/>
      </Routes>
    </BrowserRouter>
  );
}
```

## 1.9 `src/main.tsx`

Replace Vite’s default with:

```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
ReactDOM.createRoot(document.getElementById('root')!).render(<React.StrictMode><App/></React.StrictMode>);
```

> **Why this matters:** This gives Plaid reviewers a full, visible path: user signs up → logs in → sees **consent language + policies** → launches **Plaid Link (OAuth)** → returns → sees **balances**.

---

# 2) Backend adds (Express/Node)

If you **already** implemented these, skip. Otherwise, drop these in.

## 2.1 Deps

```bash
npm i express cookie-parser jsonwebtoken bcryptjs cors pg plaid dotenv
```

## 2.2 Minimal users table (Postgres)

```sql
create table if not exists users (
  id bigserial primary key,
  email text unique not null,
  pass_hash text not null,
  created_at timestamptz not null default now()
);
```

## 2.3 Server bootstrap (CORS, cookies, auth)

```ts
import 'dotenv/config';
import express from 'express';
import cors from 'cors';
import cookieParser from 'cookie-parser';
import jwt from 'jsonwebtoken';
import bcrypt from 'bcryptjs';
import { Pool } from 'pg';
import { Configuration, PlaidApi, PlaidEnvironments } from 'plaid';

const app = express();
const db = new Pool({ connectionString: process.env.DATABASE_URL, ssl:{rejectUnauthorized:false} });

app.use(cors({
  origin: ['http://localhost:5173','https://dashboard.dvo88.com'], // add your front-end origin(s)
  credentials: true
}));
app.use(express.json());
app.use(cookieParser());

// auth helpers
function setSession(res:any, user:any){
  const token = jwt.sign({ uid:user.id, email:user.email }, process.env.JWT_SECRET!, { expiresIn: '7d' });
  const secure = process.env.NODE_ENV === 'production';
  res.cookie('session', token, { httpOnly:true, secure, sameSite: secure ? 'none' : 'lax', maxAge: 7*24*3600*1000 });
}
function requireAuth(req:any,res:any,next:any){
  try {
    const token = req.cookies.session;
    const payload:any = jwt.verify(token, process.env.JWT_SECRET!);
    (req as any).user = payload; next();
  } catch { return res.status(401).json({error:'auth required'}); }
}

// users
app.post('/auth/signup', async (req,res) => {
  const { email, password } = req.body || {};
  if(!email || !password) return res.status(400).json({error:'email/password required'});
  const hash = await bcrypt.hash(password, 12);
  await db.query(`insert into users(email, pass_hash) values($1,$2) on conflict do nothing`, [email, hash]);
  const u = await db.query(`select id,email from users where email=$1`, [email]);
  setSession(res, u.rows[0]); res.json({ ok:true });
});
app.post('/auth/login', async (req,res) => {
  const { email, password } = req.body || {};
  const u = await db.query(`select id,email,pass_hash from users where email=$1`, [email]);
  if(!u.rowCount) return res.status(401).json({error:'invalid'});
  const ok = await bcrypt.compare(password, u.rows[0].pass_hash);
  if(!ok) return res.status(401).json({error:'invalid'});
  setSession(res, { id:u.rows[0].id, email:u.rows[0].email }); res.json({ ok:true });
});
app.get('/auth/me', requireAuth, (req:any,res)=> res.json({ id:req.user.uid, email:req.user.email }));
app.post('/auth/logout', (_req,res)=> { res.clearCookie('session'); res.json({ok:true}); });

// Plaid (assumes you already created plaid_items table & encryption as we discussed earlier)
const cfg = new Configuration({
  basePath: PlaidEnvironments[process.env.PLAID_ENV as 'sandbox'|'development'|'production'],
  baseOptions: { headers: { 'PLAID-CLIENT-ID': process.env.PLAID_CLIENT_ID!, 'PLAID-SECRET': process.env.PLAID_SECRET! } }
});
const plaid = new PlaidApi(cfg);

app.post('/plaid/create_link_token', requireAuth, async (req:any,res:any)=>{
  const r = await plaid.linkTokenCreate({
    user: { client_user_id: String(req.user.uid) },
    client_name: '[COMPANY] Personal Finance',
    products: ['transactions'], country_codes:['US'], language:'en',
    redirect_uri: process.env.PLAID_REDIRECT_URI
  });
  res.json({ link_token: r.data.link_token });
});

// You already have /plaid/exchange_public_token and /balances from prior work

app.listen(process.env.PORT || 8080, ()=> console.log('API up'));
```

> **Gotcha to avoid:** Cookies with `SameSite=None` **require HTTPS** and `secure: true`. In local dev, the code above sets `secure:false` and `SameSite:lax` so it works on `http://localhost:5173`.

---

# 3) Screenshot shot-list (exact captions)

Use these exact captions in your Plaid email so the reviewer can follow the flow.

1. **Sign-Up (email + password, then create account)**
   *New user creates an account. Passwords are hashed; session is an HttpOnly cookie.*

2. **Login (existing user authenticates)**
   *User logs in and lands on the dashboard (protected route).*

3. **Dashboard with consent text (before connecting)**
   *Consent statement visible; links to our Privacy Policy and Plaid’s End User Privacy Policy; “Connect your bank securely with Plaid” button; “Why we ask you to connect” link.*

4. **Consent tooltip expanded**
   *Descriptive consent details shown inline; reviewers can read without leaving the page.*

5. **Plaid Link (TD Bank OAuth window)**
   *App launches Plaid Link; bank authentication handled by Plaid; we do not see or store credentials.*

6. **Return to app (connected) + balances view**
   *After successful link and token exchange (server-side), the dashboard displays connected accounts and balances.*

---

# 4) How to run (quick)

* **Backend**: set `PLAID_CLIENT_ID`, `PLAID_SECRET`, `PLAID_ENV=sandbox`, `PLAID_REDIRECT_URI`, `JWT_SECRET`, `DATABASE_URL` (and your encryption key if you wired token encryption already). Start server.
* **Frontend**: set `VITE` proxy or use absolute paths; run `npm run dev`.
* **OAuth**: add your `PLAID_REDIRECT_URI` in the Plaid Dashboard **exactly** (scheme/host/path must match).

---

# 5) Why this satisfies Plaid (no fluff)

* Explicit **consent at point of action** + links to both policies
* **OAuth Link** used (no password capture)
* **Token exchange server-side**; tokens stored encrypted at rest (your backend)
* **Working PoC** with data displayed (balances), producing clean reviewer screenshots

If you want, I can wrap this into a single **hand-off MD** or a small **zip** with the file tree you can drop into your repo.
yes