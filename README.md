<h1 align="center">Glamira Data Engineering Pipeline</h1>
<p align="center">
  MongoDB event data to IP geolocation and product datasets.
</p>
<p align="center">
  <img src="https://img.shields.io/badge/Language-Python-3776AB?style=flat-square" alt="Language: Python">
  <img src="https://img.shields.io/badge/Database-MongoDB-47A248?style=flat-square" alt="Database: MongoDB">
  <img src="https://img.shields.io/badge/Enrichment-IP2Location-0078D4?style=flat-square" alt="Enrichment: IP2Location">
  <img src="https://img.shields.io/badge/Outputs-CSV-6F42C1?style=flat-square" alt="Outputs: CSV">
  <img src="https://img.shields.io/badge/Status-Prototype-F2C94C?style=flat-square" alt="Status: Prototype">
</p>
<p align="center">
  <a href="#overview">Overview</a> ·
  <a href="#features">Features</a> ·
  <a href="#quick-start">Quick Start</a> ·
  <a href="#pipeline">Pipeline</a> ·
  <a href="#results">Results</a> ·
  <a href="#known-issues">Known Issues</a>
</p>

---

# Overview
This project processes event documents in **MongoDB `countly.summary`** with two independent Python scripts. One deduplicates IP addresses and looks up their country, region, and city. The other extracts unique product IDs and URLs from selected events, requests product pages, and saves titles from their HTML.
> **Scope:** The repository contains the scripts and a 365-row product result. The raw BSON dump and IP2Location BIN file are external inputs and are not included in the ZIP.

---

# Features
| | Feature | Implementation |
| :---: | --- | --- |
| 🌐 | IP deduplication | MongoDB aggregation groups documents by `ip` with `allowDiskUse=True` |
| 📍 | Geolocation | IP2Location lookup of country, region, and city |
| 💾 | Batch insert | Geolocation documents inserted into `countly.ip_location` in groups of 1,000 |
| 🛍️ | Product extraction | Filters seven event types and keeps the first URL per unique product ID |
| 🕷️ | Page parsing | HTTP request followed by first nonempty `<h1>` or `<title>` text |
| 📄 | CSV export | `ip_locations.csv` and `products.csv` written in the current working directory |

The scripts connect to `mongodb://localhost:27017/`. Run them on the machine where MongoDB is reachable at that address, or adjust the connection string in both files.

---

# Quick Start
## 1. Prepare your environment
- Install **Python 3**, MongoDB, and the **MongoDB Database Tools** (`mongorestore`).
- Obtain the event dump `summary.bson` and a licensed `IP-COUNTRY-REGION-CITY.BIN` file separately.
- Put the BIN file in the repository root. Both scripts resolve file paths relative to the terminal's current working directory.
- Make sure MongoDB is available on `localhost:27017` and the machine has adequate free disk space for the dump, aggregation, and output.

The ZIP includes neither a dependency manifest nor the full raw dataset. If you cloned the Git repository and its LFS object is available, `git lfs pull` can retrieve the actual `result/ip_locations.csv`; a ZIP containing only its pointer cannot reconstruct that file.

## 2. Install Python dependencies
From the repository root:
```sh
python -m venv .venv
```

Activate the environment:

| Platform | Command |
| --- | --- |
| Windows PowerShell | `.\.venv\Scripts\Activate.ps1` |
| macOS / Linux | `source .venv/bin/activate` |

```sh
python -m pip install pymongo IP2Location pandas tqdm requests beautifulsoup4
```

## 3. Restore the event collection
With the MongoDB server running, restore your standalone BSON file:
```sh
mongorestore --db=countly --collection=summary /path/to/summary.bson
```

Replace `/path/to/summary.bson` with its real path. For a Windows terminal, use the corresponding local file path. If the collection already holds the intended data, skip the restore; avoid accidentally importing the same dump twice.

Verify the input in `mongosh`:
```javascript
use countly
db.summary.countDocuments({})
db.summary.findOne({}, {ip: 1, product_id: 1, current_url: 1, collection: 1})
```

## 4. Run the enrichment scripts
Run from the **repository root** so `IP-COUNTRY-REGION-CITY.BIN` and the generated CSV paths resolve correctly:
```sh
python src/process_ip.py
python src/product_pipeline.py
```

These scripts run separately; you may run just the one you need. `process_ip.py` **drops the existing `countly.ip_location` collection at startup**, so check that you are pointing at the intended MongoDB instance before running it again.

<details>
<summary><strong>Troubleshooting</strong></summary>

| Symptom | Check |
| --- | --- |
| `FileNotFoundError` for the BIN file | Put `IP-COUNTRY-REGION-CITY.BIN` in the repository root and run from there. |
| MongoDB connection failure | Start MongoDB or edit the URI in both scripts for your server. |
| Zero IP or product records | Confirm the dump was restored to `countly.summary` and inspect its field names. |
| Most product titles are missing | The target sites may reject or change responses; the script currently has no retries or failed-ID output. |
| `result/ip_locations.csv` appears to contain three text lines | This ZIP carries a Git LFS pointer, not the CSV contents. Retrieve the LFS object or regenerate the file. |
| Output seems absent from `result/` | Scripts write CSV files to the terminal's **current directory**, not automatically into `result/`. |

