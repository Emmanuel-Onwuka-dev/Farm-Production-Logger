# Farm-Production-Logger
# Farm Stock & Mortality Tracking (n8n)

Replaces paper/memory-based daily production logging with a structured digital
pipeline: form submission, data cleaning, mortality and feed calculations,
Airtable stock reconciliation, and conditional alerting.

---

## The Problem

Chinyere Farms had **zero digital record-keeping** for daily production.

- Egg counts, fish growth, and mortality numbers were either remembered or
  scribbled on paper.
- There was **no history** to look back on.
- There were **no trends** to spot patterns over time.
- There was **no early warning** when something went wrong, like a spike in
  mortality or a feed shortage, until it was already a serious problem.

This workflow replaces that entire process with a structured digital system
that logs, calculates, reconciles, and alerts automatically.

---

## Screenshots

**Main workflow: Production Logger 2.0**
<img width="1861" height="853" alt="Farm Production logger" src="https://github.com/user-attachments/assets/e35cd5a3-d561-45a3-a863-a5107f51723f" />

**Sub-workflow: Stock Reconciliation**

<img width="1861" height="862" alt="Stock reconsiliation sub-workflow" src="https://github.com/user-attachments/assets/2cc3e239-8e6a-4a52-9674-3e845739506e" />

**Sub-workflow: Alert Router**

<img width="1860" height="837" alt="Alert router sub-workflow" src="https://github.com/user-attachments/assets/01fcbdd2-5e9f-45d9-80c0-c514c08ef882" />


**Example output: Airtable record after a run**
<img width="1846" height="448" alt="Stock output 1" src="https://github.com/user-attachments/assets/4dd0a956-185f-4d26-a59f-be6cf59585dc" />

<img width="1848" height="467" alt="stock output 2" src="https://github.com/user-attachments/assets/910a9856-ce82-4b66-903b-83666a6db6af" />


**Example output: Telegram alert**

<img width="1182" height="608" alt="Alert output" src="https://github.com/user-attachments/assets/eb3ae13f-340b-4784-838a-ee8205c6ba25" />


---

## What it does

1. **On form submission** - structured digital entry point replaces
   paper/memory-based logging.
2. **Edit Fields** - cleans and structures raw form output into consistent
   field names (`birdDeaths`, `fishDeaths`, `pondId`, etc). Date is a required
   field with a fallback expression (`{{ $json['Date'] || $now.format('yyyy-MM-dd') }}`)
   as a second layer of protection in case a date is ever missed.
3. **Search records** - Airtable search for yesterday's record, using
   `IS_SAME({Date}, DATEADD(TODAY(), -1, 'days'), 'day')` to retrieve
   yesterday's `ClosingBirdStock` / `ClosingFishStock`, which become today's
   opening totals.
4. **Calculate mortality rates** - computes bird and fish mortality rates and
   decides the alert level.
5. **Calculate expected feeding** - flags if feed variance exceeds 15%.
6. **Create a record** - writes the full record (17 columns total), including
   the new feed-tracking fields. `ClosingBirdStock`/`ClosingFishStock` are
   written as 0 intentionally, since reconciliation hasn't run yet.
7. **Call 'Stock Reconciliation' sub-workflow** - searches Airtable for
   yesterday's record again (independently, since sub-workflows can't reach
   into the parent's nodes), subtracts today's deaths from yesterday's closing
   stock, and updates today's row with the real `ClosingBirdStock`/
   `ClosingFishStock`. This becomes tomorrow's opening stock, closing the
   loop.
8. **Call 'Alert Router' sub-workflow** - checks if `alertStatus` is anything
   other than "Normal." Sends an **Urgent Telegram alert** if something's
   flagged (now includes mortality and feed variance context), otherwise
   sends a **Routine Telegram summary** if everything's fine.

---

## Workflow structure

\```
- Production logger 2.0 (main)
- ├─ On form submission
- ├─ Edit Fields
- ├─ Search records          (yesterday's stock)
- ├─ Calculate mortality rates
- ├─ Calculate expected feeding
- ├─ Create a record
- ├─ Call: Stock Reconciliation sub-workflow
- └─ Call: Alert Router sub-workflow

- Stock Reconciliation sub-workflow
- ├─ When Executed by Another Workflow
- ├─ Search records           (Airtable formula: yesterdaystock - todaydeath)
- ├─ Calculate new stock
- └─ Update record

- Alert Router sub-workflow
- ├─ When Executed by Another Workflow
- ├─ Check for Alerts (IF node)
 -  ├─ true  → Urgent Alert (Telegram)
  -  └─ false → Routine Summary (Telegram)
\```

---

## Why sub-workflows

Airtable searches are re-run independently inside each sub-workflow rather
than passed down from the parent, since **n8n sub-workflows can't reach into
the parent workflow's node outputs**. Each sub-workflow needs to fetch its
own copy of the data it depends on.

## Setup

1. Import `production-logger-2.0.json`, `stock-reconciliation.json`, and
   `alert-router.json` into your n8n instance.
2. Connect your Airtable base (credentials + base/table IDs).
3. Connect your Telegram bot credentials and chat ID(s) for Urgent Alert /
   Routine Summary.
4. Update the two "Call sub-workflow" nodes in the main workflow to point at
   your imported sub-workflow IDs.
5. Point the form trigger at your daily logging form.

## Notes / gotchas

- Date is a required form field *and* has a fallback expression. Don't
  remove the fallback, it's there specifically to prevent silent breaks if a
  date is ever missed.
- `ClosingBirdStock`/`ClosingFishStock` are intentionally written as 0 at
  creation time. They're only correct after the Stock Reconciliation
  sub-workflow runs.
- Feed variance alert threshold is 15%, mortality threshold is 2. Adjust in
  the "Calculate mortality rates" / "Calculate expected feeding" nodes.

## License

MIT. Free to use and adapt for your own farm/inventory tracking setup.
