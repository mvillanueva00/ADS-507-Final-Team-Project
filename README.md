# ADS-507 Final Team Project
## FDA Drug Shortage Analysis Pipeline

### Team Members
- Mark Villanueva
- Nancy Walker
- Sheshma Jaganathan

---

## Project Overview

This project builds a MySQL-based data pipeline that combines two FDA datasets (National Drug Code database and Drug Shortages) to enable enriched analysis. By joining these datasets, we can answer questions that aren't possible with either dataset alone, such as:
- Which manufacturers have the highest shortage risk?
- Do branded drugs have longer shortage durations than generics?
- Which package types are most vulnerable to shortages?
- How do prescription vs OTC drugs differ in shortage patterns?
- Which routes of administration (oral, IV, injection) are most affected?

**Key Achievement:** ~90% of drug shortage records successfully matched with NDC product data, enabling comprehensive enriched analysis.

---

## Repository Structure
```
ADS-507-Final-Team-Project/
├── data/                      # Local data storage (not committed to GitHub)
│   └── DATA_SOURCE.md         # Data source documentation
├── docs/                      # Documentation and diagrams
├── scripts/                   # Python automation scripts
│   ├── download_data.py       # Downloads FDA datasets
│   ├── process_data.py        # Cleans and processes data
│   ├── load_to_mysql.py       # Loads data into MySQL
│   └── dashboard.py           # Streamlit interactive dashboard
├── sql/                       # SQL scripts
│   ├── 01_create_tables.sql   # Creates database structure
│   ├── 02_transformations.sql # Joins datasets (required SQL transformation)
│   └── 03_analysis_queries.sql# Analytical queries
├── .gitignore                 # Prevents committing large files
├── requirements.txt           # Python dependencies
└── README.md                  # This file
```

---

## Prerequisites

Before running the pipeline, ensure you have:

1. **Python 3.8+** installed
   - Check: `python --version`
   - Download: https://www.python.org/downloads/
   - **IMPORTANT:** During installation, check "Add Python to PATH"

2. **MySQL Server** installed and running
   - MySQL Workbench (recommended for running SQL scripts)
   - Download: https://dev.mysql.com/downloads/
   - Remember your root password!

3. **Git** (for cloning repository)
   - Download: https://git-scm.com/downloads
   - Or simply download as ZIP from GitHub

---

## Setup Instructions

### Step 1: Download the Repository

**Option A: Download ZIP**
1. Click the green "Code" button on GitHub
2. Click "Download ZIP"
3. Extract to a location like `C:\Users\YourName\Desktop\ADS-507`

**Option B: Clone with Git**
```bash
git clone https://github.com/YOUR-USERNAME/ADS-507-Final-Team-Project.git
cd ADS-507-Final-Team-Project
```

### Step 2: Install Python Dependencies

Open Command Prompt or PowerShell and navigate to the project folder:
```bash
cd path\to\ADS-507-Final-Team-Project
```

Install required packages:
```bash
python -m pip install -r requirements.txt
```

This installs: pandas, requests, mysql-connector-python, sqlalchemy, streamlit, plotly

**Time:** 2-3 minutes

---

## Pipeline Execution (Sequential Order)

### **Phase 1: Download Raw Data**
```bash
python scripts/download_data.py
```

**What it does:**
- Downloads FDA NDC database (~119MB compressed, expands to ~500MB)
- Downloads FDA Drug Shortages dataset (~2MB)
- Saves raw JSON files to `data/` folder

**Expected output:**
- `data/drug-ndc-0001-of-0001.json` (131,118 products)
- `data/drug-shortages-0001-of-0001.json` (1,838 shortage records)

**Time:** 2-5 minutes (depending on internet speed)

**Note:** Script automatically handles correct FDA API endpoints (uses plural "shortages" not "shortage")

---

### **Phase 2: Process and Clean Data**
```bash
python scripts/process_data.py
```

**What it does:**
- Reads raw JSON files
- Normalizes nested structures
- Extracts package_ndc for joining
- **Removes duplicates** (FDA data contains duplicate records)
- Creates clean CSV tables

**Expected output:**
- `data/ndc_core.csv` (~128,805 products after deduplication)
- `data/ndc_packaging.csv` (~244,786 packaging records after deduplication)
- `data/drug_shortages_core.csv` (~1,810 shortages after deduplication)
- `data/shortage_contacts.csv` (~1,810 contact records)

**Time:** 1-2 minutes

**Technical Note:** The script removes duplicate product_ndc and package_ndc values that exist in the raw FDA data.

---

### **Phase 3: Create MySQL Database**

Open **MySQL Workbench** and connect to your local MySQL server.

Then:
1. Click **File** → **Open SQL Script**
2. Navigate to `sql/01_create_tables.sql`
3. Click **Open**
4. Click the **lightning bolt icon** ⚡ (or press Ctrl+Shift+Enter) to execute

**What it does:**
- Creates database: `fda_shortage_db`
- Creates 4 tables with proper structure and indexes:
  - `raw_ndc` (product information)
  - `raw_ndc_packaging` (packaging details)
  - `raw_drug_shortages` (shortage data)
  - `shortage_contacts` (contact information)

**Expected output:**
- Database and empty tables created
- Green checkmarks in Action Output
- Success message displayed

**Time:** < 1 minute

**Technical Notes:**
- Uses TEXT data type for fields with variable-length content (brand_name, labeler_name, therapeutic_category, dosage_form)
- Indexes on TEXT columns use prefix lengths (e.g., `brand_name(255)`)
- Foreign key constraints ensure referential integrity

---

### **Phase 4: Load Data into MySQL**

