# Recordkeeping Technology Project

## Project Overview
This project is a web application designed for personal finance recordkeeping. It comprises two main components:
- A Python backend API (`llc-finance-api`) built with FastAPI, which integrates with Plaid for connecting to financial institutions.
- A frontend application (`llc-finance-app`) developed using HTML, CSS, and JavaScript, providing a user interface for interaction.

The project is currently in a state ready for Plaid review, with the frontend featuring a simulated Plaid Link flow for screenshot purposes.

## Building and Running

### Backend (`llc-finance-api`)
1.  Navigate to the `llc-finance-api` directory.
2.  Install Python dependencies: `pip install -r requirements.txt`
3.  Set up environment variables: Copy `.env.example` to `.env` and populate with necessary values.
4.  Run the application: `uvicorn app.main:app --reload`

### Frontend (`llc-finance-app`)
1.  Open the `index.html` file in your web browser.

## Development Conventions
(TODO: Add details on coding styles, testing practices, and contribution guidelines after further analysis of the codebase.)

## Additional Context
- The `docs/` directory contains detailed information regarding the project's architecture and initial technical plans.
- The `HANDOFF.md` files (at the project root, `llc-finance-api/`, and `llc-finance-app/`) provide a detailed summary of the project's current state, accomplishments, and immediate next steps.