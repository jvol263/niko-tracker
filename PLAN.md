# Implementation Plan: Stool Color, Doctor Report, Feed Reminder & Baby Profile

## Overview
Four changes to index.html (single file app). Estimated ~400 lines of additions.

---

## 1. Baby Profile (name, birthdate) — stored in localStorage + Firebase

**What:** A small settings section at the bottom of the app (above Export CSV) with baby name and birthdate fields. Saved to localStorage for instant access and synced to Firebase so both parents share it.

**Data:**
```js
{ babyName: 'Niko', babyBirthdate: '2025-01-15' }
// Stored at DB + '/profile.json'
// Also in localStorage key 'niko_profile'
```

**Where it flows:**
- App title: "👶 Niko Tracker" → "👶 {name} Tracker" / "👶 {name}トラッカー"
- HTML `<title>` tag updated dynamically
- Fun toasts: Replace hardcoded "Niko" with baby name (4 toast messages reference Niko)
- Doctor report header: Shows name + age calculated from birthdate
- Feed reminder: Could say "{name} hasn't eaten in..."

**UI:** Two inline fields in a "Baby Profile" card below the date navigator or in the data-actions area:
```
👶 Baby Profile
[Name: Niko          ] [Birthday: 2025-01-15]
```
- Auto-saves on blur/change (no save button needed)
- Defaults to "Niko" and empty birthdate if not set

**i18n keys:** `babyProfile`, `babyName`, `birthday`, `trackerSuffix` (for "{name} Tracker" / "{name}トラッカー")

---

## 2. Stool Color Picker + Red-Flag Alert

**What:** Add a color selector to the poo form and display color on timeline entries. Red/white/black trigger a warning.

**HTML changes (poo form):**
Add a new form-row between the pee selector and the date field:
```html
<div class="form-group">
  <label id="labelPooColor">Color</label>
  <div class="color-picker" id="pooColorPicker">
    <button type="button" class="color-opt" data-color="yellow" style="background:#F0D060" title="Yellow">
    <button type="button" class="color-opt" data-color="green" style="background:#7BAF5E" title="Green">
    <button type="button" class="color-opt" data-color="brown" style="background:#8B6914" title="Brown">
    <button type="button" class="color-opt selected" data-color="brown"> <!-- default -->
    <button type="button" class="color-opt" data-color="red" style="background:#D9534F" title="Red">
    <button type="button" class="color-opt" data-color="white" style="background:#F5F5F5;border:1px solid #ccc" title="White">
    <button type="button" class="color-opt" data-color="black" style="background:#333" title="Black">
  </div>
  <input type="hidden" id="pooColor" value="brown">
</div>
```

**CSS:** `.color-picker` flex row, `.color-opt` 36px circles, `.color-opt.selected` gets a ring/checkmark.

**JS changes:**
- Color picker click handler: sets hidden input value, toggles `.selected` class
- `saveEntry()` poo branch: capture `pooColor` value, include in entry data as `color`
- `editEntry()` poo branch: set color picker selection from `entry.color`
- `renderEntries()` poo branch: show small color dot next to poo title
- **Red-flag alert:** After saving a poo entry with color red/white/black, show a persistent warning modal (not just a toast):
  - White: "🚨 White/chalky stool may indicate a serious condition. Please contact your pediatrician immediately."
  - Red: "⚠️ Red stool may indicate blood. Please contact your pediatrician."
  - Black: "⚠️ Black stool after the first few days may be a concern. Please contact your pediatrician."

**i18n keys:** `pooColor`, `colorYellow`, `colorGreen`, `colorBrown`, `colorRed`, `colorWhite`, `colorBlack`, `alertWhiteStool`, `alertRedStool`, `alertBlackStool`, `medicalAlert`, `dismiss`

---

## 3. Feeding Time-Since Alert (configurable reminder banner)

**What:** A warm-colored banner (like the sleep banner but orange/feed-colored) that appears when it's been too long since the last feed. Tapping it opens the feed modal.

