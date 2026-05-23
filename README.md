# 🛒 AtliqMart Supply Chain Analytics

> **An end-to-end automated supply chain analytics pipeline** — from raw sales emails to AI-powered KPI insights — built with **n8n**, **PostgreSQL (Supabase)**, **Quadratic**, and **OpenExchangeRates**.

---

## 📖 Business Story

**AtliqMart** is a Gujarat-based organic food manufacturer operating in Ahmedabad and Vadodara (India) and New Jersey (USA). Despite strong market disruption, their supply chain lagged behind — customers were dissatisfied with order fulfillment, and the company needed to fix this before scaling.

The COO identified a classic supply chain problem: **inability to maintain optimum inventory**. The solution needed to be AI-powered, automated, and built on real-time data from both geographies.

---

## 🏗️ Technical Architecture

```
📧 Gmail (CSV Attachments)
        │
        ▼
   [n8n Workflow]
   ┌──────────────────────────────────┐
   │  Gmail Trigger                   │
   │       │                          │
   │       ├──► Extract Aggregate CSV ──► PostgreSQL (fact_aggregate)
   │       │                          │
   │       └──► Extract Order Line CSV ─► PostgreSQL (fact_order_line)
   └──────────────────────────────────┘
        │
        ▼
 PostgreSQL on Supabase (Cloud)
        │
        ▼
  [Quadratic AI Spreadsheet]
  ├── Real-time data pull from Postgres
  ├── Python + AI prompt-based analysis
  ├── OpenExchangeRates API (USD ↔ INR)
  └── Supply Chain KPI Dashboard
```

**Flow:** Sales data from India and USA is emailed as CSV attachments → n8n monitors Gmail and automatically extracts and inserts rows into PostgreSQL → Quadratic pulls from Postgres, applies exchange rate conversion, and computes KPIs using AI-assisted Python code.

---

## ⚙️ Tools & Technologies

