## Instructions for Developer: Integrating User Authentication and Real Plaid API

**Overall Goal:**
To enhance the `Recordkeeping Technology` project by implementing a robust user authentication system (sign-up/login) and integrating the frontend with the actual Plaid API via the existing Python/FastAPI backend. This will move the project beyond simulated Plaid connections and provide a secure user experience, crucial for Plaid review.

**Existing Framework:**
*   **Frontend (`llc-finance-app`):** Plain HTML, Tailwind CSS, Vanilla JavaScript (`js/app.js`).
*   **Backend (`llc-finance-api`):** Python, FastAPI, PostgreSQL (via `db.py`).

---

### Phase 1: Backend Implementation (Python/FastAPI)

The "Awesome" document provides Express/Node.js examples. You will need to implement the equivalent functionality in Python using FastAPI, leveraging your existing `app/db.py`, `app/main.py`, `app/models.py`, `app/schemas.py`, and `app/plaid_client.py`.

**1. Database Schema for Users:**
   *   **Action:** Add a `users` table to your PostgreSQL database.
   *   **Guidance:** In `app/models.py`, define a SQLAlchemy model for `User` (or adapt your existing `db.py` to create the table directly if not using SQLAlchemy ORM).
   *   **Schema (Conceptual, adapt to your ORM/DB layer):**
     ```sql
     CREATE TABLE IF NOT EXISTS users (
       id SERIAL PRIMARY KEY,
       email VARCHAR(255) UNIQUE NOT NULL,
       hashed_password VARCHAR(255) NOT NULL,
       created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
     );
     ```
   *   **Action:** Add a `plaid_items` table to store Plaid `access_token`s securely, linked to users.
   *   **Schema (Conceptual):**
     ```sql
     CREATE TABLE IF NOT EXISTS plaid_items (
       id SERIAL PRIMARY KEY,
       user_id INTEGER NOT NULL REFERENCES users(id),
       item_id VARCHAR(255) UNIQUE NOT NULL,
       access_token BYTEA NOT NULL, -- Store encrypted access token
       created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
     );
     ```
   *   **Note:** Ensure `access_token` is encrypted at rest. Your `security.py` or `plaid_client.py` should handle encryption/decryption.

**2. Environment Variables:**
   *   **Action:** Update your `.env.example` and `.env` files in `llc-finance-api`.
   *   **Add:**
     ```
     JWT_SECRET="YOUR_VERY_LONG_AND_RANDOM_SECRET_KEY"
     PLAID_REDIRECT_URI="http://localhost:8000/plaid/oauth" # Or your deployed frontend URL
     ```
   *   **Note:** `JWT_SECRET` is crucial for securing user sessions. Generate a strong, random string.

**3. Authentication Endpoints (`app/main.py`):**
   *   **Action:** Implement FastAPI routes for user authentication.
   *   **Guidance:**
     *   Use `app/schemas.py` to define `UserCreate` and `UserLogin` Pydantic models.
     *   Use `app/security.py` for password hashing (e.g., `bcrypt` or `passlib`) and JWT token creation/verification.
     *   Implement `set_session` and `require_auth` logic using FastAPI dependencies and secure `HTTPOnly` cookies.
   *   **Endpoints to Implement:**
     *   `POST /auth/signup`:
         *   Accepts `email` and `password`.
         *   Hashes password, stores user in `users` table.
         *   Creates a JWT session token and sets it as an `HttpOnly` cookie.
         *   Returns success.
     *   `POST /auth/login`:
         *   Accepts `email` and `password`.
         *   Verifies password against hash in DB.
         *   Creates a JWT session token and sets it as an `HttpOnly` cookie.
         *   Returns success.
     *   `GET /auth/me` (Protected):
         *   Requires authentication (check JWT cookie).
         *   Returns current user's `id` and `email`.
     *   `POST /auth/logout`:
         *   Clears the session cookie.
         *   Returns success.

