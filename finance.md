# Personal finance: automated import and reconciliation

Planning notes from a brainstorm (2026-11-11 to 2026-11-19), to continue in the new repository.

## Goal

- Tracking finances feels tedious. Finances are solid; the purpose is **looking back at
  spending** (distribution over categories) and occasionally the main account's balance to
  judge adjusting regular savings / house-loan down payments. No tight budgeting.
- **Very high degree of automation**, tolerant of downloading bank statements in **batches of
  several months**.
- **Automatic verification** of the ledger against the bank statements ("abgleichen").
- **View finances on the mobile** — being limited to the Linux desktop is a drawback today.
- Owning the data is a nice plus, not the main driver.
- **Hard requirement:** complete import of the existing history — no loss of historic data.

## Current setup

- KMyMoney (previously GnuCash; the GnuCash history was imported, with some manual fixes in
  the file). Default `.kmy` format (gzipped XML); not attached to it.
- `.kmy` files live in an ownCloud-synced folder (hosted ownCloud, strong password + 2FA),
  reachable from the mobile, but there's no way to view them there.
- History since **2005**, **30+ accounts**, most of them closed today.
- Every transaction is typed in manually from downloaded PDF bank statements — the main pain.

### Accounts and statements

- Main accounts at **DNB** (NOK, Norwegian) and **N26** (EUR); monthly statements.
- A few **savings accounts**; statements cover longer periods.
- **All statements come as PDF** — the format kept for archiving. `pdftotext -layout` yields
  usable text for both DNB and N26 statements.
- No online banking access (FinTS) for either bank.
- No credit cards; debit cards work directly on the main accounts.
- Cash: tracked only loosely, barely used.
- Investments: a few funds and stocks in the past; currently **one stock in Norway**, investor
  statements from **VPS** (Euronext VPS) as PDF — maybe other formats too (unknown).
  Fund/stock transactions were entered manually; once investing again, automation is wanted,
  and importing from the downloaded PDF would be good enough.
- A house loan (down payments).

## Decisions

| Topic | Decision | Why |
|---|---|---|
| Ledger | **beancount** format, replacing KMyMoney | Balance assertions per statement make reconciliation a by-product; built for scripted import; plain text allows manual fixes in an editor and history in git |
| UI | **fava** | Online demo judged good enough to try: category reports, balances over time, queries |
| Hosting | **fava on Fly.io, scale-to-zero**, like the Trip Archive | Mobile access while the notebook is off |
| Auth | **Basic auth, password only**, over HTTPS + unguessable hostname | Fava has no login of its own; TOTP judged unnecessary |
| Language | **Rust**, static binaries | No Python fan; no local Python environment to maintain |
| Repository | **Own repo** | — |

### Rejected alternatives

- **CSV import into KMyMoney** (both banks offer CSV downloads): rejected — downloading the
  PDFs is tedious enough, DNB's web UI is poor; no second download per statement.
- **Keep KMyMoney + own import/verify tool:** marking transactions reconciled automatically
  would mean writing into the `.kmy` file; stays desktop-only and GUI-centric.
- **hledger:** same idea, but a weaker web UI than fava.
- **Own app:** not worth it; beancount + fava cover the needs.
- **Fava at home behind a VPN:** the only always-on device is a Home Assistant box, kept for
  home automation only.
- **Static reports (HTML/PDF) into ownCloud:** simplest and offline, but fixed reports without
  drilling down. Remains a fallback.

## Target workflow

1. Download statement PDFs whenever convenient (several months at once is fine) into an inbox
   folder.
2. Run one command: everything is imported, categorized and verified.
3. Only exceptions need attention (unknown category, failed balance check, missing statement).
4. The ledger in ownCloud is current; fava on Fly.io shows it on desktop and mobile.

## Build order

1. **`.kmy` → beancount converter** for the full history. Biggest risk first; KMyMoney stays
   in use until it passes.
