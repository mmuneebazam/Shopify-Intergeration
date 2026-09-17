# Shopify CRM Bridge

**Odoo 17 custom module** integrating Shopify's Admin GraphQL API with Odoo Sales and CRM — built as a full business application, not just an API connector. Syncs products, customers, and orders from Shopify into Odoo, automates CRM lead creation on configurable business rules, generates professional QWeb PDF reports, and pushes local product changes back to Shopify through an outbound queue.

![Dashboard](screenshots/dashboard.png)

---

## Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Screenshots](#screenshots)
- [Running Tests](#running-tests)
- [Project Structure](#project-structure)
- [Design Decisions](#design-decisions)

---

## Features

| Area | What it does |
|---|---|
| **Connection Management** | Configure a Shopify store, test connectivity, view live connection status |
| **Product Sync** | Imports Shopify products/variants into `product.product`, matched by SKU, no duplicates on re-sync |
| **Customer Sync** | Imports Shopify customers into `res.partner`, matched by external ID then email |
| **CRM Automation** | Auto-creates a CRM opportunity when a new customer's qualifying order meets a configurable minimum amount |
| **Order Sync** | Imports full order headers + line items, linked to synced customers and products |
| **Audit Logging** | Every sync operation (success or failure) is recorded in `shopify.sync.log` |
| **Dashboard** | Live stats: instance count, connection status, last sync times, 24h success/failure counts, bulk sync actions |
| **Scheduled Sync** | Cron jobs for products/customers/orders, with an overlap lock to prevent concurrent runs |
| **Manual Sync Wizard** | On-demand sync with instance/resource selection |
| **PDF Reports** | Single Shopify Order PDF, and a date-range Sales Summary PDF (top products, status breakdown, totals) |
| **Two-Way Sync** | Odoo product price edits are queued and pushed to Shopify, with retry and loop-prevention |
| **Automated Tests** | 11 tests covering sync correctness, idempotency, CRM rules, and the overlap lock |
| **Mock Mode** | Full functionality demoable without a real Shopify store (`dummy-token`) |

## Architecture

All Shopify HTTP/GraphQL communication is isolated in `services/shopify_api.py` — handling retries, timeouts, rate-limit backoff, and mock mode. Nothing else in the module talks to Shopify directly.

`models/shopify_instance.py` orchestrates every sync operation (products, customers, orders), enforcing the overlap lock and writing to the audit log. From there:

- **Products** sync into `product.product`, tracked via `shopify.product.map`
- **Customers** sync into `res.partner`, tracked via `shopify.customer.map`, and can trigger a `crm.lead`
- **Orders** sync into `shopify.order` + `shopify.order.line`, linked to the synced customers and products
- **Outbound changes** (Odoo → Shopify) are queued in `shopify.outbound.queue` and processed separately, decoupling local saves from Shopify's response time

API calls never happen directly from UI/controller code — everything routes through the service layer, which is the only place that knows about HTTP, retries, and Shopify's GraphQL schema.

## Requirements

- Odoo 17.0 (Community)
- Python `requests` (bundled with Odoo's environment)
- `wkhtmltopdf` on PATH (for PDF reports)
- `sale_management` and `crm` apps installed

## Installation

```powershell
# 1. Copy the module
# custom_addons/shopify_crm_bridge/

# 2. Update apps list and install
python odoo-bin -c odoo.conf -d <your_db> -i shopify_crm_bridge --stop-after-init
```

Then assign yourself the **Integration Manager** group:
Settings → Users → your user → *Shopify Integration* category → check **Integration Manager**.

## Configuration

1. **Shopify Integration → Configuration → Shopify Instances → New**
2. Fill in **Shop Domain** (`your-store.myshopify.com`) and **Access Token**
3. Click **Test Connection**

### Getting a real Shopify Admin API token

1. Create a free [Shopify Partner account](https://partners.shopify.com) → create a development store
2. Dev store admin → **Settings → Apps and sales channels → Develop apps → Create an app**
3. **Configuration → Admin API integration** → enable scopes: `read_products`, `read_customers`, `read_orders`
4. Install the app → copy the token from **API credentials**

### Mock mode (no store required)

Set **Access Token** to exactly `dummy-token` to run the entire module — products, customers, orders, CRM automation, reports, outbound queue — against realistic built-in sample data.

## Usage

- **Manual sync**: buttons on the instance form, or **Configuration → Manual Sync** for a combined run
- **Scheduled sync**: three cron jobs (Settings → Technical → Automation → Scheduled Actions)
- **Dashboard**: **Shopify Integration → Dashboard**
- **Reports**: Print button on any Shopify Order, or **Sales → Sales Summary Report**
- **Outbound sync**: edit a synced product's Sales Price → check **Logs → Outbound Queue**

## Screenshots

| | |
|---|---|
| ![Dashboard](screenshots/dashboard.png) | ![Instance Config](screenshots/instance-config.png) |
| Dashboard overview | Instance configuration & Test Connection |
| ![Orders](screenshots/orders-list.png) | ![Sales Summary PDF](screenshots/sales-summary-pdf.png) |
| Synced Shopify orders | Sales Summary PDF report |
| ![Sync Logs](screenshots/sync-logs.png) | ![CRM Lead](screenshots/crm-lead.png) |
| Synchronization audit log | Auto-created CRM opportunity |

## Running Tests

```powershell
python odoo-bin -c odoo.conf -d <your_db> -u shopify_crm_bridge --test-enable --stop-after-init --log-level=test
```

11/11 tests passing, covering: connection handling, product/customer/order sync + idempotency, CRM lead business rule (with resync duplicate-prevention), order sync's dependency on prior customer sync, and the overlap lock.

## Project Structure

shopify_crm_bridge/
├── models/ # Core business logic: instance, mappings, order, dashboard, outbound queue
├── services/ # Shopify API client (retries, mock mode)
├── wizard/ # Manual sync + sales summary report wizards
├── views/ # Form/tree views, menus
├── report/ # QWeb PDF templates
├── security/ # Access groups and rules
├── data/ # Sequences, cron jobs
└── tests/ # Automated test suite


## Design Decisions

- **Service layer isolation**: all HTTP/GraphQL logic lives in `shopify_api.py` so models never construct raw requests — this is what makes mock mode possible without touching sync logic.
- **External ID mapping models** (not fields on `product.product`/`res.partner` directly): keeps Shopify's IDs decoupled from Odoo's own ID space, and supports multiple Shopify instances mapping to the same Odoo record set.
- **Overlap lock over database-level locking**: a simple boolean flag on the instance record keeps cron/manual/UI sync paths consistent without introducing external job-queue infrastructure.
- **CRM rule scoped to new customers only**: prevents a lead being (re-)created every time an existing customer's data is re-synced.
- **Outbound queue instead of direct write-triggered API calls**: decouples the product save transaction from Shopify's response time/availability, and gives a retry/audit surface (`shopify.outbound.queue`) instead of failing silently or blocking the UI.

## License

LGPL-3
