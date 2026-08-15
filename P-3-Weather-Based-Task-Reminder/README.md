# 🌧️ Weather-Based Task Reminder — Auto-Create "Work From Home" on Rainy Days

An n8n automation that checks tomorrow's weather forecast every evening and automatically creates a "Work From Home" event on Google Calendar if rain is expected — so you never get caught off guard by a rainy commute.

Built entirely with **built-in n8n nodes** — Schedule Trigger, HTTP Request, Code, IF, Google Calendar.

---

## 🎥 Demo

[![Watch Demo Video](https://img.shields.io/badge/▶%20Watch%20Demo-Google%20Drive-2ec4a0?style=for-the-badge&logo=googledrive&logoColor=white)](https://drive.google.com/file/d/1zfwDMnsUFsxLKswZJC3Uos2Y3sHo9fN0/view?usp=sharing)

---

## 🎯 The Problem This Solves

Deciding whether to commute into the office or work from home is easy to forget about until the morning of — by which point it's too late to plan around it. This workflow removes that daily decision entirely:

- Every evening, the workflow **checks tomorrow's full-day forecast** automatically — no manual weather-checking required.
- If **any part of tomorrow** is forecast to have rain, thunderstorms, or drizzle, a **"Work From Home" event is created automatically** on Google Calendar for the next day, complete with a reminder the night before.
- If tomorrow is expected to be clear, the workflow **does nothing** — no event, no noise, no clutter on the calendar.
- Runs unattended, once a day, with zero ongoing manual effort.

---

## 🏗️ Architecture & Flow

```
Schedule Trigger  (9:00 PM daily, Asia/Kolkata)
     │
     ▼
HTTP Request - Get Forecast
(OpenWeatherMap 5-day / 3-hour forecast API,
 GET /data/2.5/forecast?q={city}&appid={key}&units=metric)
     │
     ▼
Code - Find Tomorrow Rain
(converts each 3-hour slot's UTC timestamp to IST,
 filters for slots that fall on tomorrow's date,
 flags rainDetected = true if ANY slot shows
 Rain / Thunderstorm / Drizzle)
     │
     ▼
IF - Rain Check  (rainDetected === true)
     │
     ├── TRUE ───▶ Google Calendar - Create WFH Event
     │              (all-day event, tomorrow's date,
     │               title "Work From Home",
     │               description "Rain expected tomorrow")
     │
     └── FALSE ──▶ (nothing connected — flow ends silently)
```

---

## ✨ Key Design Decisions

- **Forecast endpoint, not Current Weather endpoint** — OpenWeatherMap's free-tier "current weather" call only reports *today's* conditions, which is useless for a "tomorrow" check. The 5-day/3-hour forecast endpoint (`/data/2.5/forecast`) is also free-tier and returns enough future slots to reliably find tomorrow's weather.
- **All date comparisons done in IST inside the Code node, not the workflow's server timezone** — OpenWeatherMap returns each forecast slot's timestamp in UTC. Converting to `Asia/Kolkata` before comparing calendar dates (using Luxon's `DateTime`, built into n8n Code nodes) avoids slots near the UTC midnight boundary being misattributed to the wrong day.
- **`rainDetected` uses `.some()`, not requiring the whole day to be rainy** — a single rainy 3-hour slot anywhere in tomorrow is enough to justify working from home; this was confirmed in live testing where only 3 of 7 matched slots (12:00, 15:00, 18:00) showed rain, and that was correctly enough to trigger the event.
- **IF false branch intentionally left unconnected** — per the spec, a clear-weather day should result in no action at all, not an empty placeholder step. Verified live: a Tiruppur forecast returning all "Clouds" slots correctly produced `rainDetected: false` and no calendar event was created.
- **Workflow timezone explicitly set to `Asia/Kolkata`** — n8n instances default their Schedule Trigger to the server's own region (often US-based on cloud hosting), not the user's real location. Left unset, "9:00 PM" would have fired at the wrong real-world time.

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Automation engine | n8n v2.28.7 |
| Scheduling | Schedule Trigger node (6-field cron, seconds-first) |
| Weather data | OpenWeatherMap 5-day/3-hour Forecast API (free tier) via HTTP Request node |
| Rain-detection logic | Code node (JavaScript, Luxon `DateTime`) |
| Branching | IF node |
| Event creation | Google Calendar node (OAuth2, free tier) |

---

## ⚙️ Setup & Installation

### Prerequisites
- A running n8n instance, version 2.28.7 or compatible
- A free OpenWeatherMap account and API key
- A Google account with Calendar access

---

### Step 1 — Get an OpenWeatherMap API key
1. Sign up at [openweathermap.org](https://openweathermap.org/api).
2. Go to the **API keys** tab and copy your default key (or generate a new one).
3. **Note:** newly generated keys can take anywhere from a few minutes up to ~2 hours to activate. Calling the API too soon returns a `401 Invalid API key` error even with a correctly pasted key — this is expected, not a workflow bug.

---

### Step 2 — Import the workflow
1. Open n8n → **+ Create Workflow** → **Import from File**.
2. Select `P3_Weather_Task_Reminder.json`.
3. All 5 nodes should appear pre-arranged on the canvas.

---

### Step 3 — Configure the Get Forecast node
| Field | What to set |
|---|---|
| `q` query parameter | Your city, in `City,CountryCode` format (e.g. `Coimbatore,IN`, `Tiruppur,IN`) |
| `appid` query parameter | Your real OpenWeatherMap API key |
| `units` query parameter | Already set to `metric` — leave as is, or change to `imperial` if preferred |

---

### Step 4 — Attach the Google Calendar credential
1. Open the **Create WFH Event** node.
2. Attach/create a Google Calendar OAuth2 credential.
3. `calendar` defaults to `primary` — change if you want events created on a different calendar.

---

### Step 5 — Confirm the workflow timezone
1. Click **⋯** (top right of canvas) → **Settings**.
2. Confirm **Timezone** is set to `Asia/Kolkata` (already set in the imported JSON, but worth double-checking after import).

---

### Step 6 — Publish the workflow
Toggle **Active** — once active, the Schedule Trigger fires automatically every evening at **9:00 PM IST**, no manual execution needed from then on.

---

## ✅ Verified Test Results

The workflow was tested live for both branches by temporarily swapping the `q` parameter to different real cities, since manual test timing doesn't always align with genuinely rainy weather locally.

### Test 1 — Clear weather (false branch)

City used: `Tiruppur,IN`

**Find Tomorrow Rain output:**
```
rainDetected: false
tomorrowDate: 2026-07-14
matchedSlotCount: 7
slotSummaries:
  2026-07-14 00:00:00 — Clouds
  2026-07-14 03:00:00 — Clouds
  2026-07-14 06:00:00 — Clouds
  2026-07-14 09:00:00 — Clouds
  2026-07-14 12:00:00 — Clouds
  2026-07-14 15:00:00 — Clouds
  2026-07-14 18:00:00 — Clouds
```

**Result:** IF node correctly routed to the false branch. No Google Calendar event was created. Confirms "clear weather → no action" works as intended.

### Test 2 — Rain forecast (true branch)

City used: `Patna,IN` (active monsoon rainfall confirmed via IMD bulletin at time of test)

**Find Tomorrow Rain output:**
```
rainDetected: true
tomorrowDate: 2026-07-14
matchedSlotCount: 7
slotSummaries:
  2026-07-14 00:00:00 — Clouds
  2026-07-14 03:00:00 — Clouds
  2026-07-14 06:00:00 — Clouds
  2026-07-14 09:00:00 — Clouds
  2026-07-14 12:00:00 — Rain
  2026-07-14 15:00:00 — Rain
  2026-07-14 18:00:00 — Rain
```

**Result:** IF node correctly routed to the true branch. Google Calendar event was created:

| Field | Value |
|---|---|
| Title | Work From Home |
| Date | Tuesday, July 14 (all-day event) |
| Description | Rain expected tomorrow |
| Reminder | The day before at 11:30pm |
| Calendar | Prasanth |
| Status | Busy |

This confirms the full chain — HTTP Request → Code → IF → Google Calendar — works correctly end to end for the rain-detected case, with a real event visible on the actual calendar.

---

## 🛠️ Common Issues

| Problem | Fix |
|---|---|
| `Get Forecast` node returns `401 Invalid API key` even though the key is pasted correctly | Newly generated OpenWeatherMap API keys aren't active immediately — activation can take a few minutes up to ~2 hours. Confirmed by testing the same URL directly in a browser and getting the same 401; the fix was simply waiting for the key to activate, not a config change. |
| Nearby cities (e.g. Tiruppur vs. Coimbatore) don't reliably produce different rain results | Geographically close cities often share the same regional forecast pattern. To reliably test the "rain detected" true branch, a city with genuinely different/active weather (e.g. one currently under an IMD heavy-rainfall bulletin) was used instead, rather than relying on the home city's forecast to happen to be rainy on test day. |
| Unsure whether the Schedule Trigger will actually fire without manual intervention | Once the workflow is toggled **Active**, the Schedule Trigger runs entirely on its own — no need to keep the editor open or manually execute it. The 9:00 PM cron fires automatically every day going forward. |

---

## 📁 Project Structure

```
P-3-Weather-Based-Task-Reminder/
├── P3_Weather_Task_Reminder.json
├── README.md
└── demo.mp4
```

---

## 👨‍💻 Author
**Sakthi Prasanth**

Project: Weather-Based Task Reminder — Auto-Create "Work From Home" on Rainy Days

Built with n8n, OpenWeatherMap, and Google Calendar