2. **PDF importers** for DNB and N26.
3. **Categorization.**
4. **Fava on Fly.io.**
5. Later: **VPS importer** for stock transactions; savings-account statements (all PDF) as
   their bank's importer allows.

## Design notes

### Tool stack

- The beancount **file format is the interface**; our tools only read and write text.
- **Validation locally: rustledger** — a Rust rewrite of beancount, claims full syntax
  compatibility, ships `bean-check` / `bean-query`-compatible commands (GPL-3.0). Not tried
  yet: verify it checks the converted ledger like beancount does.
- **PDF text:** call poppler's `pdftotext` (already works on the statements);
  `-bbox-layout` gives word coordinates for column detection. Preferred over a pure-Rust PDF
  crate.
- **Converter:** `.kmy` = gzip + XML → `flate2` + `quick-xml`.
- **Fava** stays Python, but only inside the Fly Docker image with a pinned version — nothing
  local; ongoing work is bumping the version now and then. **rustfava** (Fava-compatible, from
  the rustledger organization) exists; maturity unverified, rustledger's docs still point to
  Python fava for the web UI.

### 1. Converter (`.kmy` → beancount)

- Read accounts, payees, transactions with splits, securities and prices from the gzipped XML.
- Closed accounts get `close` directives: out of current views, history stays in reports.
- NOK and EUR accounts; funds/stocks as commodities with prices; house loan as a liability.
- **Verification:** every account's balance at each month end must equal KMyMoney's. Only
  then is the conversion accepted.
- Keep the `.kmy` archived read-only as a fallback.

### 2. PDF importers

- One importer per bank; each emits neutral records (date, amount, text, balance) and then
  beancount transactions.
- **DNB** (kontoutskrift, NOK, Norwegian): separate "Ut av konto" / "Inn på konto" columns →
  need column positions; foreign card purchases add currency/exchange-rate lines.
- **N26** (EUR): descriptions span several lines (payee, IBAN, reference).
- **Balance assertion per statement** = automatic reconciliation, also for savings accounts
  with long statement periods.
- **Safety net:** parsed transactions must sum to closing − opening balance, else stop
  (catches layout changes and missed lines).
- **Batch-safe:** process statements per account in date order; each statement's opening
  balance must equal the previous one's closing balance (finds a missing statement); remember
  imported statements (account + number/period) so re-runs are harmless; duplicate detection.
- **Transfers DNB ↔ N26** (NOK ↔ EUR) appear on both statements → must become one transfer.
- Effort estimate: a few evenings for both banks; then seconds per statement.

### 3. Categorization

- Learn payee → most frequent category from the converted history (20 years of categorized
  data); a rules file (payee/text pattern → category) overrides.
- Unknown → `Expenses:Uncategorized`, reviewed in bulk or ignored (coarse is fine for looking
  back).

### 4. Hosting (fava on Fly.io)

- Reuse the Trip Archive's Fly setup: `auto_stop_machines = "stop"`,
  `min_machines_running = 0`.
- Auth proxy (e.g. Caddy basic auth) in front of fava.
- The ledger stays in **ownCloud as the single source**; the machine fetches it via WebDAV on
  boot → always current, no deploy per import.
- ownCloud app passwords grant access to the whole account → use a **separate ownCloud user**
  with only a **read-only share of the finance folder**; its app password is a Fly secret.
- Cold start = Python start + parsing ~20 years of ledger: expected seconds, to be measured.
- Fava works in a phone browser but isn't designed for small screens.

## Privacy

- Real statements and test fixtures stay local, outside any repo.
- A sample statement shown to an AI assistant leaves the machine → share only the layout with
  names, IBANs and amounts replaced by dummy values.

## Open questions

- Does VPS offer investor statements in a format other than PDF?
- Which bank(s) hold the savings accounts — do they need importers of their own?
- Does rustledger validate the converted ledger exactly like beancount (check early)?
- How mature is rustfava — could it replace Python fava on the server later?
