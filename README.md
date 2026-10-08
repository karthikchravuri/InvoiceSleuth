# InvoiceSleuth

**Find where your money is leaking.**

InvoiceSleuth checks every invoice against its **purchase order, contract, delivery receipt and the vendor's past invoices** before it is paid. When something doesn't add up, it explains *what* happened, *why* it is suspicious, *how much money* is at risk, and *what to do next*.

> Hackathon prototype. All vendors, invoices and amounts are **synthetic demo data**.

---

## The demo in one picture

Search `INV-1025` →

```
Contract   ₹50,000 / laptop
   ↓
Invoice    ₹57,500 / laptop      ← 15% higher than the contract
   ↓
Quantity   10 laptops            ← matches PO and delivery
   ↓
₹75,000 potential loss  →  Payment Hold Recommended
```

---

## Run it (≈ 5 minutes)

You need **Python 3.10+** and **Node.js 18.18+**.

### 1. Backend (FastAPI, port 8000)

```bash
cd backend
python -m venv .venv
# macOS / Linux:
source .venv/bin/activate
# Windows:
.venv\Scripts\activate

pip install -r requirements.txt
uvicorn main:app --reload --port 8000
```

The SQLite database (`data/invoicesleuth.db`) is created and filled with demo data automatically on first start.

### 2. Frontend (Next.js, port 3000)

In a second terminal:

```bash
cd frontend
npm install
npm run dev
```

Open **http://localhost:3000** and search `INV-1025`.

> The frontend forwards `/api/*` to the backend, so there are no CORS issues. If your backend runs somewhere else, copy `.env.example` to `frontend/.env.local` and change `BACKEND_URL`.

---

## 3-minute demo script

1. **Open** InvoiceSleuth — one search box, nothing else to explain.
2. **Search** `INV-1025` (or click *Try the demo*).
3. **Point at the top card:** *HIGH RISK — ₹75,000 potential loss.* "The invoice price is 15% higher than the active contract."
4. **Why is this suspicious?** Quantity ✓, delivery ✓, but contract price ⚠. Root cause + financial impact.
5. **Evidence:** Contract → PO → Invoice → Delivery. Only the invoice price disagrees.
6. **Vendor history:** ABC normally charges ₹49K–₹52K. This invoice: ₹57.5K (red bar).
7. **ML analysis:** Isolation Forest independently says *high deviation from normal vendor behavior*.
8. **Risk score 87/100** — every point comes from a named check, nothing mysterious.
9. Click **Place Payment Hold** → status changes to PAYMENT HOLD with a timestamp.
10. **Recovery** appears → **Generate Dispute** → edit → **Approve** → **Send Demo**.

Closing line: *"InvoiceSleuth doesn't just flag an invoice. It explains where the money is leaking and helps stop the loss."*

**Before presenting again:** click *Reset demo data* in the footer (or `POST /api/demo/reset`) to put everything back.

### Other planted cases worth showing if judges ask

| Invoice  | Problem | Risk |
| -------- | ------- | ---- |
| INV-1025 | Price 15% above contract | HIGH |
| INV-1058 | Duplicate of INV-1041 (same PO + amount) | HIGH |
| INV-1063 | Billed 80 LED panels, only 64 delivered | HIGH |
| INV-1090 | Bills "Wireless headset" — not on the PO | HIGH |
| INV-1072 | 5× the vendor's usual order size (caught by ML + history, all rules pass) | MEDIUM |
| INV-1081 | GST charged at 28% instead of the contract's 18% | MEDIUM |
| INV-1027 | Price 5% above contract | MEDIUM |
| INV-1026 | Everything matches | LOW |

**Upload:** the Upload page has two sample PDFs (also in `data/samples/`). `INV-1101` comes out clean; `INV-1102` bills 12 monitors when only 9 were delivered.

---

## How it works

### 1. Rule checks (three-way match + extras)

For each invoice the backend loads its PO, the vendor's active contract, the delivery receipt (GRN) and other invoices from the same vendor, then checks:

