# POS & Sales Tracker — Carwash & Auto Detailing

A full point-of-sale and sales-tracking system built entirely on **Google Apps Script + Google Sheets**, designed for a carwash and auto detailing business. It handles everything from ringing up a transaction to printing a receipt, with Google Sheets acting as the live, zero-cost database.

No external hosting, no monthly SaaS fee, no separate database — just a Google Sheet, a Web App deployment, and vanilla JavaScript.

---

## Why I built this

Small service businesses (carwashes, detailing shops, etc.) are often stuck choosing between expensive POS subscriptions or a messy paper logbook. This project shows that a lightweight, fully custom POS can be built on tools most small business owners already have access to — Google Sheets — while still supporting real-world POS features like size-based pricing, packages, discounts, and receipt printing.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Google Apps Script (`Code.gs`) |
| Data store | Google Sheets |
| Frontend | HTML, CSS, vanilla JavaScript (`Index.html`, `Receipt.html`, `CSS.html`) |
| Deployment | Google Apps Script Web App |
| Client-server calls | `google.script.run` (async RPC to Apps Script) |

Everything runs inside Google's ecosystem — Sheets doubles as the database, and the Apps Script Web App serves the UI, meaning the entire system can be deployed for free.

---

## Core Features

### 🧾 Point-of-Sale Interface
- Clean, mobile-friendly single-page POS UI for staff to process transactions in real time.
- Live "New Transaction" indicator in the header showing the next transaction number before it's even submitted.
- One-click shortcut to open the Receipt Printing page in a new view.

### 🚗 Customer & Vehicle Intake
- Optional customer name, contact number, and address fields for walk-ins or regulars.
- Car brand dropdown covering 25+ major brands plus a dedicated motorcycle/tricycle category (Honda, Yamaha, Suzuki, Rusi, Ducati, Kawasaki, TVS, KTM, CFMOTO) — built specifically around the vehicle mix a real carwash sees.
- Car model/color free-text field and **required** plate number field to keep every transaction traceable.
- Required "Assigned Employee" selector for accountability and future performance tracking.

### 📏 Size-Based Dynamic Pricing
- Six vehicle-size tiers (Small, Medium, Big, Large, X-Large, XX-Large), each mapped to its own price column in the Services/Packages sheet.
- Selecting a vehicle size instantly re-prices every visible service and package — no page reload, no manual lookup.

### 🧽 Services, Packages & Add-Ons
- **Individual Services** and **Packages** are pulled live from separate Google Sheet tabs, so the business owner can add, rename, or reprice offerings just by editing a spreadsheet — no code changes required.
- Optional **Add-Ons** sheet is fully self-detecting: if the tab doesn't exist yet, the Add-Ons section gracefully shows "no add-ons available" instead of breaking the app.
- Products are filtered by an `Active` flag and ordered with a `SortOrder` column, giving the owner full control over what's shown and in what order — again, entirely from the spreadsheet.
- Automatic transaction type detection (`Individual`, `Package`, or `Mixed`) based on what's actually in the cart.

### 🛒 Order Summary & Cart
- Running cart with live item list, matched against vehicle size and category.
- Editable **percentage-based discount** with automatic subtotal/discount/total recalculation.
- Clear breakdown of Subtotal → Discount → Total before checkout.

### 💵 Payment Handling
- Multiple payment methods: Cash, GCash, Maya, Card.
- Real-time **change calculation** as the cashier types in the amount received.
- Automatic transaction status: a sale is marked **Completed** once amount received covers the total, or **Pending** if it doesn't — no manual status toggling needed.

### 🔢 Auto-Incrementing Transaction IDs
- Sequential, human-readable transaction IDs (e.g. `DEMO-000001`, `DEMO-000002`, …) generated and persisted through a `Settings` sheet, with a configurable prefix — so this can be rebranded per business without touching code.

### 🧾 Receipt Printing System
- Dedicated **Receipt Printing** page, separate from the POS screen, so a transaction can be reprinted anytime by ID — even after the till has moved to the next customer.
- Search any transaction by ID and instantly render a formatted, print-ready receipt: business header, itemized list, vehicle details, discount line, totals, payment method, amount received, and change.
- Print-optimized layout (`.no-print` UI elements disappear automatically when the browser print dialog is triggered) so the printed receipt only shows the receipt itself.
- Friendly error states for invalid or not-found transaction IDs, and for backend/network failures.