**Settings UI:** A small dropdown in the data-actions / settings area:
```
🍼 Feed Reminder: [Off ▼]  (options: Off / 2.5h / 3h / 3.5h / 4h / 4.5h)
```
Default: Off (so it doesn't surprise existing users). Saved to `localStorage` key `niko_feed_reminder`.

**Banner HTML (after sleep banner):**
```html
<div class="feed-reminder-banner" id="feedReminderBanner" onclick="openModal('feed')">
  🍼 It's been <span id="feedReminderTime">3h 45m</span> since last feed
</div>
```

**CSS:** Same pattern as `.sleep-active-banner` but with `background: var(--feed)` (orange).

**JS logic:**
- `checkFeedReminder()` function:
  - Read reminder threshold from localStorage
  - If 'off' or not set, hide banner and return
  - Find most recent feed entry timestamp (reuse logic from renderTimeSinceLast)
  - If diff > threshold, show banner with time since last feed
  - If diff <= threshold, hide banner
- Called from: `startTimeSinceRefresh()` interval (every 60s), after `syncFromCloud()`, after `saveEntry()` for feed type
- Banner hides automatically when a new feed is logged

**i18n keys:** `feedReminder`, `feedReminderOff`, `feedReminderBanner` ("It's been {time} since last feed" / "{time}ミルクが経ちました"), `reminderLabel`

---

## 4. Doctor Visit Report (on-screen overlay)

**What:** A new overlay (like Trends) showing a comprehensive report for pediatrician visits.

**Access:** New button in the bottom nav or data-actions area:
```html
<button class="nav-btn" onclick="openDoctorReport()">
  🩺 <span id="navLabelReport">Report</span>
</button>
```
Add as a third button in the bottom nav (Summaries | Trends | Report).

**Report Period:** Date picker at top defaulting to "last 30 days" with a preset for "last 2 weeks" and "last 3 months". Simple: just a "since" date picker.

**HTML structure:**
```html
<div class="report-overlay" id="reportOverlay" onclick="closeReportBg(event)">
  <div class="report-card">
    <div class="modal-header">
      <h2 id="reportTitle">🩺 Doctor Visit Report</h2>
      <button class="modal-close" onclick="closeReport()">×</button>
    </div>
    <div class="report-period">
      <label>Since:</label>
      <input type="date" id="reportSince" onchange="renderDoctorReport()">
    </div>
    <div id="reportBody"></div>
    <button class="data-btn" onclick="copyReport()">📋 Copy to Clipboard</button>
  </div>
</div>
```

**Report sections (rendered into reportBody):**

### Section A: Baby Info
```
👶 Niko — 13 months old
Report period: Jan 21, 2026 – Feb 21, 2026
```
Age calculated from birthdate. If no birthdate set, just show name.

### Section B: Feeding Summary
```
🍼 Feeding Summary
  Avg daily volume:     680 ml
  Avg feeds per day:    6.2
  Avg time between:     3h 15m
  Primary type:         Formula (85%)
  Volume trend:         ↑ Increasing
```
- Calculate from all feed entries in period
- Trend: compare first half vs second half of period

### Section C: Sleep Summary
```
💤 Sleep Summary
  Avg daily sleep:      14h 20m
  Avg longest stretch:  5h 45m
  Day / Night split:    35% / 65%
  Sleep trend:          → Stable
```

### Section D: Diaper Summary
```
💩 Diaper Summary
  Avg poos per day:     3.2
  Stool colors:         Brown (72%), Yellow (22%), Green (6%)
  ⚠️ Red stool recorded on Feb 3
```
- Flag any concerning colors with dates

### Section E: Notes & Questions for Doctor
```
📝 Topics to Discuss
  • Feb 18: Seemed fussy after evening feed, spit up more than usual
  • Feb 14: Noticed dry patch on left cheek
  • Feb 10: First time rolling back to front consistently
  • Feb 3: Red stool — one occurrence, resolved next day
```
- All notes from the period, reverse chronological
- Auto-include any red-flag stool color events even if no note was written

### Section F: Alerts & Patterns (auto-generated)
```
⚡ Patterns & Alerts
  • Feed volume increased 12% over this period
  • Longest feeding gap: 5h 20m on Feb 8
  • 2 days with fewer than 4 feeds
  • Red stool recorded once (Feb 3)
```
- Only show if there are noteworthy items; skip section if nothing unusual

**Copy to clipboard:** Formats the report as plain text and copies via `navigator.clipboard.writeText()`.

**CSS:** Reuse `.trends-overlay` / `.trends-card` pattern. Add `.report-section` for each block, `.report-section-title` for headers, `.report-row` for label/value pairs, `.report-alert` for warning items.

**i18n keys:** `doctorReport`, `reportPeriod`, `since`, `feedingSummary`, `avgDailyVolume`, `avgFeedsPerDay`, `avgTimeBetween`, `primaryType`, `volumeTrend`, `sleepSummary`, `avgDailySleep`, `avgLongestStretch`, `sleepTrend`, `diaperSummary`, `avgPoosPerDay`, `stoolColors`, `topicsToDiscuss`, `patternsAlerts`, `copyToClipboard`, `copied`, `increasing`, `decreasing`, `stable`, `navReport`, `reportSinceLabel`, plus all color names

---

## Implementation Order

1. **Baby Profile** — foundation (name/birthdate used by report and toasts)
2. **Stool Color Picker** — small, self-contained form change
3. **Feed Reminder Banner** — small, reuses sleep banner pattern
4. **Doctor Visit Report** — largest feature, depends on #1 and #2

## Files Modified
- `index.html` only (single-file app)
- Push to GitHub + sync OneDrive backup after all features complete
