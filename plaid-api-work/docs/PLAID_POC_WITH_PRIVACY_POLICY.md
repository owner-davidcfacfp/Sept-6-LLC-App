
# Plaid PoC Guide + Privacy Policy

> Hand‑off doc for dev. Contains:  
> 1. Full proof‑of‑concept (PoC) frontend + backend code and reviewer instructions  
> 2. Privacy Policy draft tailored to our app and Plaid requirements  

---

## Part A. Plaid PoC Frontend + Backend Implementation Guide

### 0. Why This Matters
Plaid reviewers expect to see a **realistic user journey**:  
1. User signs up / logs in  
2. Explicit consent text appears in the dashboard (with links to Privacy Policies)  
3. User clicks “Connect Bank” → Plaid Link OAuth flow opens  
4. User authenticates via Plaid (we never see credentials)  
5. Return to app → user sees connected account balances  

This doc provides frontend (React) and backend (Express/Node) code to deliver that flow.

---

### 1. Frontend (React + Vite + TypeScript)

(…full code and instructions from `PLAID_POC_GUIDE.md` here …)

---

### 2. Backend (Express/Node)

(…backend deps, DB schema, sample routes as in `PLAID_POC_GUIDE.md` …)

---

### 3. Screenshot Captions for Plaid Review

1. **Sign‑Up (email + password)** – user creates account.  
2. **Login** – user logs in and lands on dashboard.  
3. **Dashboard with consent text** – consent statement + Privacy Policy links visible.  
4. **Consent tooltip expanded** – shows Plaid explanation.  
5. **Plaid Link (TD Bank OAuth)** – bank auth handled by Plaid.  
6. **Return + balances view** – user sees connected accounts & balances.

---

### 4. Running Locally
- Backend on port 8080, frontend Vite dev server on 5173.  
- Configure Plaid Sandbox env vars: `PLAID_CLIENT_ID`, `PLAID_SECRET`, `PLAID_ENV=sandbox`, `PLAID_REDIRECT_URI`.  
- Add same redirect URI in Plaid Dashboard (must match exactly).  
- Test flow end‑to‑end, then screenshot each step.

---

### 5. Why Plaid Will Approve
- Explicit **consent text** with both Privacy Policies linked.  
- **OAuth flow** only; no credential handling.  
- Access tokens stored server‑side, encrypted.  
- Screenshots demonstrate full PoC journey.

---

## Part B. Privacy Policy Draft

**Effective Date:** [DATE]

### 1. Who We Are
[COMPANY] (“we,” “us,” “our”) provides a personal finance dashboard that allows users to securely connect their own financial accounts using Plaid. Our service helps users view balances and recent transactions in a single dashboard.

### 2. Information We Collect
**From you:** Email and password (hashed before storage), session cookies.  
**From Plaid:** With your consent, we access account identifiers, balances, and recent transactions.  
**We do not collect:** Bank usernames or passwords.

### 3. How We Use Information
- Provide the dashboard experience  
- Maintain your account and authenticate sessions  
- Troubleshoot errors and improve security  
- Comply with legal obligations  
- Never sell or rent data

### 4. How We Share Information
Only with:  
- Service providers (hosting, DB) under contract  
- Plaid (for account connectivity)  
- Law/regulators if required

### 5. Plaid
By using our service, you authorize Plaid and [COMPANY] to access financial info from your chosen institutions. Plaid’s Privacy Policy: [https://plaid.com/legal/#end-user-privacy-policy](https://plaid.com/legal/#end-user-privacy-policy).

### 6. Security
- **Encryption in transit:** TLS 1.2+  
- **Encryption at rest:** AES-256-GCM for Plaid tokens & DB storage  
- **Access control:** Tokens only stored server-side

### 7. Data Retention & Deletion
We retain data only as long as needed to provide services or as required by law. You can request deletion at [CONTACT EMAIL].

### 8. Your Rights
Depending on jurisdiction, you may request access, correction, or deletion. Contact us at [CONTACT EMAIL].

### 9. Children’s Privacy
Not directed to individuals under 18.

### 10. Changes
We may update this Privacy Policy; changes will be posted with a new effective date.

### 11. Contact
[COMPANY NAME]  
[POSTAL ADDRESS]  
Email: [CONTACT EMAIL]

---

## Reviewer Notes (for Plaid Submission)
- Only Plaid OAuth is used; no bank credentials ever collected.  
- Explicit consent copy + Privacy Policy links visible in the UI.  
- Tokens encrypted at rest, never exposed client-side.  
- Prototype is currently for internal/personal use, not public distribution.  
- Building toward a consumer-ready product with strong security foundations.

---