### 📊 Live Sales Snapshot
- Backend function aggregates **today's total sales and transaction count** on demand, filtering only `Completed` transactions and normalizing dates to the script's timezone — foundation for a dashboard or daily close-out report.

### 🛡️ Data Integrity & Resilience
- Every sheet read validates that required columns exist before processing, throwing a clear, human-readable error (e.g. `Column "ServiceID" is missing from "Services" sheet`) instead of failing silently or crashing.
- All numeric fields are sanitized through a shared `toNumber_()` helper that strips currency symbols/commas and safely defaults to `0`, so malformed spreadsheet input never breaks a transaction.
- Every value rendered into HTML is passed through an `escapeHtml()` function on both the POS and Receipt pages, protecting against injection from customer-entered fields.
- Historical schema safety: discount fields (`Subtotal`, `DiscountPercent`, `DiscountAmount`) were deliberately appended at the *end* of the Transactions sheet so upgrading the system never shifts or breaks existing columns/data.

### 🧩 Modular, Spreadsheet-Driven Configuration
- Business rules live in the spreadsheet, not the code: services, packages, add-ons, pricing tiers, transaction number prefixes, and starting transaction numbers are all editable by a non-technical shop owner directly in Google Sheets.
- Multi-page routing handled through a single Apps Script entry point (`doGet`), switching between the POS view and the Receipt view via a `?page=` URL parameter — with a graceful fallback error page if the app fails to load.

---

## System Architecture

```
Google Sheet (data + config)
   ├─ Services            → individual service catalog & size-based pricing
   ├─ Packages             → bundled service catalog & size-based pricing
   ├─ AddOns (optional)    → optional extras, auto-detected
   ├─ Transactions         → one row per sale (customer, vehicle, totals, payment)
   ├─ TransactionItems     → one row per line item per sale
   └─ Settings             → transaction ID prefix/counter, key-value config
            │
            ▼
Google Apps Script (Code.gs)
   ├─ doGet()              → routes to POS or Receipt page
   ├─ getServices/Packages/AddOns()  → live catalog + pricing
   ├─ saveTransaction()    → validates, computes totals, writes rows
   ├─ generateTransactionID() → sequential ID generation
   ├─ getTransactionByID() → receipt lookup
   └─ getTodaySales()      → daily sales aggregation
            │
            ▼
Frontend (HTML/CSS/JS)
   ├─ Index.html   → POS terminal UI
   └─ Receipt.html → receipt search & print UI
```

Communication between frontend and backend uses Apps Script's built-in `google.script.run` async bridge — no external API, no auth tokens, no server to maintain.

---

## Example Flow

1. Cashier opens the POS page and selects a vehicle size (e.g. *Large*).
2. Prices for every service and package instantly update to the Large-tier price.
3. Cashier adds a "Premium Wash" package and a "Tire Black" add-on to the cart.
4. A 10% loyalty discount is applied — subtotal, discount, and total recalculate live.
5. Cashier selects GCash as the payment method and enters the amount received; change is computed instantly.
6. On submit, the system generates transaction `DEMO-000042`, saves the transaction and its line items to the spreadsheet, and marks it Completed.
7. Later, staff open the Receipt Printing page, search `DEMO-000042`, and print a formatted receipt for the customer.

---

## What This Project Demonstrates

- Designing a relational-style data model (Transactions ↔ TransactionItems) on top of a non-relational tool like Google Sheets.
- Building a dynamic pricing engine driven entirely by spreadsheet configuration.
- Writing defensive backend code: schema validation, input sanitization, and graceful degradation for optional features.
- Structuring a multi-page Apps Script web app with clean client/server separation.
- Practical UX details for a real cashier workflow: keyboard shortcuts (Enter to search), print-specific styling, live totals, and clear error messaging.

---

## Notes

This is a demo/portfolio build (transaction prefix `DEMO`, placeholder business name/address). The underlying pattern — Sheets as database, Apps Script as backend, HTML/JS as frontend — can be repointed to any service-based business (salons, laundry shops, repair shops, etc.) by editing the Services/Packages sheets and swapping the branding in `Index.html` and `Receipt.html`.