**IMPORTANT:** Before running, update database credentials in `scripts/load_to_mysql.py`:

Open the file and change line ~18:
```python
DB_USER = 'root'              # Your MySQL username
DB_PASSWORD = 'your_password' # CHANGE THIS to your MySQL password!
```

Then run:
```bash
python scripts/load_to_mysql.py
```

**What it does:**
- Connects to MySQL database
- Loads CSV files into corresponding tables
- Uses `if_exists='append'` to avoid foreign key conflicts
- Verifies row counts

**Expected output:**
```
raw_ndc: 128805 rows
raw_ndc_packaging: 244786 rows
raw_drug_shortages: 1810 rows
shortage_contacts: 1810 rows
```

**Time:** 1-3 minutes

**Technical Note:** Script uses `append` mode instead of `replace` to avoid foreign key constraint violations during loading.

---

### **Phase 5: Run SQL Transformations**

Open **MySQL Workbench** and run:
1. **File** → **Open SQL Script**
2. Select `sql/02_transformations.sql`
3. Click the **lightning bolt** ⚡

**What it does:**
- **CRITICAL:** Joins shortage data with NDC data (required SQL transformation!)
- Creates enriched table: `shortages_with_ndc` (~1,810 records)
- Uses `ROW_NUMBER()` to generate shortage_id values
- Creates 4 analysis views for dashboard queries
- Adds indexes for performance

**Expected output:**
- Table `shortages_with_ndc` created with ~1,810 rows
- Match rate: ~90% (shortages successfully matched with NDC data)
- 4 views created: 
  - `current_package_shortages`
  - `multi_package_shortages`
  - `manufacturer_risk_analysis`
  - `current_manufacturer_risk`

**Time:** < 1 minute

**Technical Notes:**
- Uses `ROW_NUMBER() OVER (ORDER BY s.package_ndc)` to generate shortage_id
- Indexes on TEXT columns include prefix lengths
- LEFT JOIN preserves all shortage records, even those without NDC matches

---

### **Phase 6: Run Analysis Queries**

Open **MySQL Workbench** and run:
1. **File** → **Open SQL Script**
2. Select `sql/03_analysis_queries.sql`
3. Click the **lightning bolt** ⚡

**What it does:**
- Executes 10 analytical queries
- Demonstrates insights only possible through joined data

**Expected output:**
- 10+ Result tabs with query outputs
- Click through Result 1, Result 2, etc. to see analysis results
- Examples: top manufacturers, brand vs generic, route analysis, etc.

**Time:** < 1 minute

**Tip:** MySQL Workbench displays results in tabs. Click each "Result" tab to view different query outputs (unlike Jupyter notebooks which display inline).

---

### **Phase 7: Run Interactive Dashboard**

**IMPORTANT:** Update database password in `scripts/dashboard.py` (line ~30)

Then run:
```bash
python -m streamlit run scripts/dashboard.py
```

**What it does:**
- Launches interactive web dashboard
- Opens automatically in your browser at `http://localhost:8501`
- Displays real-time visualizations and metrics

**Features:**
- Key metrics overview (total/current shortages, affected manufacturers)
- Top manufacturers by risk (bar chart)
- Brand vs generic analysis (pie chart)
- Route of administration analysis (bar chart)
- Prescription vs OTC comparison
- Detailed shortage table (searchable, filterable, downloadable)

**To stop the dashboard:** Press `Ctrl+C` in the terminal

**Time:** Dashboard loads in 5-10 seconds

**Technical Note:** If you get "streamlit not recognized", use `python -m streamlit` instead of just `streamlit`

---

## Verification Checklist

After completing all phases, verify:

- [ ] `data/` folder contains 6 files (2 JSON, 4 CSV)
- [ ] MySQL database `fda_shortage_db` exists
- [ ] 4 raw tables contain data (check row counts in Workbench)
- [ ] `shortages_with_ndc` table exists and has ~1,810 rows
- [ ] Match percentage is approximately 90%
- [ ] 4 analysis views exist
- [ ] Analysis queries return results (10+ Result tabs)
- [ ] Dashboard launches and displays data

---

## Data Sources

- **FDA NDC Database:** https://open.fda.gov/apis/drug/ndc/
- **FDA Drug Shortages:** https://open.fda.gov/apis/drug/drugshortages/

See `data/DATA_SOURCE.md` for detailed documentation.

---

## Project Results

### Data Pipeline Statistics
- **128,805** drug products in NDC database (after deduplication)
- **244,786** packaging records (after deduplication)
- **1,810** drug shortage records (after deduplication)
- **~90%** successful match rate (shortages matched with NDC data)
- **1,810** enriched shortage records with full product details

### Key Insights Enabled by Data Join
- Brand name vs generic drug shortage patterns
- Product type analysis (prescription vs OTC vs bulk ingredients)
- Route of administration vulnerability assessment
- Package type distribution in shortages
- Manufacturer portfolio risk analysis

These insights are **only possible** because we joined shortage data with the NDC product database.

---

## Technical Implementation Notes

### Design Decisions
- **CSV intermediate files:** Easier to inspect and debug than direct JSON→MySQL
- **Separate processing script:** Allows re-processing without re-downloading
- **Views instead of tables:** Analysis views are regenerated from source data
- **Streamlit over Power BI:** Better reproducibility, cross-platform, code-based
- **ROW_NUMBER() for IDs:** Generates shortage_id values without AUTO_INCREMENT issues

---

## License

Educational project for ADS-507 at University of San Diego

---

**Last Updated:** February 1, 2026  
**Pipeline Status:** Fully functional and tested on Windows 11  
**Tested On:** Python 3.14.2, MySQL 8.0, Windows PowerShell
