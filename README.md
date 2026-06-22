# 🚗 Automotive Competitor Price Tracking System

A production-grade automated data pipeline that collects daily vehicle price data from major automotive brands in Turkey and stores it in a centralized dashboard database for competitive intelligence.

> **Status:** Live in production — running daily at Skoda Yuce Auto

---

## 💡 The Problem I Solved

The existing system relied on **RPA (Robotic Process Automation)** to scrape competitor pricing from HTML pages. Every time a brand updated their website layout, the RPA scripts broke — requiring manual intervention and rewriting.

**Root cause:** RPA reads the visual HTML structure. It's fragile by design.

**My approach:** Instead of scraping HTML, I inspected each brand's website using browser DevTools (Network tab) to identify the **internal API endpoints** their pages call to load pricing data. By calling those endpoints directly, I retrieve clean, structured JSON — bypassing the HTML layer entirely.

This makes the system **layout-independent**: as long as the underlying API contract doesn't change, the pipeline runs without maintenance.

---

## 🏗️ Architecture
```

┌─────────────────────────────────────────────────┐

│                   app.py (Orchestrator)          │

│  - Discovers all brand scripts under /brands/   │

│  - Runs them sequentially (or filtered)         │

│  - Cleans up JSON_Files after completion        │

└────────────────────┬────────────────────────────┘

│ runs each

┌──────────▼──────────┐

│  /brands/           │

│  scrape_audi.py     │

│  scrape_ford.py     │   Per-brand scraper

│  scrape_toyota.py   │   - Hits brand API endpoint

│  scrape_dacia.py    │   - Saves raw JSON to /JSON_Files/

│  scrape_renault.py  │   - Calls its loader script

│  scrape_hyundai.py  │

│  scrape_volkswagen  │

└──────────┬──────────┘

│ passes JSON path to

┌──────────▼──────────┐

│  /loaders/          │

│  audi_json_to_db    │   Per-brand loader

│  ford_json_to_db    │   - Reads JSON

│  toyota_json_to_db  │   - Normalizes & cleans data

│  ...                │   - Deduplicates (deletes today's

└──────────┬──────────┘     records before re-inserting)

│ writes to

┌──────────▼──────────┐

│   MS SQL Server     │

│                     │

│  Audi_auto_data     │  ← Brand-specific tables

│  Ford_auto_data     │

│  Toyota_auto_data   │

│  ...                │

│                     │

│  dashboard_auto_data│  ← Unified dashboard table

│  (all brands)       │     (Marka, Fiyat, car_detail, tarih)

└─────────────────────┘
```

---

## 🔑 Key Technical Decisions

### Why API-based instead of HTML scraping?

| Approach | Fragility | Maintenance | Data Quality |
|---|---|---|---|
| RPA / HTML scraping | Breaks on any layout change | High — requires rewrite per change | Inconsistent, needs heavy parsing |
| **API-based (this system)** | Stable as long as API contract holds | Low — self-healing | Clean JSON, structured from source |

### Per-brand API strategies

Each brand exposes data differently — finding the right endpoint and understanding its structure was the core engineering challenge:

- **Ford** — REST JSON API, separate endpoints for Binek / FordStore categories, merged into single payload before DB write
- **Toyota** — XML feed converted to JSON via `xmltodict`, with complex validation to filter out non-vehicle entries (OTV muafiyet rows, campaign-only records)
- **Renault / Dacia** — Shared WordPress REST API (`wp-json`), different query params per brand, multi-year price grouping
- **Volkswagen** — Versioned JSON file endpoint, nested XML-like structure within JSON, requires recursive year extraction
- **Hyundai** — Single endpoint with nested `productList → yearDetailList → priceDetailList` structure

### Idempotent daily runs

Every loader checks whether today's data already exists before inserting. If it does, it deletes and re-inserts. This means the pipeline can be safely re-run without producing duplicate records.

### Dual-table write

Each record is written to two places simultaneously:
- A **brand-specific table** (`Audi_auto_data`, `Ford_auto_data`, etc.) with full detail columns
- A **unified dashboard table** (`dashboard_auto_data`) with a normalized schema across all brands, used for cross-brand comparison reporting

---

## 📁 Project Structure

/

├── app.py                  # Orchestrator — runs all brand scripts

├── app_sayfa.py            # Alternative entry point (page-based)

├── config.py               # DB connection string (not committed)

├── brands/                 # Per-brand scraper scripts

│   ├── scrape_audi_price_list_json.py

│   ├── scrape_ford_price_list_json.py

│   ├── scrape_toyota_price_list_json.py

│   ├── scrape_dacia_price_list_json.py

│   ├── scrape_renault_price_list_json.py

│   ├── scrape_hyundai_price_list_json.py

│   └── scrape_volkswagen_price_list_json.py

├── loaders/                # Per-brand DB loader scripts

│   ├── audi_json_to_db.py

│   ├── ford_json_to_db.py

│   ├── toyota_json_to_db.py

│   ├── dacia_json_to_db.py

│   ├── renault_json_to_db.py

│   ├── hyundai_json_to_db.py

│   └── volkswagen_json_to_db.py

├── JSON_Files/             # Temporary JSON storage (auto-cleaned after run)

└── docs/                   # Architecture diagrams, screenshots
---

## ⚙️ How to Run

**Run all brands:**
```bash
python app.py
```

**Run a single brand:**
```bash
python app.py --only ford
```

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Language | Python 3 |
| HTTP / Scraping | `scrapling` (stealthy headers, anti-bot bypass) |
| XML parsing | `xmltodict` (Toyota XML feed) |
| Database | MS SQL Server via `pyodbc` |
| Orchestration | `subprocess` + `argparse` |
| Data format | JSON (all brands normalized before DB write) |

---

## 📊 Data Output Schema (Dashboard Table)

| Column | Description |
|---|---|
| `Marka` | Brand name (Audi, Ford, Toyota...) |
| `Fiyat` | List price (INT, TL) |
| `Kampanyali_Fiyat` | Campaign price if available (INT or NULL) |
| `car_detail` | Full model descriptor (model + specs + year) |
| `tarih` | Date of data collection (DD.MM.YYYY) |

---

## 🔒 What's Not Included

- `config.py` — contains internal DB connection string (excluded from repo)
- Raw JSON files — auto-deleted after each run
- Internal dashboard UI and reporting layer (separate system)

---



