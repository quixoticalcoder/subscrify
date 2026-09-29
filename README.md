# subscrify

**A subscription operations workspace with a customer storefront, quotation-to-invoice workflows, and billing reports.**

subscrify brings product configuration, recurring plans, customer records, subscriptions, invoices, and payments into one React application backed by a Flask API. Staff manage the commercial workflow through an admin workspace; customers browse the catalog, check out, and review their orders and invoices through a separate portal.

This repository is a functional application prototype. Recurring billing dates are calculated in application code, but invoice generation and renewal are triggered through API actions: there is no background billing scheduler or automatic recurring charge service. The [implementation limits](#implementation-limits) describe the remaining work before a production deployment.

## Watch demo video

https://youtu.be/LB2FhDEiW6E?si=xKNWb2cSmSOAacQi

## Contents

- [Capabilities](#capabilities)
- [Architecture](#architecture)
- [Local setup](#local-setup)
- [Configuration](#configuration)
- [Using the application](#using-the-application)
- [API map](#api-map)
- [Repository layout](#repository-layout)
- [Development and validation](#development-and-validation)
- [Implementation limits](#implementation-limits)
- [Deployment notes](#deployment-notes)

## Capabilities

| Area | Implemented behavior |
| --- | --- |
| Account access | Login with email or login ID, bcrypt password hashing, JWT sessions, first-user admin registration, staff invitations, and email OTP password reset. |
| Catalog | Products and services, images, attribute values, price adjustments for variants, taxes, and product recurring-price records. |
| Subscription configuration | Recurring plans, quotation templates, discounts, and payment terms. |
| Staff operations | Create subscription drafts, send quotations, confirm orders, generate invoices, renew, close, and create upsell orders. |
| Billing | Invoice line items, percentage or fixed taxes, discount percentages, outstanding balances, manual payment records, and Razorpay checkout integration. |
| Customer portal | Public catalog and product pages, persistent cart, checkout, order history, invoice views, profile editing, renewal, and closure actions. |
| Reporting | Dashboard totals, subscription and revenue reports, overdue invoices, payment reports, and CSV/Excel exports for supported reports. |
| Interface | Light/dark themes, guided tours, staff navigation shortcuts, and printable receipt/report views. |
| Optional AI assistant | A browser-based Groq chat assistant that collects application records through existing APIs and includes them in its prompt context. |

The three account roles are `admin`, `internal`, and `portal`. They shape navigation and many API permissions, but authorization is incomplete on some endpoints; the role system must not be treated as a complete tenant-isolation boundary.

## Architecture

```mermaid
flowchart LR
    Browser[React application] -->|Axios / JWT| API[Flask REST API]
    API --> ORM[SQLAlchemy models]
    ORM --> DB[(PostgreSQL)]
    API --> Email[Brevo email]
    API --> Images[Cloudinary uploads]
    API --> Payments[Razorpay orders]
    Browser -->|Optional chat and record context| AI[Groq API]
```

| Layer | Technologies |
| --- | --- |
| Web client | React 19, React Router 7, Redux Toolkit, Vite 7 |
| Styling and charts | Tailwind CSS 4, Lucide, GSAP, Recharts |
| API | Flask, Flask-CORS, Flask-JWT-Extended |
| Persistence | Flask-SQLAlchemy, PostgreSQL via psycopg2 |
| Authentication | bcrypt and JWT access tokens |
| Exports | pandas and openpyxl |
| External services | Brevo, Cloudinary, Razorpay, and optional Groq |

A user can have a linked contact record. Contacts own subscriptions and invoices. Subscription lines reference catalog products; invoice generation copies those lines into invoice lines. Invoice totals are computed from line values and referenced tax records, and completed payment records reduce the amount due.

## Local setup

### Requirements

- Git, Python 3.10 or newer, and PostgreSQL.
- Node.js 20.19+ or 22.12+ with npm for Vite 7.
- A local PostgreSQL role that can connect to the database you create.
- Service credentials only for the integrations you intend to exercise.

The Python dependency file is currently unpinned. Use a dedicated virtual environment; compatibility depends on the versions installed.

### 1. Clone the repository

```bash
git clone https://github.com/quixoticalcoder/subscrify.git
cd subscrify
```

### 2. Configure and start the API

```bash
cd server
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp env.example .env
createdb subscrify
```

On Windows, activate the virtual environment with `venv\Scripts\activate`. If your PostgreSQL installation requires an explicit user or host, pass those options to `createdb`.

Edit `server/.env`: set `DATABASE_URL` to the credentials for your database, and replace both secret-key placeholders. Then run:

```bash
flask --app run:app init-db
python run.py
```

The API listens on `http://localhost:5000`. `init-db` creates missing tables; it does not migrate existing schemas. The development entry point also calls `create_all()` and enables Flask debug mode.

Verify database connectivity:

```bash
curl http://localhost:5000/api/health
```

A successful response has `status: "healthy"` and `database: "ok"`. A database connectivity failure returns HTTP 503.

### 3. Start the frontend

Open a second terminal from the repository root:

```bash
cd frontend
npm ci
cp .env.example .env
npm run dev
```

Open the URL printed by Vite, normally `http://localhost:5173`. The client defaults to the API at `http://localhost:5000`.

### 4. Create your first account

Open `/signup` on a fresh database. The **first registered user becomes an administrator**; later public signups create portal accounts. Register the intended administrator before exposing a new installation to other users.

Login IDs must contain 6–12 characters. Passwords must be longer than eight characters and contain uppercase, lowercase, and a special character. Registration attempts a welcome email through Brevo; delivery requires valid service configuration.

### Optional demo dataset

> **Destructive operation:** `server/seed.py` calls `db.drop_all()` before rebuilding the schema. It erases the database selected by `DATABASE_URL`. Use it only against a disposable demo database.

After deliberately configuring a disposable database, run from `server/` with the virtual environment active:

```bash
python seed.py
```

The seed creates sample users, products, plans, contacts, subscriptions, invoices, and payments. Its generated history uses fixed dates, so dashboard results can change as the current date moves beyond the demo period.

| Role | Email | Demo password |
| --- | --- | --- |
| Admin | `admin@demo.com` | `Admin@123!` |
| Internal | `john@demo.com` | `Internal@123!` |
| Portal | `aarav.sharma0@customer.com` | `Portal@123!` |

These are public sample credentials for the disposable dataset.

## Configuration

Backend values live in `server/.env`, copied from [`server/env.example`](server/env.example). The `run.py` entry point loads this file before constructing the app.

| Variable | Purpose |
| --- | --- |
| `DATABASE_URL` | SQLAlchemy connection URL, normally `postgresql://USER:PASSWORD@localhost:5432/subscrify`. |
| `SECRET_KEY` | Flask application secret. |
| `JWT_SECRET_KEY` | Key used to sign access tokens. |
| `JWT_ACCESS_TOKEN_EXPIRES` | Access-token lifetime in seconds; default `3600`. |
| `BREVO_API_KEY` | Transactional email API credential. |
| `BREVO_SENDER_EMAIL` | Sender address configured with your email provider. |
| `BREVO_SENDER_NAME` | Email display name; `subscrify`. |
| `CLOUDINARY_CLOUD_NAME` | Cloudinary account cloud name. |
| `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET` | Server-side upload credentials. |
| `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET` | Credentials for online payment order creation and signature verification. |
| `FRONTEND_URL` | Frontend origin used for links in emails; default `http://localhost:5173`. This does not configure CORS. |

Frontend configuration is documented in [`frontend/.env.example`](frontend/.env.example):

```dotenv
VITE_BASE_URL=http://localhost:5000
```

Use the server origin **without `/api`** because endpoint paths already include that prefix. Vite embeds these values at build time; restart development or rebuild after changing them. Never put server secrets in a `VITE_` variable.

The chatbot asks for a Groq API key in its settings and stores it in browser local storage. It currently requests `llama-3.1-8b-instant`. Its context comes from application API responses rather than an embedding index or vector database. When used, that context is sent directly from the browser to Groq.

## Using the application

### Staff workflow

1. Sign in and open `/admin/dashboard`.
2. Configure taxes, attributes, products, recurring plans, and customer contacts.
3. Create a subscription with customer, plan, and order lines.
4. Send a quotation if needed, then confirm the subscription.
5. Generate an invoice. This creates a draft invoice, activates a confirmed subscription, and advances its next invoice date where a plan applies.
6. Confirm the invoice and review its payments and outstanding balance.
7. Use reports to inspect subscriptions, revenue, overdue balances, and payments.

The modeled subscription states include `draft`, `quotation`, `quotation_sent`, `confirmed`, `active`, and `closed`. Individual actions enforce their own transitions; renewal creates a new subscription record. Invoice states include `draft`, `confirmed`, `paid`, and `cancelled`.

### Customer workflow

Browse `/portal`, open a product, and add it to the cart. Sign in before checkout and complete the required address details. Checkout creates a confirmed subscription and invoice. Online checkout opens Razorpay; the cash option leaves an invoice to be settled later rather than recording a completed cash payment automatically.

Customers can review orders and invoices under `/portal/orders` and update their details under `/portal/profile`.

### Browser state

Tokens, user data, cart contents, theme, tour completion, and the optional Groq key use `subscrify_` local-storage keys. The naming cleanup starts a fresh browser session and cart for installations that used the previous keys. Existing database names and Cloudinary assets are not migrated automatically: keep an explicit `DATABASE_URL` for an existing database.

## API map

All groups below are relative to `/api`. Protected requests use `Authorization: Bearer <token>`.

| Group | Representative routes |
| --- | --- |
| Health | `GET /health` |
| Authentication | `POST /auth/register`, `/auth/login`, `/auth/forgot-password`, `/auth/verify-otp`, `/auth/reset-password`; `GET /auth/me` |
| Staff and customers | `/users`, `/contacts` |
| Catalog | `/products`, `/attributes`, `/taxes`; public `GET /products/public` |
| Configuration | `/recurring-plans`, `/quotation-templates`, `/discounts`, `/payment-terms` |
| Subscriptions | `/subscriptions`; `POST /subscriptions/:id/confirm`, `/create-invoice`, `/renew`, `/close`, `/upsell` |
| Invoices | `/invoices`; `POST /invoices/:id/confirm`, `/send`, `/cancel` |
| Payments | `/payments`; `POST /payments/razorpay/create-order`, `/payments/razorpay/verify` |
| Reports | `GET /reports/dashboard`, `/subscriptions`, `/revenue`, `/overdue-invoices`, `/payments` |
| Customer portal | `/portal/products`, `/portal/recurring-plans`, `/portal/orders`, `/portal/checkout`, `/portal/profile` |
| Uploads | `/upload` blueprint for Cloudinary image uploads |

Subscription, overdue-invoice, and payment reports accept `?export=csv` or `?export=excel`. This table is an orientation guide, not a complete permission contract. Consult [`server/app/routes`](server/app/routes) for methods, payloads, and current checks, and [`frontend/src/configs/api.js`](frontend/src/configs/api.js) for client wrappers.

## Repository layout

```text
subscrify/
├── frontend/
│   ├── src/
│   │   ├── components/common/   # Shared UI, tour, and Groq assistant
│   │   ├── configs/api.js       # Axios client and endpoint wrappers
│   │   ├── layout/              # Admin and portal shells
│   │   ├── pages/               # Auth, admin, and portal screens
│   │   ├── store/               # Authentication, cart, and theme state
│   │   └── App.jsx              # Routes and client guards
│   ├── public/                 # Static assets
│   └── .env.example
├── server/
│   ├── app/
│   │   ├── __init__.py          # Canonical Flask app factory
│   │   ├── models/__init__.py   # SQLAlchemy entities and totals
│   │   ├── routes/              # API blueprints
│   │   └── utils/               # Email, password, date, and role helpers
│   ├── env.example
│   ├── requirements.txt
│   ├── run.py                  # Environment loading, CLI, dev entry point
│   ├── seed.py                 # Destructive demo-data generator
│   └── wsgi.py                 # Alternative WSGI entry point
└── README.md
```

## Development and validation

From `frontend/`:

```bash
npm run dev       # Local development
npm run build     # Production assets in dist/
npm run preview   # Local preview of those assets
npm run lint      # ESLint checks
```

From `server/`, with its virtual environment active:

```bash
python -m compileall -q app run.py seed.py wsgi.py
flask --app run:app routes
```

The repository does not include a maintained automated test suite or migration history. The frontend build currently succeeds with a large-bundle warning. ESLint currently reports 26 errors and 12 warnings, including unused bindings and hook dependency issues; it is not a passing quality gate yet.

When changing workflows, exercise subscription creation → confirmation → invoice generation, both customer and staff access, and export results against an isolated database. Mock email and payment providers for local tests. SQLite can support a lightweight smoke check but does not establish PostgreSQL compatibility; some date assignments also differ between database drivers.

## Implementation limits

- **Authorization:** several general record-detail and payment routes require a JWT without consistently checking ownership or staff role. Public product serialization includes internal fields such as cost price. Audit server-side access checks and response fields before handling real customer data.
- **Payments:** signature checking does not provide a complete persisted order-to-invoice binding, replay protection, or independent provider-side amount verification. The general verification route accepts a client-supplied amount. Portal checkout commits the order and invoice before creating the provider order, so a provider failure can leave records behind. Razorpay order creation currently uses INR.
- **Pricing:** portal checkout uses the product sales price plus variant adjustment; configured recurring-price records are not consistently used there. Fixed discounts and validation of quantities, variants, and referenced products require further work.
- **Billing automation:** there is no included worker for scheduled invoicing, auto-close, recurring charges, or payment reconciliation. Plan fields alone do not implement those services.
- **Authentication:** reset OTPs and verification state live in process memory and use Python's general-purpose random generator. They do not survive restarts or synchronize across workers. Rate limiting and durable reset storage are not implemented.
- **Persistence:** schema creation is based on `create_all()`, with no committed migrations. Record-number generation and monetary calculations need review for concurrency and financial precision; calculations convert stored numeric values to floats.
- **Runtime:** CORS allows broad origins, the development server enables debug mode, and the chatbot key is browser-stored. Large client bundles, unpinned backend dependencies, and the current lint backlog remain maintenance work.

## Deployment notes

Build the frontend with the intended API origin, serve `frontend/dist/` through a static host, and configure an SPA fallback to `index.html` for nested routes. Run the backend from `server/` under a WSGI server, for example:

```bash
gunicorn --bind 0.0.0.0:5000 run:app
```

Initialize the database explicitly before serving requests. The alternative `wsgi.py` does not load `.env` itself, so its environment must be provided externally. Multiple workers need shared reset-token storage before the password-reset flow can work reliably across processes.

Resolve the authorization and payment gaps above, narrow CORS, configure HTTPS and service credentials, introduce migrations and backups, and test with PostgreSQL and provider test credentials before deploying for real billing.

## Contributing and license

Keep changes focused, describe the behavior they affect, and include verification steps. Use `subscrify` for the project name in UI text, package metadata, emails, and documentation.

No license file is included in this repository. Public source availability does not by itself grant an open-source license.