**Validation:** The README was checked against the supplied source and CSV snapshot. End-to-end execution was not verified because the raw BSON, BIN file, and local MongoDB instance were not supplied.

</details>

---

# Pipeline
| Stage | Input → output | Actual behavior |
| --- | --- | --- |
| **1 · Restore** | `summary.bson` → `countly.summary` | Manual MongoDB import using Database Tools |
| **2 · Enrich IPs** | Unique `summary.ip` → `countly.ip_location` + `ip_locations.csv` | Aggregation, IP2Location lookup, and batched inserts |
| **3 · Extract products** | Selected events → product ID / URL pairs | First qualifying URL stored per ID in a Python dictionary |
| **4 · Crawl titles** | Product URLs → `products.csv` | Sequential requests, parse `<h1>` then `<title>`; failed pages omitted |

<details>
<summary><strong>Implementation details</strong></summary>

- `process_ip.py` matches non-null `ip` values and groups them in MongoDB. Each successfully resolved IP is inserted into `ip_location` in batches of 1,000. After processing, `pandas` reads the **entire collection into memory** to write the CSV.
- `product_pipeline.py` filters the `collection` field to seven named product events. It uses `product_id`, falling back to `viewing_product_id`, and `current_url`, falling back to `referrer_url`.
- For each unique product ID, the first qualifying URL is used. Requests have a five-second timeout and a fixed User-Agent. The code checks HTTP 200 and parses the first usable `<h1>` or `<title>`.
- Product failures increment a counter but are not written to the CSV or a retry file. The loop sleeps **0.05 seconds** between requests; it does not have a concurrency or proxy pool.

</details>

---

# Results
| Artifact | What is verifiable from this ZIP |
| --- | --- |
| `result/products.csv` | **365 rows**, **365 unique `product_id` values**, no empty `product_name` or `url` fields |
| `result/ip_locations.csv` | Git LFS pointer only; its text declares an object size of **143,067,899 bytes**, but the actual CSV rows are not present |

The original README reports **284,021** enriched IPs and fewer than two missing region/city values. Those figures cannot be recalculated from this ZIP because the LFS object is absent. Likewise, this version's product snapshot is **365**, so a later result of roughly 19,000 products should not be attributed to this archived code or CSV without the corresponding run artifacts.

<details>
<summary><strong>Output schemas</strong></summary>

| Output | Columns |
| --- | --- |
| `ip_locations.csv` | `ip`, `country`, `region`, `city` |
| `products.csv` | `product_id`, `product_name`, `url` |

You can check the produced counts in `mongosh` with `db.ip_location.countDocuments({})`, or count unique product IDs from the generated CSV. Raw result counts depend on the restored dump and which external product pages respond.

</details>

---

# Architecture
| Path | Responsibility |
| --- | --- |
| `src/process_ip.py` | IP aggregation, location lookup, MongoDB inserts, CSV export |
| `src/product_pipeline.py` | Event filtering, product deduplication, HTTP extraction, CSV export |
| `result/products.csv` | Included snapshot of successful product crawls |
| `result/ip_locations.csv` | LFS pointer in this ZIP |
| `README.md` | Project documentation |

The repository contains **no** GCP deployment configuration or automation script. The original README describes a GCS and VM workflow, but those environment steps are outside the checked-in code.

---

# Known Issues
**Prototype behavior:** dataset restoration is manual and product crawling is best effort. Both scripts assume local MongoDB and fixed file paths.

<details>
<summary><strong>View source-review findings</strong></summary>

| Area | Finding |
| --- | --- |
| Reruns | IP processing drops `countly.ip_location` before recomputing it. |
| Memory | IP CSV export loads all enriched documents into a pandas DataFrame; product extraction keeps all unique IDs in a dictionary. |
| Crawling | No retries, proxy rotation, persistent checkpoint, or failed-ID list; request exceptions are suppressed. |
| Data completeness | A successful HTTP page may contain a generic `<title>` rather than a reliable product name. |
| Raw data | Neither `summary.bson` nor the BIN database is bundled. |
| LFS | The IP CSV in the archive is a pointer, not directly usable data. |
| Tests | No test suite, dependency lockfile, or measured run log is included. |

</details>

---

# Roadmap
- Externalize database URI and input/output paths into configuration.
- Add safe reruns, checkpoints, and streaming CSV export.
- Track successful and failed products separately with reasons.
- Add bounded retries, respectful request pacing, and source-specific selectors.
- Record run counts and validate output completeness.
- Document VM deployment and large-file retrieval when those steps are reproducible.

---

# Contributing
Open an issue or submit a focused pull request. Include the input schema, reproduction steps, sanitized output, and how the change was checked. Avoid publishing raw IP addresses or licensed geolocation files unintentionally.

# License
No `LICENSE` file is included in the supplied archive. Maintainers should document the repository's terms and the permitted use of the underlying data and IP2Location database.

---

<p align="center"><a href="#glamira-data-engineering-pipeline">Back to top ↑</a></p>
