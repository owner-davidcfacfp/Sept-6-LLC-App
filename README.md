# Recordkeeping Technology Project

This project is a web application for personal finance recordkeeping. It consists of a Python backend API that uses Plaid to connect to financial institutions and a simple HTML, CSS, and JavaScript frontend.

## Status

The project is currently in a state where it is ready for Plaid review. The frontend has a simulated Plaid Link flow for screenshot purposes. For a detailed summary of the project's current state, what has been accomplished, and the immediate next steps, please see the [HANDOFF.md](HANDOFF.md) file.

## Getting Started

To get the project running, you will need to have Python and Node.js installed.

**Backend (`llc-finance-api`):**

1.  Navigate to the `llc-finance-api` directory.
2.  Install the Python dependencies: `pip install -r requirements.txt`
3.  Set up your environment variables by copying `.env.example` to `.env` and filling in the required values.
4.  Run the application: `uvicorn app.main:app --reload`

**Frontend (`llc-finance-app`):**

1.  Open the `index.html` file in your web browser.

## Project Structure

This project is organized into two main components:

-   `llc-finance-api/`: The backend API (Python/FastAPI).
-   `llc-finance-app/`: The frontend application (HTML/CSS/JS).

For more detailed information about the project's architecture and the initial technical plans, please see the documents in the `docs/` directory.
