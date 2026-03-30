# CMR Falabella Credit Card Dashboard

Interactive spending and debt analysis dashboard for CMR Falabella credit card statements. Generates a static HTML file served via GitHub Pages.

**Live dashboard:** [ManuelBeckerT.github.io/expenses](https://manuelbeckert.github.io/expenses)

---

## Overview

The project reads raw credit card data from two sources:
1. **Excel files** (`.xlsx`) — one per billing statement, containing individual transactions
2. **PDF files** — billing statements used to extract debt/balance data (carried balance, payments, interest rates)

It produces a single `index.html` dashboard with:
- Month-over-month stacked bar chart (with optional "carried debt" overlay)
- Donut/pie chart with month selector
- Monthly summary table
- Category breakdown with progress bars
- Top 20 merchants table
- Category explorer (per-category monthly chart + transaction drill-down)
- Transaction search
- Debt situation section (balance timeline, billed vs. paid chart, period-by-period table)
- Insights: payment rate, debt growth projection, fixed expense breakdown

---

## Project Structure

```
bank_statement_analysis/
├── raw/                    # Raw Excel statement files (gitignored)
├── raw_pdf/                # Raw PDF statement files (gitignored)
├── build_transactions.py   # Parses raw/ → transactions.csv + summary CSVs
├── generate_dashboard.py   # Reads CSVs + debt_analysis.json → index.html
├── transactions.csv        # One row per transaction (gitignored)
├── monthly_summary.csv     # Monthly spend × category (gitignored)
├── top_merchants.csv       # Top merchants by total spend (gitignored)
├── debt_analysis.json      # Parsed debt data from PDFs (gitignored)
└── index.html              # Generated dashboard (committed, served by GitHub Pages)
```

---

## Setup

### Dependencies

```bash
pip install openpyxl pdfplumber --break-system-packages
```

Python 3.13+ is required. No other dependencies — the dashboard uses Chart.js from CDN.

### Raw data

Place files in the correct directories:
- Excel statements → `raw/` (one `.xlsx` per billing period)
- PDF statements → `raw_pdf/` (one `.pdf` per billing period)

Neither directory is committed to git (see `.gitignore`).

---

## Workflow

### Step 1 — Parse transactions

```bash
python3 build_transactions.py
```

Reads all `.xlsx` files in `raw/` and outputs:
- `transactions.csv` — one row per transaction with fields: `fecha`, `year`, `month`, `month_label`, `descripcion`, `titular`, `monto_total_clp`, `cuotas_pendientes`, `valor_cuota_clp`, `tipo_pago`, `tipo_movimiento`, `categoria`, `source_file`
- `monthly_summary.csv` — total spend per month × category
- `top_merchants.csv` — merchants sorted by total spend

**Duplicate file handling:** The script skips known duplicate Excel files via the `SKIP_FILES` set at the top of the file.

**Categorization:** Transactions are auto-categorized using keyword rules in `CATEGORY_RULES`. Each transaction is matched against keywords (case-insensitive, first match wins). Unmatched transactions fall into `"Otros"`.

### Step 2 — Extract debt data from PDFs

Run the inline PDF parsing script (or a standalone version) using `pdfplumber` to read each PDF and extract:
- `periodo_desde` / `periodo_hasta` — billing period dates (format `DD/MM/YYYY`)
- `saldo_inicio` — carried balance at start of period
- `total_facturado` — total billed for the period
- `pagado` — amount paid
- `saldo_final` — balance carried to next period (`total_facturado - pagado`)
- `tasa_interes_pct` — monthly interest rate
- `pago_minimo` — minimum payment due

Output is saved as `debt_analysis.json`:

```json
{
  "periods": [
    {
      "periodo_desde": "25/02/2025",
      "periodo_hasta": "24/03/2025",
      "saldo_inicio": 5800000,
      "total_facturado": 12400000,
      "pagado": 8000000,
      "saldo_final": 4400000,
      "tasa_interes_pct": 2.18,
      "pago_minimo": 350000
    },
    ...
  ]
}
```

### Step 3 — Generate dashboard

```bash
python3 -c "exec(compile(open('generate_dashboard.py').read().replace(\"__file__\", \"'generate_dashboard.py'\"), 'generate_dashboard.py', 'exec'))"
```

> **Why this command?** Python 3.13 raises `SystemError: Negative size passed to PyUnicode_New` when compiling very large f-string scripts as top-level modules. The `exec(compile(...))` workaround avoids this bytecode limit.

Reads `transactions.csv` and `debt_analysis.json`, then writes `index.html`.

### Step 4 — Publish

```bash
git add index.html
git commit -m "Regenerate dashboard"
git push
```

GitHub Pages serves `index.html` from the `main` branch root.

---

## Data Model

### Billing periods vs. calendar months

CMR Falabella billing periods run from the **25th of one month to the 24th of the next**. The dashboard maps each billing period ending `24/MM/YYYY` to calendar month `YYYY-MM` for alignment with transaction data.

### Transaction types

| `tipo_movimiento` | Description |
|---|---|
| `Gasto` | Regular purchase |
| `Devolucion` | Refund (negative `valor_cuota_clp`) |
| `Pago` | Card payment (excluded from spend analysis) |

### Payment types

| `tipo_pago` | Description |
|---|---|
| `Contado` | Paid in full at purchase |
| `Cuotas` | Installment purchase (`cuotas_pendientes > 0`) |

`valor_cuota_clp` is the amount charged in the current period (installment amount if in cuotas, full amount if contado). This is the field used for all spend calculations.

---

## Dashboard Sections

### KPI cards
Seven summary metrics in a single row: total spent, monthly average, monthly median, transaction count, estimated fixed expenses, discretionary average, and installment ratio.

### Debt situation (⚠️)
Highlighted section showing:
- Current carried balance and interest rate
- Estimated interest accumulated over 12 months (computed as `saldo_inicio × tasa_interes_pct` per period)
- Balance timeline chart (line chart of `saldo_inicio` per period)
- Billed vs. paid grouped bar chart per period
- Full period-by-period table with % paid column

### Monthly bar chart
Stacked bar chart sorted per month (largest category at bottom). Includes:
- **Deuda arrastrada** — `saldo_final` of each billing period injected as a pseudo-category (dark red). Toggle via checkbox.
- Median and average reference lines (include carried debt in the calculation).
- Custom canvas plugin for per-month sorting (Chart.js native stacking always uses the same order).
- Custom tooltip showing sorted breakdown on hover.

### Category breakdown
Horizontal progress bars for all spending categories (excluding "Deuda arrastrada", which is not a spending category). Bar width is scaled to 3× for readability (max 100%).

### Category explorer
Search and select any category to see:
- Month-by-month bar chart for that category
- All transactions grouped by merchant, filterable by month

### Transaction search
Free-text search across all transactions. Results show a monthly bar chart and a grouped merchant table.

### Insights
**Panel 1 — "¿Cuánto necesita Titular realmente?"**
- Essential recurring expenses (supermercado, farmacia, combustible, condominio)
- Fixed contracted expenses (seguros, golf, servicios digitales, donaciones)
- Discretionary expenses (viajes, restaurantes, ropa, cosmética, vinos)
- Items to review (education, golf membership, insurance costs)

**Panel 2 — Debt and commitment analysis**
- % of bill paid each month (average across all settled periods)
- Debt growth rate (+137% in 12 months) with linear projection to credit limit
- Monthly fixed burden breakdown: unavoidable contracts + optional contracts + essentials + interest cost
- **Debt projection chart** — line chart combining historical saldo data with a 12-month linear regression projection. Includes a reference line at the credit limit ($30M).

---

## Customization

### Adding categories

Edit `CATEGORY_RULES` in `build_transactions.py`. Each rule is a tuple of `(category_name, [keywords])`. Keywords are matched case-insensitively as substrings of the transaction description. First match wins — order matters.

After editing, re-run `build_transactions.py` then `generate_dashboard.py`.

### Skipping duplicate files

Add the filename to `SKIP_FILES` in `build_transactions.py`:

```python
SKIP_FILES = {"5321a041-b604-4ab6-8e81-d381c85142da.xlsx"}
```

### Excluding months from baseline calculations

Edit `SKIP_FOR_BASELINE` in `generate_dashboard.py`:

```python
SKIP_FOR_BASELINE = {"2025-04"}  # partial month
```

This affects the median and average reference lines in the bar chart.

### Credit limit

Update `CUPO_COMPRAS` in `generate_dashboard.py` (default: `30_000_000` CLP).

---

## .gitignore

```
raw/
raw_pdf/
debt_analysis.json
*.py
*.csv
__pycache__/
```

Only `index.html` and `README.md` are committed. All raw data, scripts, and intermediate CSVs stay local.
