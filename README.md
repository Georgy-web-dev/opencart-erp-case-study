# OpenCart ERP — case study

Most OpenCart stores outgrow OpenCart. It handles a catalogue and a checkout well, and
then the business needs warehouses, costing, supplier debts, multi-currency ledgers and
a CRM — and the usual answer is to buy a separate ERP and spend the next year keeping
two systems in sync.

This was the other answer: build that ERP layer **as OpenCart modules**, inside the
admin panel the staff already use, against one database. Delivered for a live wholesale
store with real orders running through it the whole time.

**This repository contains no source code.** The work was done under a client
engagement, so what follows is an architecture and delivery write-up with redacted
screenshots. Happy to walk through the implementation in an interview.

| | |
|---|---|
| **Role** | Sole developer — design, build, deployment |
| **Duration** | July – August 2026 |
| **Platform** | OpenCart 3.0.3.9, PHP, MySQL, Twig, jQuery |
| **Scale** | ~70,000 lines of PHP across 273 files; 44 installable submodules |
| **Delivery** | 12 numbered modules, each deployed and verified on the live store |

---

## What was built

### ERP core

![ERP dashboard](img/erp-dashboard.png)

A dashboard fronting the whole system — action shortcuts, section counters, and live
figures for units on hand, stock value at moving-average cost, receivables and payables.

**Inventory**
- Warehouses and cells, with manual order fulfilment
- Stock balances across the whole catalogue, stock adjustments, low-stock reorder alerts
- **Reservations** — stock is held at order time rather than at dispatch, which is what
  actually fixed the overselling
- **Packages / boxes** — derived from order lines rather than stored, so box contents
  can never drift out of sync with what was sold
- Goods receipt with **moving-average costing**, receipt journals and line-level detail

![Warehouses](img/warehouses.png)

**Finance**
- Cash accounts, cashflow, bank statements, payment documents
- Debt ledger — who owes us and who we owe, per currency, with a single-currency roll-up
- Multi-currency with automatically refreshed FX rates, and no cron dependency
- Purchase orders, sales orders, sales invoices, goods issue, returns
- Product tax with HSN codes, and generated print forms

**CRM**

![Counterparty CRM](img/counterparty-crm.png)

Counterparties unified buyers and suppliers into one record with contacts, contracts,
groups, per-counterparty currency and a **credit limit that warns rather than blocks** —
a deliberate call, because a hard block would have stopped legitimate trade at the worst
possible moment. Existing OpenCart customers link into counterparties, which closed a
long-standing duplicate-record problem without a destructive migration.

### Multi-channel messenger

![Messenger channels](img/messenger-channels.png)

A unified inbox on the order screen: **WhatsApp** (via a self-hosted Evolution API on
Docker), **Telegram**, **Viber** and **email over SMTP**. Text, voice notes, images,
video and document attachments, with unread badges surfaced in the order list.
Conversations attach to the order they belong to, so context stops living on someone's
phone.

### Shipping

Courier API integration — AWB generation, live status polling, cancellation, and webhook
push for status updates. Validated against the real API rather than mocks.

---

## Engineering notes

**OCMOD matches one line at a time.** OpenCart 3 splits target files on `\n` and runs
`stripos` per line, so a multi-line `<search>` block can never match — and it fails
*silently*: no error, no log entry, the operation is skipped and the file simply never
gets patched. Order-window panels didn't render for a week because their anchor was a
three-line block. Every anchor after that was a single unique line, using `index` to
disambiguate and `offset` to insert at a structural boundary.

**`storage/` is not always where you think.** When it's relocated outside the webroot,
the old in-webroot directory usually still exists and is empty — so the obvious place to
check for logs and compiled modification output will tell you nothing is compiled and
there are no logs, and both are lies. Read `config.php` before believing either.

**Everything is additive.** New tables are `oc_erp_*` (InnoDB/utf8mb4) alongside
OpenCart's own MyISAM/utf8 tables, with no foreign keys crossing the boundary. Cart and
order hooks went through `oc_event` where OCMOD would have been brittle. Core files stay
patchable and the store still upgrades.

**Written down as it went.** A running dev log records each module, what was verified
live, and — importantly — which earlier entries were later proven wrong. Superseded
decisions were struck through rather than deleted, so the reasoning survives.

---

## What I'd change

The scope grew module by module from client feedback, which kept every release useful
but left the finance and document layers sharing concepts they should have shared
explicitly from the start — that surfaced later as a merge of adjustments and journals.
Given the same brief again I'd model documents generically before building the first
one, and I'd push harder, earlier, for a staging copy of the store; verifying on
production is survivable but it makes every deploy heavier than it needs to be.
