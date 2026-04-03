# EKO Transactions Data Pipeline

## Overview

This project implements a **data pipeline** that processes fuel transaction data and enriches it with pricing information.  
The final result is an **Excel report grouped by company**, containing calculated totals based on both:

- EKO price (original transaction price)
- GTA price (company-specific negotiated price)

---

## Pipeline Architecture

```
Excel Files → Data Cleaning → SQLite Database → SQL Merge → Excel Report
```

### Main components:
- **transactions** → raw fuel transactions
- **prices** → company price lists over time
- **cards** → mapping between card numbers and companies

---

## 📂 Project Structure

```
project_root/
│
├── 2_DATABASE/
│   └── eko_cards.db
│
├── 3_PROJECT_DATA/
│   ├── eko_transactions.xlsx
│   ├── prices.xlsx
│   └── cards.xlsx
│
├── A_FINAL_RESULT/
│   └── eko_transactions.xlsx
│
└── pipeline.py
```

---

## Pipeline Flow (Step by Step)

### 1. Import Transactions (`transaction_import()`)

Reads raw transaction data from Excel and performs cleaning.

#### Key operations:
- Removes product codes from `Material`
  ```
  "110000000 Diesel EKONOMY" → "DIESEL EKONOMY"
  ```
- Normalizes text (uppercase, trimmed spaces)
- Cleans card numbers (removes non-digits)
- Converts dates to standard format (`YYYY-MM-DD`)
- Filters invalid rows:
  - Missing dates
  - Zero or negative quantities

#### Output:
Data is inserted into SQLite table:
```
transactions
```

---

### 2. Import Prices (`price_import()`)

Reads company pricing data.

#### Key operations:
- Standardizes:
  - company names
  - product names
- Converts dates and prices
- Ensures `eik` column exists in DB
- Inserts data safely into:
```
prices
```

#### Important:
Each price has:
- `eik` (company identifier)
- `product`
- `date`
- `final_price`

---

### 3. 🏢 Import Company Mapping (`company_import()`)

Maps card numbers to companies.

#### Source:
```
cards.xlsx
```

#### Output table:
```
cards (card_number → company)
```

---

### 4. 🔗 Merge Data (`merge_tables()`)

This is the **core logic of the pipeline**.

#### SQL Logic:

```sql
SELECT MAX(p2.date)
FROM prices p2
WHERE p2.eik = c.eik
  AND p2.product = t.material
  AND p2.date <= t.date
```

### 📌 What this does:

For each transaction:
- Finds the **latest price BEFORE or ON the transaction date**

#### Example:

| Price Date | Price |
|----------|------|
| 01.04    | 1.60 |
| 05.04    | 1.64 |
| 10.04    | 1.70 |

Transaction on:
- **03.04 → 1.60**
- **06.04 → 1.64**

✔ Correct historical pricing

---

### 💡 Join Conditions:

```
transactions → cards → prices
```

- `t.card_number = c.card_number`
- `c.eik = p.eik`
- `p.product = t.material`

---

### 🔢 Price Rounding

```sql
CAST(p.final_price * 100 + 0.5 AS INTEGER) / 100.0
```

✔ Ensures:
- 1.605 → 1.61  
- 1.604 → 1.60  

---

### 5. 📄 Generate Final Report (`generate_result()`)

Creates Excel output grouped by company.

#### For each company:
- Separate sheet
- Max 31 characters (Excel limit)

#### Calculations:

```
eko_total = bill_qty * price
gta_total = bill_qty * final_price
```

#### Final Columns:

| Column | Description |
|------|------------|
| Дата | Transaction date |
| Станция | Fuel station |
| Име | Vehicle |
| Номер на карта | Card number |
| Име на артикул | Product |
| Литри | Quantity |
| GTA цена | Negotiated price |
| ЕКО цена | Original price |
| Сума по GTA цена | Total (GTA) |
| Сума по ЕКО цена | Total (EKO) |

---

## How to Run

### Execute full pipeline:

```python
execute_pipeline()
```

### What happens:
1. Deletes old transactions
2. Imports new transactions
3. Merges with prices
4. Generates final Excel

---

## Maintenance Functions

### Delete all data:
```python
delete_transactions()
delete_prices()
delete_company()
```

### Delete prices for specific month:
```python
delete_prices_by_month(2026, 3)
```

---

## Key Considerations

### 1. Data Consistency
- Product names must match between:
  - transactions
  - prices

### 2. Date Logic
- Prices must be historical (not future)
- Always uses:
```
MAX(date <= transaction_date)
```

### 3. Company Matching
- Based on `eik` (most reliable)
- Avoid relying only on names

---

## Summary

This pipeline:

✔ Cleans raw Excel data  
✔ Stores structured data in SQLite  
✔ Applies correct historical pricing logic  
✔ Generates professional Excel reports  

---

## Future Improvements

- Add validation layer (missing matches)
- Add logging instead of print()
- Detect unmatched transactions
- Performance optimization for large datasets