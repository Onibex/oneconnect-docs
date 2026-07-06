# 📊 OneConnect — Logs Module (Dashboard)

> **Last updated:** July 2026
> **Frontend version:** v2.9.0+

---

## How to Access

1. Open a SAP link dashboard.
2. Click on the **Go to Logs** section.
3. You will see a table with logs coming from three sources: **Producer**, **Consumer**, and **SAP**.

---

## Available Filters

You can filter the logs using the controls at the top of the screen:

| Filter | How it works | Options | Default |
|--------|-------------|---------|---------|
| **Search** | Type any keyword to search within log messages | *(empty)* | *(empty)* |
| **Last / Range** | Toggle between relative time or specific dates | `Last`, `Range` | `Last` |
| **Time** (Last mode) | Show logs from a relative time window | `Last Minute`, `Last 5 Minutes`, `Last Hour`, `Last 6 Hours`, `Last 12 Hours`, `Last 24 Hours`, `Last 7 Days` | `Last 7 Days` |
| **Start Date** (Range mode) | Pick a start date | Any date | Yesterday |
| **Start Time** (Range mode) | Pick a start time (12-hour AM/PM format) | Any time | Current time |
| **End Date** (Range mode) | Pick an end date | Any date | Today |
| **End Time** (Range mode) | Pick an end time (12-hour AM/PM format) | Any time | Current time |
| **Level** | Filter by severity | `ALL`, `ERROR`, `WARN`, `FATAL` | `ALL` |
| **Source** | Filter by origin | `ALL`, `PRODUCER`, `CONSUMER`, `SAP` | `ALL` |

---

## ⚠️ Important: Time Zone

All dates and times are handled in **UTC** automatically.

This means:

- You select the time on your local clock (for example, Venezuela at UTC-4).
- The system automatically converts it to UTC when sending it to the backend.
- You don't need to do any manual conversion — just use your normal time.

---

## Understanding the Table

| Column | What it shows |
|--------|--------------|
| **Time** | When the log happened (e.g. `Jul 1, 9:14 AM`) |
| **Level** | A colored badge: `ERROR` (red), `WARN` (orange), `FATAL` (dark red) |
| **Source** | Where the log came from: `PRODUCER`, `CONSUMER`, or `SAP` |
| **Logger** | The name of the logger that generated the log |
| **Message** | A short preview of the log message (up to 150 characters) |
| **Actions** | A button 📄 to view the full log message |

---

## Viewing the Full Log Message

Click the 📄 button in the **Actions** column. A pop-up will open showing:

- Complete timestamp with seconds
- Level, Source, and Logger details
- The full message text in a scrollable box

You can also click **"Copy Message"** to copy the entire message to your clipboard.

---

## Exporting to Excel

Click the 📥 download button in the filter bar. This exports **all visible logs** (up to 1000) as an Excel file.

The file will be named: `ONE_CONNECT_LOGS_YYYY-MM-DD.xlsx`

---

## Email Notifications (Add Email button)

A button labeled **"Add Email"** appears next to the page title **only if** it has been enabled. This feature allows you to receive email alerts when `ERROR` or `FATAL` logs occur.

### How to enable it

To make the button visible, the environment variable `NEXT_PUBLIC_ENABLE_ADD_EMAIL_LOGS_BUTTON` must be set to `true`.

This is configured in:

| File | What to set |
|------|------------|
| `.env.development` or `.env.production` | `NEXT_PUBLIC_ENABLE_ADD_EMAIL_LOGS_BUTTON=true` |

By default, the button is hidden (`false`).

### What it does

Once you click **"Add Email"**, you can manage which email addresses receive notifications for the current workspace. To avoid receiving too many emails, the system only sends one notification per error type every 60 minutes.

---

## Play / Pause

- **Play** ▶️ (default): Automatically refreshes the logs every 10 seconds.
- **Pause** ⏸️: Stops the automatic refresh.

---

## Pagination

At the bottom of the table you can:

- Choose how many rows to show: **10**, **15**, or **25**
- Navigate between pages using the arrow buttons ◀ ▶
- See the current page and total pages (e.g. `1 - 5`)