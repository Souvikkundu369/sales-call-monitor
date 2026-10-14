# Sales Call Monitor — Missed Calls That Never Got a Call Back

> **A missed call isn't the problem. A missed call nobody returned is.** This system counts the second one, for every salesperson, every day, at zero software cost.

<p>
  <img src="https://img.shields.io/badge/role-Sole%20Builder-orange" alt="Sole Builder">
  <img src="https://img.shields.io/badge/Google_Apps_Script-4285F4?style=flat&logo=google&logoColor=white" alt="Apps Script">
  <img src="https://img.shields.io/badge/Google_Sheets-34A853?style=flat&logo=googlesheets&logoColor=white" alt="Google Sheets">
  <img src="https://img.shields.io/badge/Node.js-339933?style=flat&logo=nodedotjs&logoColor=white" alt="Node.js">
</p>

A call-log monitoring system for a sales team handling customer enquiries and bookings. Each salesperson's phone backs up its call log on a schedule; the system imports it, classifies every call, works out which missed calls were never returned, and produces a daily report, a dashboard, and a per-salesperson briefing document that management uses with each person.

---

## The problem

Management wanted to know which salespeople weren't picking up or calling customers back. The company already had an AI system that scores sales calls from their **recordings** ([call-analysis-crm](https://github.com/Souvikkundu369/call-analysis-crm)), but a recording only exists for a call that was *answered*. A missed call, a declined call, or a call never returned leaves no recording, so that system is blind to exactly the behaviour in question.

That needed a different data source: the phone's own call log.

---

## The metric that matters

The raw "missed calls" count is misleading: a busy salesperson misses calls *because* they're on other calls. What actually loses a customer is a missed call **nobody ever returned**.

So the core metric is **missed calls never called back**, measured per *episode*: a missed call opens an episode, a call back to that number within the window closes it, and a fresh missed call after the call-back opens a new one. The business rule is that salespeople respond around the clock, so there are no shift hours: every missed call counts, at any hour, against a 30-minute call-back clock.

---

## Zero-cost pipeline

```
Salesperson phone
  └─ call-log backup app (full backup, on a schedule)
        └─ FolderSync ──► Google Drive (one folder per phone)
                              │  Apps Script import every 30 min
                              ▼
                         Google Sheet: Calls · Daily · Staff · Files · Alerts · Deleted Calls
                              │
          ┌───────────────────┼─────────────────────────┐
          ▼                   ▼                         ▼
   07:00 daily email    PIN-protected web dashboard   Detailed PDF briefing
   (full previous day)  (hour×person heatmap,         (one section per salesperson,
                         leaderboard, filters)          generated with headless Chrome)
```

No telephony subscription, no per-seat software, no custom app on the phones.

---

## Never trust the format: real phones disagree

The parser was built against the actual backup files from each phone, not against the format documentation, and that turned up behaviour no spec would have warned about:

- **One phone records a *declined* call as an *incoming* call with 0 seconds**, not as "rejected." Counting it at face value would make declining a customer look like answering one.
- **Missed calls carry ring time in the "duration" field.** 25 seconds on a missed call is the phone ringing, not a conversation.
- **Call-type codes vary by phone model.** One phone uses extra codes for blocked numbers; another uses codes that are genuinely ambiguous. Ambiguous ones are classified as *other* and reported separately, never guessed.
- **Two SIMs per phone**, both used for work, so the staff setup takes a list of SIMs rather than assuming one.
- Telemarketing and service numbers are excluded so they don't inflate anyone's missed-call count.

---

## The false accusation that almost shipped

The first version detected **deleted calls** by comparing consecutive backups: a call present in one backup and missing from the next was flagged *DELETED*.

On real data it flagged calls that nobody had deleted. Phones cap their call log at a fixed size (8,000 entries on most of these phones, 6,000 on one), and every backup was sitting exactly at that cap. Each new call pushed the oldest one out, so old entries "disappeared" constantly, and the diff reported them as deletions. In a system whose output is used to brief staff, that would have been a false accusation printed in a report.

The fix redefined "deleted" to match how deletion actually looks:

- A call counts as **deleted** only if it's **recent** (within the last 7 days) **and** missing from **two consecutive** backups. The first miss is only *Missing?*.
- A call older than the oldest entry still in the backup is **rolled off**, which is normal behaviour, not misconduct.
- Earlier false flags were corrected automatically, and the affected daily figures rebuilt.

The same principle runs through the whole system: **a monitoring tool that wrongly accuses people is worse than having no tool.**

---

## Seeing calls the phone never logged

A switched-off phone doesn't log the calls it missed, so they can't be counted from the call log at all. The system measures them indirectly: **"phone silent"** is the longest gap between backups. A phone that goes quiet for hours during the day shows up, even though its call log looks clean. An hourly check also alerts when a phone stops sending backups.

---

## Designed knowing what it *can't* do

Being honest about limits was part of the design:

- A call log on a phone the salesperson controls **can't be made tamper-proof**. The system makes tampering *visible* (recent-deletion detection, silent-phone alerts, the backup sync set to never mirror deletions back to Drive) rather than claiming it's impossible.
- The only fully tamper-proof option is a paid cloud-telephony number, documented as the upgrade path rather than oversold.
- The briefing PDF ends with a page explaining **how every number is worked out**, including the blind spots, so anyone reading it can check the logic rather than just trust it.
- Daily emails ship in **test mode** until recipients are explicitly configured; nothing goes to management by accident.

---

## Staying fast inside Google's limits

Eight phones produce over 100,000 call-log rows, and one phone's first backup alone carried more than a year of history. Two constraints shaped the design:

- **Sheets slows down sharply at that size**, so the system keeps a rolling 120-day window, and pruning runs before each import and each daily report.
- **Apps Script has a daily trigger-runtime quota**, so the import runs every 30 minutes rather than every 15. The figures stay fresh enough for a daily report while staying well inside the quota.

---

## Testing

The import, classification and daily-calculation logic is written as **pure functions**, which lets it run outside Google: **65 offline tests** load the same Apps Script code into Node and run it against the real backup files. The PDF generator reuses those same functions, so the briefing document and the live dashboard can't disagree on a number.

---

## Status

Live since October 2026 across 8 sales phones. The per-salesperson briefing PDF is in use by management; the web dashboard is being rolled out.

---

## Tech stack

`Google Apps Script` · `Google Sheets` · `Google Drive` · `FolderSync` · `Node.js (vm-based testing, PDF generation)` · `Headless Chrome` · `HTML/CSS/SVG`

---

### Author

**Souvik Kundu** — Business Intelligence & Automation Engineer.

📫 [LinkedIn](https://linkedin.com/in/souvik-kundu-bi) · [GitHub](https://github.com/Souvikkundu369)