**4. Plaid Endpoints (`app/main.py` and `app/plaid_client.py`):**
   *   **Action:** Implement FastAPI routes for Plaid integration.
   *   **Guidance:**
     *   Your existing `app/plaid_client.py` should contain the Plaid API client setup.
     *   Ensure `PLAID_ENV`, `PLAID_CLIENT_ID`, `PLAID_SECRET` are correctly loaded from environment variables.
   *   **Endpoints to Implement:**
     *   `POST /plaid/create_link_token` (Protected):
         *   Requires authentication.
         *   Calls Plaid API to create a `link_token` for the authenticated user.
         *   Returns the `link_token`.
     *   `POST /plaid/exchange_public_token` (Protected):
         *   Requires authentication.
         *   Accepts `public_token` from the frontend.
         *   Calls Plaid API to exchange `public_token` for `access_token`.
         *   **Encrypts** the `access_token` and stores it in the `plaid_items` table, linked to the user.
         *   Returns success.
     *   `GET /balances` (Protected):
         *   Requires authentication.
         *   Retrieves the user's encrypted `access_token` from `plaid_items`.
         *   **Decrypts** the `access_token`.
         *   Calls Plaid API to fetch account balances using the `access_token`.
         *   Returns the balances.

**5. CORS Configuration (`app/main.py`):**
   *   **Action:** Ensure your FastAPI CORS middleware is correctly configured.
   *   **Guidance:**
     *   `Allow-Credentials` must be `True` for cookies to work across origins.
     *   `Allow-Origins` should include your frontend's development URL (e.g., `http://localhost:8000` if running locally, or `http://localhost:5173` if using Vite's default) and your deployed frontend URL.
     *   Example:
       ```python
       from fastapi.middleware.cors import CORSMiddleware

       app.add_middleware(
           CORSMiddleware,
           allow_origins=["http://localhost:8000", "https://your-deployed-frontend.render.com"], # Add your frontend origins
           allow_credentials=True,
           allow_methods=["*"],
           allow_headers=["*"],
       )
       ```

---

### Phase 2: Frontend Implementation (HTML/CSS/JS)

You will adapt the concepts from the "Awesome" document's React components into vanilla JavaScript and HTML, maintaining your existing Tailwind CSS styling.