| Check | Fails when |
| ----- | ---------- |
| Quantity | invoiced quantity > PO quantity |
| Delivery | invoiced quantity > quantity received |
| Item | invoiced item is not the PO/contract item |
| Contract price | unit price > contract price + 2% tolerance |
| Duplicate | same vendor + same PO + same amount already invoiced |
| Invoice total | > 2.5× the vendor's largest past invoice |
| GST | wrong rate, CGST/SGST vs IGST mix-up, or invalid GSTIN checksum |

### 2. ML check — Isolation Forest (scikit-learn)

* Trained on 300 invoices (200 past invoices + 100 from the current quarter).
* 3 features per invoice, each measured against **that vendor's own history**: unit price, quantity, invoice amount (as "how many normal spreads away from the vendor's average").
* Invoices that are easy to isolate from the rest get a low anomaly score → "high deviation from normal vendor behavior".
* The UI also says which feature drove it (price / quantity / amount).
* Anomaly ≠ fraud. The UI says so.

### 3. Risk score (transparent, not a black box)

The score is the **sum of points from the checks that failed**, capped at 100:

| Factor | Points |
| ------ | -----: |
| Price deviation (>10% / 3–10%) | 35 / 18 |
| Contract mismatch | 30 |
| Billed for goods not received | 65 (30 if ≤10% short) |
| Quantity above PO | 40 |
| Item not on PO | 50 |
| Duplicate invoice | 70 |
| GST issue | 35 |
| Unusual invoice amount | 30 |
| Vendor history deviation (ML) | 0–25 |

**65+ = HIGH**, **30+ = MEDIUM**. INV-1025: 35 + 30 + 22 (ML) = **87**.

### 4. Potential loss

The money that would be overpaid: price difference × quantity, undelivered units × price, the whole invoice for a duplicate or unordered item, or the excess GST.

---

## Project structure

```
invoice-sleuth/
├── backend/
│   ├── main.py           FastAPI routes
│   ├── analysis.py       checks, Isolation Forest, risk score, explanations
│   ├── seed.py           deterministic synthetic data (20 vendors, 100 invoices, …)
│   ├── extract.py        local PDF text reader for uploads
│   ├── db.py             SQLite schema
│   ├── make_samples.py   regenerates the sample invoice PDFs
│   └── requirements.txt
├── frontend/             Next.js + TypeScript + Tailwind + Recharts + Lucide
│   ├── app/              /, /invoice/[id], /invoices, /investigations, /vendors, /upload
│   ├── components/
│   └── lib/api.ts
├── data/
│   └── samples/          sample invoice PDFs for the upload demo
├── .env.example
└── README.md
```

## API (for the curious)

| Method | Path | |
| ------ | ---- | - |
| GET | `/api/search?q=` | invoice number, vendor name, or "high risk" |
| GET | `/api/invoices` | all invoices with risk + status |
| GET | `/api/invoices/{no}` | full investigation |
| POST | `/api/invoices/{no}/action` | `{"action": "hold" \| "approve" \| "investigate"}` |
| POST / PUT | `/api/invoices/{no}/dispute` | generate / edit / approve / send (demo) |
| GET | `/api/summary` | investigations dashboard |
| GET | `/api/vendors`, `/api/vendors/{id}` | vendor list and price history |
| POST | `/api/upload`, `/api/invoices` | read an uploaded file, then check it |
| POST | `/api/demo/reset` | restore the demo data |

Interactive docs: http://localhost:8000/docs

## Honest limitations

* **Synthetic data only.** No real company or financial data.
* **Payment hold and dispute emails are simulated.** No bank, ERP or email is connected.
* **GST validation is local** (rates, CGST/SGST vs IGST, GSTIN format + checksum). Government GST API is not connected.
* **Uploads:** text-based PDFs are read locally with `pypdf`. Image OCR is not included, so PNG/JPG fields are typed in by the user.
* **No login / multi-company support** — out of scope for a hackathon prototype.

## Tech

Next.js 15 · TypeScript · Tailwind CSS · Recharts · Lucide · FastAPI · SQLite · scikit-learn (Isolation Forest) · pypdf
