What would I do differently?

Version 1

Introduce database migration tooling (e.g., Alembic) instead of relying on ad‑hoc SQL statements.

Suggested task
Add Alembic migrations for user and Plaid tables

Start task
Add CSRF protection because sessions are stored in cookies.

Suggested task
Implement CSRF protection for auth endpoints

Start task
Rate-limit authentication routes to reduce brute‑force risks.

Suggested task
Rate-limit authentication endpoints





Version 2
Potential improvements

The existing backend still allows “*” origins while using cookies; tightening CORS to explicit URLs would prevent unintended cross-origin access

Suggested task
Restrict CORS configuration

Start task
Cookie-based authentication should include CSRF protection to stop cross-site forgery attacks.

Suggested task
Add CSRF protection

Start task
Database tables are created inline; using a migration tool like Alembic would give versioned, reversible schema changes

Suggested task
Adopt database migrations