**1. User Authentication Pages/Sections:**
   *   **Action:** Create new HTML files for `login.html` and `signup.html` in `llc-finance-app/`.
   *   **Guidance:**
     *   Each file should be a complete HTML document, including your existing Tailwind CSS setup.
     *   Design simple forms for email and password input, and a submit button.
     *   Include links between `login.html` and `signup.html` (e.g., "Already have an account? Log in").
     *   **Example `signup.html` (structure):**
       ```html
       <!DOCTYPE html>
       <html lang="en">
       <head>
           <meta charset="UTF-8">
           <meta name="viewport" content="width=device-width, initial-scale=1.0">
           <title>Sign Up</title>
           <script src="https://cdn.tailwindcss.com"></script>
           <link rel="stylesheet" href="css/style.css"> <!-- If you have a custom CSS file -->
       </head>
       <body class="flex items-center justify-center min-h-screen bg-slate-50">
           <div class="bg-white p-8 rounded-xl shadow-lg border border-slate-200 max-w-md w-full">
               <h1 class="text-2xl font-bold text-slate-800 mb-6 text-center">Create Account</h1>
               <form id="signup-form">
                   <div class="mb-4">
                       <label for="email" class="block text-slate-700 text-sm font-bold mb-2">Email:</label>
                       <input type="email" id="email" name="email" class="shadow appearance-none border rounded w-full py-2 px-3 text-slate-700 leading-tight focus:outline-none focus:shadow-outline" required>
                   </div>
                   <div class="mb-6">
                       <label for="password" class="block text-slate-700 text-sm font-bold mb-2">Password:</label>
                       <input type="password" id="password" name="password" class="shadow appearance-none border rounded w-full py-2 px-3 text-slate-700 mb-3 leading-tight focus:outline-none focus:shadow-outline" required>
                   </div>
                   <button type="submit" class="bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline w-full">Sign Up</button>
                   <p id="auth-message" class="text-red-500 text-xs italic mt-4 text-center"></p>
               </form>
               <p class="text-center text-slate-600 text-sm mt-4">Already have an account? <a href="login.html" class="text-blue-600 hover:underline">Log in</a></p>
           </div>
           <script src="js/auth.js"></script> <!-- New JS file for auth logic -->
       </body>
       </html>
       ```
   *   **Action:** Create a new JavaScript file `llc-finance-app/js/auth.js` to handle the authentication logic for `login.html` and `signup.html`.
   *   **Guidance:**
     *   In `auth.js`, add event listeners for form submissions.
     *   Use `fetch` to send `POST` requests to your backend's `/auth/signup` and `/auth/login` endpoints.
     *   On successful authentication, redirect the user to `index.html` (your dashboard).
     *   Handle errors and display messages to the user.
     *   **Crucially, ensure `credentials: 'include'` is set in your `fetch` requests** for cookies to be sent and received.
     *   **Example `js/auth.js` (conceptual):**
       ```javascript
       document.addEventListener('DOMContentLoaded', () => {
           const signupForm = document.getElementById('signup-form');
           const loginForm = document.getElementById('login-form');
           const authMessage = document.getElementById('auth-message');

           const handleAuth = async (event, endpoint) => {
               event.preventDefault();
               authMessage.textContent = ''; // Clear previous messages

               const email = event.target.elements.email.value;
               const password = event.target.elements.password.value;

               try {
                   const response = await fetch(`http://localhost:8000/auth/${endpoint}`, { // Adjust backend URL
                       method: 'POST',
                       headers: { 'Content-Type': 'application/json' },
                       body: JSON.stringify({ email, password }),
                       credentials: 'include' // IMPORTANT for cookies
                   });

                   if (!response.ok) {
                       const errorData = await response.json();
                       throw new Error(errorData.error || `Authentication failed: ${response.status}`);
                   }

                   // Success: Redirect to dashboard
                   window.location.href = 'index.html';

               } catch (error) {
                   console.error('Auth error:', error);
                   authMessage.textContent = error.message;
               }
           };

           if (signupForm) {
               signupForm.addEventListener('submit', (e) => handleAuth(e, 'signup'));
           }
           if (loginForm) {
               loginForm.addEventListener('submit', (e) => handleAuth(e, 'login'));
           }
       });
       ```

**2. Frontend Dashboard (`index.html` and `js/app.js`):**
   *   **Action:** Modify `index.html` to check for user authentication on load.
   *   **Guidance:**
     *   At the very beginning of `js/app.js` (or in a new `init.js` loaded first), make a `fetch` call to `GET /auth/me`.
     *   If `GET /auth/me` fails (e.g., 401 Unauthorized), redirect the user to `login.html`.
     *   If successful, proceed with loading the dashboard.
     *   **Example `js/app.js` (initial check):**
       ```javascript
       document.addEventListener('DOMContentLoaded', async () => {
           // --- Authentication Check ---
           try {
               const response = await fetch('http://localhost:8000/auth/me', { credentials: 'include' }); // Adjust backend URL
               if (!response.ok) {
                   throw new Error('Not authenticated');
               }
               const user = await response.json();
               console.log('User authenticated:', user.email);
               // You can update a welcome message on the dashboard here if needed
           } catch (error) {
               console.warn('User not authenticated, redirecting to login:', error);
               window.location.href = 'login.html'; // Redirect to login page
               return; // Stop further dashboard loading
           }

           // --- Rest of your existing DOMContentLoaded logic goes here ---
           // ... (e.g., accountsData, updateDashboardBalances, modal logic, etc.)
       });
       ```
   *   **Action:** Add a "Logout" button/link to your `index.html` header.
   *   **Guidance:**
     *   Attach an event listener in `js/app.js` to this button.
     *   On click, make a `POST` request to `/auth/logout` (with `credentials: 'include'`) and then redirect to `login.html`.

**3. Frontend Plaid Integration (`js/app.js`):**
   *   **Action:** Replace the simulated Plaid Link token creation and `onSuccess` logic with real API calls.
   *   **Guidance:**
     *   **Remove the `setTimeout` simulation** in the `connectBankBtn` event listener.
     *   **Modify `connectBankBtn` event listener:**
       *   Make a `fetch` call to your backend's `POST /plaid/create_link_token` endpoint (with `credentials: 'include'`).
       *   Use the `link_token` received from the backend to initialize `Plaid.create`.
     *   **Modify `plaidHandler.onSuccess` callback:**
       *   Make a `fetch` call to your backend's `POST /plaid/exchange_public_token` endpoint (with `credentials: 'include'`) to send the `public_token`.
       *   After successful exchange, make another `fetch` call to your backend's `GET /balances` endpoint (with `credentials: 'include'`) to retrieve and display real account data.
     *   **Example `js/app.js` (Plaid section, conceptual):**
       ```javascript
       // ... (existing Plaid.create setup)

       connectBankBtn.addEventListener('click', async () => {
           connectBankBtn.disabled = true;
           connectBankBtn.textContent = 'Connecting...';

           try {
               // 1. Get Link Token from your backend
               const linkTokenResponse = await fetch('http://localhost:8000/plaid/create_link_token', { // Adjust backend URL
                   method: 'POST',
                   headers: { 'Content-Type': 'application/json' },
                   credentials: 'include'
               });

               if (!linkTokenResponse.ok) {
                   throw new Error('Failed to get Plaid Link Token from backend.');
               }
               const { link_token } = await linkTokenResponse.json();

               // Update the Plaid handler with the real token
               plaidHandler.update({ token: link_token });
               plaidHandler.open(); // Open Plaid Link

           } catch (error) {
               console.error('Error initiating Plaid Link:', error);
               connectBankBtn.disabled = false;
               connectBankBtn.textContent = 'Connect Bank';
               alert('Failed to connect to Plaid. Please try again.');
           }
       });

       // ... (inside plaidHandler.onSuccess)
       onSuccess: async (public_token, metadata) => {
           console.log('Plaid link success!');
           console.log('public_token:', public_token);
           console.log('metadata:', metadata);

           try {
               // 2. Exchange Public Token with your backend
               const exchangeResponse = await fetch('http://localhost:8000/plaid/exchange_public_token', { // Adjust backend URL
                   method: 'POST',
                   headers: { 'Content-Type': 'application/json' },
                   body: JSON.stringify({ public_token }),
                   credentials: 'include'
               });

               if (!exchangeResponse.ok) {
                   throw new Error('Failed to exchange public token with backend.');
               }
               await exchangeResponse.json(); // Assuming backend returns success

               // 3. Fetch real balances from your backend
               const balancesResponse = await fetch('http://localhost:8000/balances', { credentials: 'include' }); // Adjust backend URL
               if (!balancesResponse.ok) {
                   throw new Error('Failed to fetch balances from backend.');
               }
               const realBalances = await balancesResponse.json();
               console.log('Real balances:', realBalances);

               // Update your dashboard with realBalances data
               // Example: accountsData.llcBank.balance = realBalances.checking_balance;
               // updateDashboardBalances();

               // Hide the connect button, show success message and refresh button
               // ... (your existing success UI logic)

           } catch (error) {
               console.error('Error during Plaid token exchange or balance fetch:', error);
               alert('Failed to process Plaid connection. Please try again.');
               // Re-enable connect button or show error UI
           }
       },
       // ... (rest of plaidHandler)
       ```

---

### Phase 3: Verification

**1. Backend Testing:**
   *   Start your FastAPI backend (`uvicorn app.main:app --reload`).
   *   Use a tool like Postman, Insomnia, or `curl` to test each new backend endpoint:
     *   `POST /auth/signup`
     *   `POST /auth/login`
     *   `GET /auth/me` (test with and without session cookie)
     *   `POST /auth/logout`
     *   `POST /plaid/create_link_token` (test with authenticated user)
     *   `POST /plaid/exchange_public_token` (requires a `public_token` from Plaid Link)
     *   `GET /balances` (test with authenticated user and connected Plaid item)

**2. Frontend Testing:**
   *   Open `signup.html` in your browser.
   *   Create a new user. Verify redirection to `index.html`.
   *   Log out.
   *   Open `login.html`. Log in with the new user. Verify redirection to `index.html`.
   *   On `index.html`, click "Connect Bank".
   *   Go through the Plaid Link flow (use Plaid's Sandbox credentials for testing, e.g., `user_good` / `pass_good` for a successful connection).
   *   Verify that the dashboard updates with real (sandbox) account balances after a successful Plaid connection.
   *   Test the "Refresh Balances" button (if implemented to call `/balances` again).

---

### Security Reminders:

*   **Never hardcode API keys or secrets** in frontend or backend code. Always use environment variables.
*   **Validate all user input** on the backend to prevent injection attacks.
*   Ensure **password hashing** is robust (e.g., `bcrypt`).
*   Use **secure, HttpOnly cookies** for session management.
*   Be mindful of **CORS settings** to prevent unauthorized access to your API.

---