| Layer | Tool | Purpose |
|---|---|---|
| Automation | [n8n](https://n8n.io) | Gmail trigger → CSV extraction → Postgres insert |
| Database | [Supabase (PostgreSQL)](https://supabase.com) | Cloud-hosted relational database |
| AI Spreadsheet | [Quadratic](https://www.quadratichq.com) | Prompt-based Python analysis + DB connection |
| Exchange Rates | [OpenExchangeRates](https://openexchangerates.org) | Real-time USD ↔ INR currency conversion |
| Data Source | Gmail CSV Attachments | India & USA sales data, emailed periodically |

---

## 🗂️ Database Schema

### Fact Tables (Transactions)

**`fact_order_line`** — Line-level order data
| Column | Description |
|---|---|
| `order_id` | Unique order identifier |
| `order_placement_date` | Date order was placed |
| `customer_id` | Customer reference |
| `product_id` | Product reference |
| `order_qty` | Quantity ordered |
| `agreed_delivery_date` | Committed delivery date |
| `actual_delivery_date` | Actual delivery date |
| `delivery_qty` | Quantity delivered |
| `In Full` | 1 if delivered in full, else 0 |
| `On Time` | 1 if delivered on agreed date, else 0 |
| `On Time In Full` | 1 if both On Time and In Full, else 0 |

**`fact_aggregate`** — Order-level summary
| Column | Description |
|---|---|
| `order_id` | Unique order identifier |
| `customer_id` | Customer reference |
| `order_placement_date` | Date order was placed |
| `on_time` | On-time delivery flag |
| `in_full` | In-full delivery flag |
| `otif` | On-Time In-Full flag |

### Dimension Tables (Reference Data)

| Table | Description |
|---|---|
| `dim_customers` | Customer names, cities, currency (USD/INR) |
| `dim_products` | Product names, categories, prices (INR & USD) |
| `dim_date` | Date dimension with month, year, quarter columns |
| `dim_target_orders` | Target order quantities per customer/product |
| `exchange_rate` | Daily USD→INR rates from OpenExchangeRates |

---

## 🤖 n8n Automation Workflow

The n8n workflow is triggered whenever a new email arrives in the designated Gmail account with sales data attached.

**Workflow Steps:**
1. **Gmail Trigger** — Watches for incoming emails with CSV attachments
2. **Extract from File: Aggregate** — Parses `fact_aggregate` CSV (order-level summary)
3. **Insert rows in a table** — Inserts aggregate data into PostgreSQL
4. **Extract from File (Order Line)** — Parses `fact_order_line` CSV (line-level details)
5. **Insert rows in a table1** — Inserts order line data into PostgreSQL

**Date Format Fix:** The incoming CSVs use `dd-mm-yyyy` format. n8n transformation applied:
```
toDateTime("dd-mm-yyyy").format('yyyy-mm-dd')
```

---

## 📊 Supply Chain KPIs

All KPIs are computed in Quadratic using Python and AI-assisted prompts, pulling directly from PostgreSQL.

| KPI | Definition | Audience |
|---|---|---|
| **Total Order Lines** | Count of all order line items | Operations |
| **Line Fill Rate** | Lines delivered in full / Total lines ordered | Supply Planners, Production Managers |
| **Volume Fill Rate** | Sum of qty delivered / Sum of qty ordered | Sales, Negotiation Teams |
| **Total Orders** | Count of distinct orders | Management |
| **On Time Delivery %** | Orders delivered on agreed date / Total orders | Warehouse & Distribution |
| **In Full Delivery %** | Orders delivered in full / Total orders | Supply Planners |
| **OTIF %** | Orders that are both On Time AND In Full / Total orders | Supply Chain Directors |

> **OTIF (On-Time In-Full)** is the primary reliability metric in supply chain management and is aligned with the SCOR model framework.

### KPI Definitions Explained

- **Line Fill Rate:** If 3 order lines exist and 2 are delivered fully → Line Fill Rate = 2/3 = 66.7%
- **Volume Fill Rate:** If 15 units ordered and 12 delivered → Volume Fill Rate = 12/15 = 80%
- **OTIF:** An order must satisfy *both* on-time AND in-full conditions to count as OTIF = 1

---

## 💱 Currency Conversion

Products are priced in both INR and USD. AtliqMart serves customers in India (INR) and the US (USD). The pipeline integrates **OpenExchangeRates** to apply daily exchange rates for consistent cross-geography reporting.

Exchange rates are stored in the `exchange_rate` dimension table and joined during analysis in Quadratic.

---

## 📁 Repository Structure

```
├── data/
│   ├── raw/
│   │   ├── fact_aggregate_india_2025-05-17.csv    # India aggregate (email sample)
│   │   ├── fact_aggregate_usa_2025-05-17.csv      # USA aggregate (email sample)
│   │   ├── fact_order_line_india_2025-05-17.csv   # India order lines (email sample)
│   │   └── fact_order_line_usa_2025-05-17.csv     # USA order lines (email sample)
│   └── quadratic_exports/
│       ├── SalesAnalysis_-_fact_order_line.csv    # Full order line dataset
│       ├── SalesAnalysis_-_fact_summary.csv       # Summary dataset
│       ├── SalesAnalysis_-_fact_aggregate.csv     # Aggregate dataset
│       ├── SalesAnalysis_-_KPI.csv                # Computed KPIs
│       ├── SalesAnalysis_-_Business_Questions.csv # Business Q&A outputs
│       ├── SalesAnalysis_-_dim_customers.csv      # Customer dimension
│       ├── SalesAnalysis_-_dim_products.csv       # Product dimension
│       ├── SalesAnalysis_-_dim_date.csv           # Date dimension
│       ├── SalesAnalysis_-_dim_target_orders.csv  # Target orders
│       └── SalesAnalysis_-_exchange_rate.csv      # Exchange rates
├── quadratic/
│   └── SalesAnalysis.grid                         # Quadratic spreadsheet file
├── docs/
│   └── Business_Story.docx                        # Project brief & architecture notes
├── n8n/
│   └── workflow_screenshot.png                    # n8n workflow diagram
└── README.md
```

---

## 🚀 How to Reproduce

### 1. Set Up PostgreSQL on Supabase
- Create a new Supabase project
- Create the tables: `fact_order_line`, `fact_aggregate`, `dim_customers`, `dim_products`, `dim_date`, `dim_target_orders`, `exchange_rate`
- Note your connection string:
  ```
  postgresql://postgres.<project-ref>:[PASSWORD]@aws-1-us-east-2.pooler.supabase.com:5432/postgres
  ```

### 2. Configure n8n
- Import the workflow (see `/n8n/`)
- Connect your Gmail account
- Set your PostgreSQL credentials in the Postgres nodes
- Activate the workflow — it will trigger on every new email

### 3. Set Up Quadratic
- Open [Quadratic](https://www.quadratichq.com) and create a new spreadsheet
- Connect to your PostgreSQL database using the Supabase connection string
- Import the `.grid` file from `/quadratic/SalesAnalysis.grid`
- Add your OpenExchangeRates API key for live currency conversion

### 4. Run Analysis
- Send a test email with India/USA CSV attachments to your Gmail
- Verify rows appear in PostgreSQL
- Open Quadratic — data will auto-populate; KPIs will compute via Python cells

---

## 📈 Business Questions Answered

Using Quadratic's AI-powered Python cells, the following questions were answered:

- Top 5 customers by order value
- On-time delivery performance by city
- In-full rate by product category
- OTIF trends over time
- Revenue comparison: India vs USA (normalized to USD)

---

## 🔑 Key Learnings

- **Fact tables** store transactional data; **Dimension tables** store slowly-changing reference data — a core star-schema design principle.
- **Always include a `dim_date` table** with pre-computed month, year, and quarter columns for efficient time-series analysis.
- **AI can hallucinate** — in production supply chain decisions, always validate AI outputs against source data before acting.
- The **SCOR model** (Supply Chain Operations Reference) provides the standard framework for supply chain KPIs like OTIF.

---

## 📄 License

This project is for educational and portfolio purposes.

---

*Built with n8n · Supabase PostgreSQL · Quadratic · OpenExchangeRates*
