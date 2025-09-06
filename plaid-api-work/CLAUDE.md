# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a personal finance recordkeeping application built for LLC management. The repository structure includes:
- **Backend (root level)**: Python/FastAPI service with Plaid integration for bank data
- **Frontend (`llc-finance-app/`)**: Vanilla HTML/CSS/JavaScript single-page application

The system is designed as a single-user application that persists dashboard state as JSON documents and optionally fetches real bank balances via Plaid API.

## Architecture

### Backend (root level)
- **FastAPI** application with SQLAlchemy + PostgreSQL
- **Single-user authentication** via `X-API-Key` header
- **Plaid integration** for bank account connections and balance fetching
- **Encryption** of Plaid access tokens using Fernet (cryptography library)
- **Three main data stores**:
  - `app_state`: JSON document storing dashboard state
  - `plaid_items`: Encrypted Plaid access tokens by institution
  - `plaid_balances`: Cached account balances from Plaid API

### Frontend (llc-finance-app/)
- **Vanilla JavaScript** SPA - no build process required
- **State persistence** via backend API calls
- **Plaid Link integration** for connecting bank accounts
- **Query parameter configuration**: `?apiBase=...&apiKey=...` stored in localStorage

## Development Commands

### Backend Development
```bash
# Set up environment (from repository root)
python -m venv .venv && source .venv/bin/activate  # Linux/Mac
# or: .venv\Scripts\activate                        # Windows

# Install dependencies
pip install -r requirements.txt

# Run development server
uvicorn app.main:app --reload --port 8000

# Alternative: Run with specific host/port
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

### Environment Setup
Copy `.env.example` to `.env` and configure:
- `DATABASE_URL`: PostgreSQL connection string
- `API_KEY`: Shared secret for authentication
- `FERNET_KEY`: Generate with `python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`
- `PLAID_CLIENT_ID`, `PLAID_SECRET`, `PLAID_ENV`: Optional Plaid configuration

### Frontend Development
No build process required. Simply open `llc-finance-app/index.html` in a browser.

For API integration, append query parameters:
```
index.html?apiBase=http://localhost:8000&apiKey=your-api-key
```

## API Endpoints

### Core State Management
- `GET /state` - Retrieve saved dashboard state
- `PUT /state` - Save dashboard state (JSON document up to 1MB)

### Plaid Integration
- `POST /plaid/link-token` - Generate Plaid Link token for account connection
- `POST /plaid/exchange` - Exchange public token for encrypted access token
- `POST /plaid/refresh-balances` - Fetch current balances from all connected accounts

### Health Check
- `GET /healthz` - Service health status

## Deployment

### Render Platform (Primary)
```bash
# Deploy using blueprint (from repository root)
render blueprint deploy

# Set required secrets via CLI or dashboard:
# - DATABASE_URL
# - API_KEY  
# - FERNET_KEY
# - PLAID_CLIENT_ID (optional)
# - PLAID_SECRET (optional)
```

### Local Testing
```bash
# Test API endpoints
curl -H "X-API-Key: your-key" http://localhost:8000/healthz
curl -H "X-API-Key: your-key" http://localhost:8000/state

# Test state update
curl -X PUT -H "X-API-Key: your-key" -H "Content-Type: application/json" \
  -d '{"state":{"test":"data"}}' http://localhost:8000/state
```

## Key Files

### Backend Core
- `app/main.py` - FastAPI application and route handlers
- `app/models.py` - SQLAlchemy table definitions
- `app/security.py` - Authentication and encryption utilities
- `app/plaid_client.py` - Plaid API client configuration
- `app/schemas.py` - Pydantic request/response models
- `app/db.py` - Database connection and session management

### Frontend Core
- `index.html` - Main application interface
- `js/app.js` - Application logic, API integration, Plaid Link handling
- `privacy.html` - Plaid-required privacy policy page

### Deployment
- `Dockerfile` - Container build for Render deployment
- `render.yaml` - Render platform configuration
- `requirements.txt` - Python dependencies

## Testing and Quality Assurance

This project does not currently have automated tests or linting configured. Manual testing is performed via:
- API endpoint testing using curl commands (see Local Testing section)
- Frontend testing by opening `index.html` in browsers
- Integration testing with Plaid sandbox environment

## Current Status

The project is ready for Plaid production approval. The frontend includes simulated Plaid Link flow for screenshots. All core functionality is implemented:
- Secure token storage and encryption
- Real-time balance fetching
- State persistence
- Authentication

Next steps involve transitioning from simulated to live Plaid integration once production API keys are approved.