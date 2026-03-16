# SC Request Presentation Site Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a GitHub Pages presentation site that walks stakeholders through the SC Request redesign proposal so leadership can give buy-in to implement it.

**Architecture:** A single-page application using vanilla HTML/CSS/JS with a sticky sidebar navigation menu. Each section of the spec becomes a presentation section. No build tools — pure static files deployable directly via GitHub Pages from the `docs/site/` directory.

**Tech Stack:** HTML5, CSS3 (custom properties), vanilla JavaScript, GitHub Pages

---

## Chunk 1: Site Shell & Navigation

### Task 1: Create site structure and index.html shell

**Files:**
- Create: `docs/site/index.html`
- Create: `docs/site/styles.css`
- Create: `docs/site/app.js`

- [ ] **Step 1: Create `docs/site/index.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>SC Request Redesign — GoTo</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <div class="layout">
    <!-- Sidebar nav -->
    <nav class="sidebar" id="sidebar">
      <div class="sidebar-header">
        <div class="sidebar-logo">GoTo</div>
        <div class="sidebar-title">SC Request Redesign</div>
      </div>
      <ul class="nav-list" id="nav-list">
        <li><a href="#problem" class="nav-link active">The Problem</a></li>
        <li><a href="#solution" class="nav-link">Proposed Solution</a></li>
        <li><a href="#flow" class="nav-link">Request Flow</a></li>
        <li><a href="#calendar" class="nav-link">Calendar Integration</a></li>
        <li><a href="#data-model" class="nav-link">Data Model</a></li>
        <li><a href="#email" class="nav-link">Email Redesign</a></li>
        <li><a href="#options" class="nav-link">Implementation Options</a></li>
        <li><a href="#next-steps" class="nav-link">Next Steps</a></li>
      </ul>
    </nav>

    <!-- Main content -->
    <main class="content" id="content">
      <section id="problem" class="section">
        <div class="section-inner">
          <span class="section-label">The Problem</span>
          <h1>The SC Request Process Has Friction</h1>
          <p class="lead">Today's process works — but it requires too many manual steps and disconnected tools.</p>
        </div>
      </section>

      <section id="solution" class="section">
        <div class="section-inner">
          <span class="section-label">Proposed Solution</span>
          <h1>A Smarter SC Request in Salesforce</h1>
        </div>
      </section>

      <section id="flow" class="section">
        <div class="section-inner">
          <span class="section-label">Request Flow</span>
          <h1>How It Works</h1>
        </div>
      </section>

      <section id="calendar" class="section">
        <div class="section-inner">
          <span class="section-label">Calendar Integration</span>
          <h1>Real-Time Availability</h1>
        </div>
      </section>

      <section id="data-model" class="section">
        <div class="section-inner">
          <span class="section-label">Data Model</span>
          <h1>What Changes in Salesforce</h1>
        </div>
      </section>

      <section id="email" class="section">
        <div class="section-inner">
          <span class="section-label">Email Redesign</span>
          <h1>Better Notifications</h1>
        </div>
      </section>

      <section id="options" class="section">
        <div class="section-inner">
          <span class="section-label">Implementation Options</span>
          <h1>Two Ways to Build It</h1>
        </div>
      </section>

      <section id="next-steps" class="section">
        <div class="section-inner">
          <span class="section-label">Next Steps</span>
          <h1>What We Need to Move Forward</h1>
        </div>
      </section>
    </main>
  </div>
  <script src="app.js"></script>
</body>
</html>
```

- [ ] **Step 2: Create `docs/site/styles.css`**

```css
/* ── Reset & base ─────────────────────────── */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

:root {
  --sidebar-width: 260px;
  --blue: #0070d2;
  --blue-dark: #005fb2;
  --blue-light: #e8f4fd;
  --text: #1a1a2e;
  --text-muted: #6b7280;
  --border: #e5e7eb;
  --bg: #ffffff;
  --section-bg-alt: #f8fafc;
  --green: #059669;
  --amber: #d97706;
  --red: #dc2626;
  --radius: 10px;
  --shadow: 0 1px 3px rgba(0,0,0,0.08), 0 4px 16px rgba(0,0,0,0.05);
}

html { scroll-behavior: smooth; }
body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; color: var(--text); background: var(--bg); }

/* ── Layout ───────────────────────────────── */
.layout { display: flex; min-height: 100vh; }

/* ── Sidebar ──────────────────────────────── */
.sidebar {
  width: var(--sidebar-width);
  background: var(--text);
  position: fixed;
  top: 0; left: 0; bottom: 0;
  display: flex; flex-direction: column;
  z-index: 100;
  overflow-y: auto;
}
.sidebar-header {
  padding: 28px 20px 20px;
  border-bottom: 1px solid rgba(255,255,255,0.08);
}
.sidebar-logo {
  font-size: 11px; font-weight: 700; letter-spacing: 2px;
  text-transform: uppercase; color: var(--blue); margin-bottom: 8px;
}
.sidebar-title {
  font-size: 14px; font-weight: 600; color: #fff; line-height: 1.4;
}
.nav-list { list-style: none; padding: 16px 0; }
.nav-list li { margin: 2px 0; }
.nav-link {
  display: block; padding: 10px 20px;
  color: rgba(255,255,255,0.6); text-decoration: none;
  font-size: 13px; font-weight: 500; border-radius: 0;
  transition: all 0.15s; border-left: 3px solid transparent;
}
.nav-link:hover { color: #fff; background: rgba(255,255,255,0.06); }
.nav-link.active { color: #fff; border-left-color: var(--blue); background: rgba(0,112,210,0.15); }

/* ── Content ──────────────────────────────── */
.content { margin-left: var(--sidebar-width); flex: 1; }

/* ── Sections ─────────────────────────────── */
.section { min-height: 100vh; display: flex; align-items: center; border-bottom: 1px solid var(--border); }
.section:nth-child(even) { background: var(--section-bg-alt); }
.section-inner { max-width: 860px; padding: 80px 60px; width: 100%; }
.section-label {
  display: inline-block; font-size: 11px; font-weight: 700;
  letter-spacing: 2px; text-transform: uppercase;
  color: var(--blue); margin-bottom: 16px;
}
.section-inner h1 { font-size: 42px; font-weight: 700; line-height: 1.15; margin-bottom: 20px; color: var(--text); }
.lead { font-size: 18px; color: var(--text-muted); line-height: 1.7; max-width: 600px; }

/* ── Cards ────────────────────────────────── */
.cards { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 16px; margin-top: 32px; }
.card {
  background: var(--bg); border: 1px solid var(--border);
  border-radius: var(--radius); padding: 24px;
  box-shadow: var(--shadow);
}
.card-icon { font-size: 28px; margin-bottom: 12px; }
.card h3 { font-size: 15px; font-weight: 600; margin-bottom: 8px; }
.card p { font-size: 13px; color: var(--text-muted); line-height: 1.6; }

/* ── Comparison table ─────────────────────── */
.compare { width: 100%; border-collapse: collapse; margin-top: 32px; font-size: 14px; }
.compare th { background: var(--text); color: #fff; padding: 12px 16px; text-align: left; font-size: 12px; text-transform: uppercase; letter-spacing: 1px; }
.compare td { padding: 12px 16px; border-bottom: 1px solid var(--border); vertical-align: top; }
.compare tr:last-child td { border-bottom: none; }
.compare .yes { color: var(--green); font-weight: 600; }
.compare .no { color: var(--red); font-weight: 600; }
.compare .partial { color: var(--amber); font-weight: 600; }
.compare .highlight { background: rgba(0,112,210,0.04); }

/* ── Step flow ────────────────────────────── */
.steps { margin-top: 32px; display: flex; flex-direction: column; gap: 0; }
.step { display: flex; gap: 20px; align-items: flex-start; padding: 20px 0; border-bottom: 1px solid var(--border); }
.step:last-child { border-bottom: none; }
.step-num {
  width: 36px; height: 36px; border-radius: 50%;
  background: var(--blue); color: #fff;
  display: flex; align-items: center; justify-content: center;
  font-size: 14px; font-weight: 700; flex-shrink: 0;
}
.step-body h3 { font-size: 15px; font-weight: 600; margin-bottom: 4px; }
.step-body p { font-size: 13px; color: var(--text-muted); line-height: 1.6; }
.step-auto .step-num { background: var(--green); }

/* ── Badge ────────────────────────────────── */
.badge {
  display: inline-block; padding: 3px 10px; border-radius: 20px;
  font-size: 11px; font-weight: 600; text-transform: uppercase; letter-spacing: 0.5px;
}
.badge-blue { background: var(--blue-light); color: var(--blue); }
.badge-green { background: #d1fae5; color: var(--green); }
.badge-amber { background: #fef3c7; color: var(--amber); }

/* ── Email mockup ─────────────────────────── */
.email-mockup {
  border: 1px solid var(--border); border-radius: var(--radius);
  overflow: hidden; box-shadow: var(--shadow); margin-top: 32px; max-width: 600px;
}
.email-header { background: var(--blue); padding: 16px 24px; display: flex; align-items: center; gap: 12px; }
.email-header-badge { background: #fff; color: var(--blue); font-size: 11px; font-weight: 700; padding: 3px 10px; border-radius: 4px; }
.email-header-title { color: #fff; font-size: 15px; font-weight: 600; }
.email-body { padding: 24px; background: #fff; color: var(--text); }
.email-meta { font-size: 13px; color: var(--text-muted); margin-bottom: 20px; }
.email-grid {
  display: grid; grid-template-columns: 1fr 1fr; gap: 12px 24px;
  background: #f4f6f9; border-radius: 8px; padding: 16px 20px; margin-bottom: 20px;
}
.email-field-label { font-size: 11px; text-transform: uppercase; color: #888; font-weight: 600; margin-bottom: 3px; }
.email-field-value { font-size: 13px; font-weight: 600; }
.email-field-value.link { color: var(--blue); }
.email-field-value.date { color: var(--red); }
.email-notes { margin-bottom: 20px; }
.email-notes-label { font-size: 11px; text-transform: uppercase; color: #888; font-weight: 600; margin-bottom: 8px; }
.email-notes-body { background: #fffbf0; border-left: 3px solid var(--amber); padding: 12px 16px; border-radius: 0 6px 6px 0; font-size: 13px; line-height: 1.6; color: #333; }
.email-actions { display: flex; gap: 10px; margin-bottom: 20px; flex-wrap: wrap; }
.email-btn { padding: 9px 16px; border-radius: 5px; font-size: 12px; font-weight: 600; text-decoration: none; cursor: pointer; border: none; }
.email-btn-primary { background: var(--blue); color: #fff; }
.email-btn-secondary { background: #fff; color: var(--blue); border: 1.5px solid var(--blue); }
.email-footer { border-top: 1px solid var(--border); padding-top: 14px; font-size: 12px; color: #888; }

/* ── Data table ───────────────────────────── */
.data-table { width: 100%; border-collapse: collapse; font-size: 13px; margin-top: 24px; }
.data-table th { background: #f4f6f9; padding: 10px 14px; text-align: left; font-size: 11px; text-transform: uppercase; letter-spacing: 1px; color: var(--text-muted); border-bottom: 2px solid var(--border); }
.data-table td { padding: 10px 14px; border-bottom: 1px solid var(--border); vertical-align: top; }
.data-table code { background: #f4f6f9; padding: 2px 6px; border-radius: 4px; font-size: 12px; font-family: 'SF Mono', 'Fira Code', monospace; }
.data-table .new { color: var(--blue); font-weight: 600; }
.data-table .modified { color: var(--amber); font-weight: 600; }

/* ── Responsive ───────────────────────────── */
@media (max-width: 768px) {
  .sidebar { transform: translateX(-100%); transition: transform 0.2s; }
  .sidebar.open { transform: translateX(0); }
  .content { margin-left: 0; }
  .section-inner { padding: 48px 24px; }
  .section-inner h1 { font-size: 28px; }
  .email-grid { grid-template-columns: 1fr; }
}
```

- [ ] **Step 3: Create `docs/site/app.js`**

```javascript
// Active nav link tracking via IntersectionObserver
const sections = document.querySelectorAll('.section');
const navLinks = document.querySelectorAll('.nav-link');

const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        const id = entry.target.id;
        navLinks.forEach(link => {
          link.classList.toggle('active', link.getAttribute('href') === `#${id}`);
        });
      }
    });
  },
  { threshold: 0.4 }
);

sections.forEach(section => observer.observe(section));
```

- [ ] **Step 4: Open `docs/site/index.html` in browser to verify shell renders**

Expected: Sidebar on left with all 8 nav items, content area on right, each section takes full viewport height.

- [ ] **Step 5: Commit**

```bash
git add docs/site/
git commit -m "feat: add presentation site shell with sidebar nav"
```

---

## Chunk 2: Section Content — Problem & Solution

### Task 2: Fill in "The Problem" section

**Files:**
- Modify: `docs/site/index.html` — replace `#problem` section inner content

- [ ] **Step 1: Replace the `#problem` section-inner content in `index.html`**

Replace the `#problem` section's `<div class="section-inner">` content with:

```html
<span class="section-label">The Problem</span>
<h1>The SC Request Process Has Friction</h1>
<p class="lead">Today's process works — but it requires too many manual steps and disconnected tools that slow AEs down.</p>

<div class="cards" style="margin-top:40px">
  <div class="card">
    <div class="card-icon">📋</div>
    <h3>Manual SC Selection</h3>
    <p>AEs must pick from a full list of SCs every time. Their assigned SC is tracked in a spreadsheet outside of Salesforce.</p>
  </div>
  <div class="card">
    <div class="card-icon">📅</div>
    <h3>Disconnected Scheduling</h3>
    <p>Scheduling a meeting with the customer and securing the SC's calendar are two separate steps with no coordination.</p>
  </div>
  <div class="card">
    <div class="card-icon">📧</div>
    <h3>Unhelpful Notifications</h3>
    <p>The current email says "See Meeting Invite" instead of the actual time. No direct links to Salesforce records.</p>
  </div>
  <div class="card">
    <div class="card-icon">📊</div>
    <h3>No Reporting History</h3>
    <p>SC requests are only tracked as Events. There's no structured record to report on SC utilization or request patterns.</p>
  </div>
</div>
```

- [ ] **Step 2: Verify in browser — 4 problem cards render correctly**

- [ ] **Step 3: Commit**

```bash
git add docs/site/index.html
git commit -m "feat: add problem section content"
```

### Task 3: Fill in "Proposed Solution" section

**Files:**
- Modify: `docs/site/index.html` — replace `#solution` section inner content

- [ ] **Step 1: Replace the `#solution` section-inner content**

```html
<span class="section-label">Proposed Solution</span>
<h1>A Smarter SC Request — Without Leaving Salesforce</h1>
<p class="lead">The same "Request SC" button AEs already know. A completely redesigned experience behind it.</p>

<div class="cards" style="margin-top:40px">
  <div class="card">
    <div class="card-icon">🎯</div>
    <h3>Auto-Assigned SC</h3>
    <p>The AE's assigned SC is pre-populated automatically. Override available if needed. No more hunting through a full list.</p>
  </div>
  <div class="card">
    <div class="card-icon">📆</div>
    <h3>Live Calendar Availability</h3>
    <p>See the next 5 open slots for both AE and SC in real time. Pick a time while on the phone with the customer.</p>
  </div>
  <div class="card">
    <div class="card-icon">📨</div>
    <h3>Instant Outlook Invite</h3>
    <p>Optionally send a calendar invite to both AE and SC automatically on submit — no separate step required.</p>
  </div>
  <div class="card">
    <div class="card-icon">📈</div>
    <h3>Full Request History</h3>
    <p>Every SC request is stored as a structured record on the Opportunity — reportable, searchable, always accessible.</p>
  </div>
</div>

<div style="margin-top:40px;padding:24px;background:var(--blue-light);border-radius:var(--radius);border-left:4px solid var(--blue)">
  <p style="font-size:13px;font-weight:700;text-transform:uppercase;letter-spacing:1px;color:var(--blue);margin-bottom:8px">Zero Retraining Required</p>
  <p style="font-size:15px;color:var(--text)">The entry point stays exactly the same — "Request SC" in the Opportunity action bar. AEs don't need to learn anything new about where to find it.</p>
</div>
```

- [ ] **Step 2: Verify in browser — 4 solution cards + blue callout render correctly**

- [ ] **Step 3: Commit**

```bash
git add docs/site/index.html
git commit -m "feat: add solution section content"
```

---

## Chunk 3: Section Content — Flow & Calendar

### Task 4: Fill in "Request Flow" section

**Files:**
- Modify: `docs/site/index.html` — replace `#flow` section inner content

- [ ] **Step 1: Replace the `#flow` section-inner content**

```html
<span class="section-label">Request Flow</span>
<h1>How It Works — Step by Step</h1>
<p class="lead">The AE completes the entire process in one modal. Everything else happens automatically.</p>

<div class="steps" style="margin-top:40px">
  <div class="step">
    <div class="step-num">1</div>
    <div class="step-body">
      <h3>AE clicks "Request SC" in the Opportunity action bar</h3>
      <p>Same button, same location. A Lightning modal opens instantly — no page navigation.</p>
    </div>
  </div>
  <div class="step">
    <div class="step-num">2</div>
    <div class="step-body">
      <h3>SC is auto-populated from the AE's assignment</h3>
      <p>The system looks up the AE's assigned SC automatically. The AE can override and choose any SC from a dropdown if needed.</p>
    </div>
  </div>
  <div class="step">
    <div class="step-num">3</div>
    <div class="step-body">
      <h3>Live calendar availability loads</h3>
      <p>The next 5 open time slots for both the AE and SC appear instantly. The AE can also browse a specific date. Duration defaults to 1 hour.</p>
    </div>
  </div>
  <div class="step">
    <div class="step-num">4</div>
    <div class="step-body">
      <h3>AE selects request type, product line, and fills in details</h3>
      <p>Dropdowns for Request Type (14 options) and Product Line (17 options). Freeform notes field for context, current state, and demo requirements.</p>
    </div>
  </div>
  <div class="step">
    <div class="step-num">5</div>
    <div class="step-body">
      <h3>AE reviews and submits</h3>
      <p>Optional checkbox: "Send Outlook calendar invite to SC automatically" — checked by default. One click to submit.</p>
    </div>
  </div>
  <div class="step step-auto">
    <div class="step-num">✓</div>
    <div class="step-body">
      <h3>Everything else happens automatically</h3>
      <p>
        SC Request record created &nbsp;·&nbsp;
        Event logged on Opportunity &nbsp;·&nbsp;
        SC added to Opportunity Team &nbsp;·&nbsp;
        Opportunity fields updated &nbsp;·&nbsp;
        Email sent to AE, SC, and both managers &nbsp;·&nbsp;
        Outlook invite sent (if checked)
      </p>
    </div>
  </div>
</div>
```

- [ ] **Step 2: Verify in browser — 5 numbered steps + green auto step render correctly**

- [ ] **Step 3: Commit**

```bash
git add docs/site/index.html
git commit -m "feat: add request flow section content"
```

### Task 5: Fill in "Calendar Integration" section

**Files:**
- Modify: `docs/site/index.html` — replace `#calendar` section inner content

- [ ] **Step 1: Replace the `#calendar` section-inner content**

```html
<span class="section-label">Calendar Integration</span>
<h1>Schedule While You're on the Phone</h1>
<p class="lead">The AE sees real availability for both themselves and the SC — no back-and-forth, no separate scheduling step.</p>

<div class="cards" style="margin-top:40px">
  <div class="card">
    <div class="card-icon">⚡</div>
    <h3>Next 5 Open Slots</h3>
    <p>The system shows the next 5 times that work for both AE and SC automatically — no date hunting required.</p>
  </div>
  <div class="card">
    <div class="card-icon">🗓️</div>
    <h3>Date Picker</h3>
    <p>If the AE has a specific date in mind (from the customer conversation), they can select it and see all available times that day.</p>
  </div>
  <div class="card">
    <div class="card-icon">⏱️</div>
    <h3>Duration Options</h3>
    <p>30 min, 1 hour, or 90 min. Defaults to 1 hour. Slots update instantly when duration changes.</p>
  </div>
  <div class="card">
    <div class="card-icon">📬</div>
    <h3>Optional Outlook Invite</h3>
    <p>After picking a slot, the AE can choose to auto-send an Outlook calendar invite to the SC. Opt-out if already scheduled.</p>
  </div>
</div>

<div style="margin-top:32px;padding:20px 24px;background:#fff;border:1px solid var(--border);border-radius:var(--radius);box-shadow:var(--shadow)">
  <p style="font-size:12px;font-weight:700;text-transform:uppercase;letter-spacing:1px;color:var(--text-muted);margin-bottom:12px">If calendar is unavailable</p>
  <p style="font-size:14px;color:var(--text);line-height:1.7">The form never blocks submission. If the Microsoft 365 API is unavailable or no slots are found, the AE can always enter a date and time manually and proceed.</p>
</div>
```

- [ ] **Step 2: Verify in browser — 4 calendar cards + fallback note render correctly**

- [ ] **Step 3: Commit**

```bash
git add docs/site/index.html
git commit -m "feat: add calendar integration section content"
```

---

## Chunk 4: Section Content — Data Model, Email & Options

### Task 6: Fill in "Data Model" section

**Files:**
- Modify: `docs/site/index.html` — replace `#data-model` section inner content

- [ ] **Step 1: Replace the `#data-model` section-inner content**

```html
<span class="section-label">Data Model</span>
<h1>What Changes in Salesforce</h1>
<p class="lead">Two new custom objects. A few new fields on the Opportunity. Everything else is standard Salesforce.</p>

<table class="data-table" style="margin-top:40px">
  <thead>
    <tr>
      <th>Object</th>
      <th>Status</th>
      <th>Purpose</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>SC_Request__c</code></td>
      <td><span class="new">New</span></td>
      <td>Stores every SC request for reporting, history display on Opportunity, and audit trail</td>
    </tr>
    <tr>
      <td><code>SC_Pairing__c</code></td>
      <td><span class="new">New</span></td>
      <td>AE→SC assignment data — replaces the external spreadsheet. Maintained by SC managers directly in Salesforce.</td>
    </tr>
    <tr>
      <td><code>Opportunity</code></td>
      <td><span class="modified">Modified</span></td>
      <td>New field: <code>SC_Request_Notes__c</code>. Existing fields auto-populated on submit: Primary Solution Consultant, SC Date Requested, SC On Team ✓, Demo ✓</td>
    </tr>
    <tr>
      <td><code>Event</code></td>
      <td>Standard</td>
      <td>Created on submit, linked to Opportunity. Title format preserved: "Request for [Type] for [Account] w/ a Solutions Consultant"</td>
    </tr>
    <tr>
      <td><code>OpportunityTeamMember</code></td>
      <td>Standard</td>
      <td>SC added to Opportunity Team with role "Solution Consultant" and Read/Write access</td>
    </tr>
  </tbody>
</table>

<div style="margin-top:32px;padding:20px 24px;background:#f0fdf4;border-radius:var(--radius);border-left:4px solid var(--green)">
  <p style="font-size:13px;font-weight:700;color:var(--green);margin-bottom:6px">SC Pairing — No More Spreadsheet</p>
  <p style="font-size:14px;color:var(--text);line-height:1.7">SC managers will maintain AE→SC assignments directly in Salesforce via a simple list view. One SC can be assigned to many AEs. Changes take effect immediately — no import required after the initial setup.</p>
</div>
```

- [ ] **Step 2: Verify in browser — data table + green callout render correctly**

- [ ] **Step 3: Commit**

```bash
git add docs/site/index.html
git commit -m "feat: add data model section content"
```

### Task 7: Fill in "Email Redesign" section

**Files:**
- Modify: `docs/site/index.html` — replace `#email` section inner content

- [ ] **Step 1: Replace the `#email` section-inner content**

```html
<span class="section-label">Email Redesign</span>
<h1>Notifications That Actually Help</h1>
<p class="lead">The current email is a plain text list. The new one is structured, scannable, and has direct links to everything.</p>

<div class="email-mockup">
  <div class="email-header">
    <div class="email-header-badge">SC REQUEST</div>
    <div class="email-header-title">A New SC Request Has Been Submitted</div>
  </div>
  <div class="email-body">
    <p class="email-meta">Submitted by <strong>Juliana Gigas</strong> on <strong>March 16, 2026 at 10:34 AM</strong></p>
    <div class="email-grid">
      <div>
        <div class="email-field-label">Opportunity</div>
        <div class="email-field-value link">POC - AIOPT →</div>
      </div>
      <div>
        <div class="email-field-label">Primary Contact</div>
        <div class="email-field-value">Robert W. Sorensen</div>
      </div>
      <div>
        <div class="email-field-label">Request Type</div>
        <div class="email-field-value">Trial Engagement/POC</div>
      </div>
      <div>
        <div class="email-field-label">Product Line</div>
        <div class="email-field-value">GoToConnect</div>
      </div>
      <div>
        <div class="email-field-label">Assigned SC</div>
        <div class="email-field-value">Joe Varvel</div>
      </div>
      <div>
        <div class="email-field-label">Meeting Date & Time</div>
        <div class="email-field-value date">March 20, 2026 at 8:00 AM</div>
      </div>
    </div>
    <div class="email-notes">
      <div class="email-notes-label">What does the customer want to achieve / solve?</div>
      <div class="email-notes-body">Customer is evaluating GoToConnect for their 250-seat contact center. Looking for a demo focused on the supervisor dashboard and real-time reporting capabilities.</div>
    </div>
    <div class="email-actions">
      <button class="email-btn email-btn-primary">View Opportunity →</button>
      <button class="email-btn email-btn-secondary">View SC Request →</button>
      <button class="email-btn email-btn-secondary">View Event →</button>
    </div>
    <div class="email-footer">
      Sent to: <strong>Joe Varvel</strong> (SC) · <strong>Juliana Gigas</strong> (AE) · <strong>Taylor Brinton</strong> (AE Manager) · <strong>Tony Purcell</strong> (SC Manager)
    </div>
  </div>
</div>

<div class="cards" style="margin-top:32px">
  <div class="card">
    <div class="card-icon">🕐</div>
    <h3>Actual Meeting Time</h3>
    <p>No more "See Meeting Invite." The exact date and time is in the email.</p>
  </div>
  <div class="card">
    <div class="card-icon">🔗</div>
    <h3>Three Direct Links</h3>
    <p>One click to the Opportunity, SC Request record, or Event — right from the email.</p>
  </div>
  <div class="card">
    <div class="card-icon">👥</div>
    <h3>Recipient Transparency</h3>
    <p>Everyone can see who else received the notification — no guessing.</p>
  </div>
</div>
```

- [ ] **Step 2: Verify in browser — email mockup and 3 cards render correctly**

- [ ] **Step 3: Commit**

```bash
git add docs/site/index.html
git commit -m "feat: add email redesign section with mockup"
```

### Task 8: Fill in "Implementation Options" section

**Files:**
- Modify: `docs/site/index.html` — replace `#options` section inner content

- [ ] **Step 1: Replace the `#options` section-inner content**

```html
<span class="section-label">Implementation Options</span>
<h1>Two Ways to Build It</h1>
<p class="lead">Both options deliver the same experience and write the same data to Salesforce. The difference is the platform.</p>

<table class="compare" style="margin-top:40px">
  <thead>
    <tr>
      <th>Criterion</th>
      <th>⭐ Option A — Salesforce Native</th>
      <th>Option C — Slack Workflow</th>
    </tr>
  </thead>
  <tbody>
    <tr class="highlight">
      <td><strong>Where AE works</strong></td>
      <td class="yes">Inside Salesforce — no context switch</td>
      <td class="partial">Slack — separate from SFDC</td>
    </tr>
    <tr>
      <td><strong>Opportunity context</strong></td>
      <td class="yes">Automatic — already on the record</td>
      <td class="partial">AE must search for Opportunity</td>
    </tr>
    <tr class="highlight">
      <td><strong>Build complexity</strong></td>
      <td class="partial">Higher — LWC + Apex + Connected App</td>
      <td class="yes">Medium — Node.js + Slack Bolt SDK</td>
    </tr>
    <tr>
      <td><strong>Iteration speed</strong></td>
      <td class="no">Slow — Salesforce deployment process</td>
      <td class="yes">Fast — redeploy backend service</td>
    </tr>
    <tr class="highlight">
      <td><strong>IT/Security compliance</strong></td>
      <td class="yes">Native SFDC — already approved</td>
      <td class="partial">New Slack app — needs approval</td>
    </tr>
    <tr>
      <td><strong>Writes to Salesforce</strong></td>
      <td class="yes">Yes — natively</td>
      <td class="yes">Yes — via REST API</td>
    </tr>
    <tr class="highlight">
      <td><strong>Long-term maintenance</strong></td>
      <td class="yes">One system to maintain</td>
      <td class="partial">SFDC + Slack app + backend service</td>
    </tr>
  </tbody>
</table>

<div style="margin-top:32px;padding:24px;background:var(--blue-light);border-radius:var(--radius);border-left:4px solid var(--blue)">
  <p style="font-size:13px;font-weight:700;text-transform:uppercase;letter-spacing:1px;color:var(--blue);margin-bottom:8px">⭐ Recommendation: Option A — Salesforce Native</p>
  <p style="font-size:15px;color:var(--text);line-height:1.7">AEs already live in Salesforce. Keeping everything native means no context-switching, no extra systems to maintain, and no new IT approvals. The higher build complexity is a one-time cost — the long-term simplicity is worth it.</p>
</div>
```

- [ ] **Step 2: Verify in browser — comparison table and recommendation callout render correctly**

- [ ] **Step 3: Commit**

```bash
git add docs/site/index.html
git commit -m "feat: add implementation options comparison section"
```

---

## Chunk 5: Next Steps & GitHub Pages Setup

### Task 9: Fill in "Next Steps" section

**Files:**
- Modify: `docs/site/index.html` — replace `#next-steps` section inner content

- [ ] **Step 1: Replace the `#next-steps` section-inner content**

```html
<span class="section-label">Next Steps</span>
<h1>What We Need to Move Forward</h1>
<p class="lead">Most decisions are already made. A few confirmations will unlock the full implementation plan.</p>

<div class="steps" style="margin-top:40px">
  <div class="step">
    <div class="step-num">1</div>
    <div class="step-body">
      <h3>Stakeholder Approval <span class="badge badge-blue">In Progress</span></h3>
      <p>Get buy-in from SC leadership and Sales leadership to proceed with Option A (Salesforce Native).</p>
    </div>
  </div>
  <div class="step">
    <div class="step-num">2</div>
    <div class="step-body">
      <h3>M365 OAuth Confirmation <span class="badge badge-amber">Pending IT</span></h3>
      <p>Confirm with IT that the existing SSO / App Registration can be extended to cover <code>Calendars.Read</code> and <code>Calendars.ReadWrite</code> scopes for the Salesforce Connected App.</p>
    </div>
  </div>
  <div class="step">
    <div class="step-num">3</div>
    <div class="step-body">
      <h3>SC Pairing Spreadsheet Export <span class="badge badge-amber">Pending SC Managers</span></h3>
      <p>SC managers export the current AE→SC assignment spreadsheet for one-time import into the new <code>SC_Pairing__c</code> object in Salesforce.</p>
    </div>
  </div>
  <div class="step">
    <div class="step-num">4</div>
    <div class="step-body">
      <h3>Development Kickoff <span class="badge badge-green">Ready</span></h3>
      <p>Full implementation plan is written and ready to execute. Estimated effort includes: 2 custom objects, 1 LWC modal component, 1 Apex controller, 1 Connected App setup, email flow, and Opportunity layout changes.</p>
    </div>
  </div>
</div>

<div style="margin-top:48px;padding:32px;background:var(--text);border-radius:var(--radius);text-align:center">
  <p style="color:rgba(255,255,255,0.6);font-size:13px;margin-bottom:8px">Questions or feedback?</p>
  <p style="color:#fff;font-size:18px;font-weight:600">Let's make the SC request process work the way it should.</p>
</div>
```

- [ ] **Step 2: Verify in browser — all sections complete, nav links all working**

- [ ] **Step 3: Commit**

```bash
git add docs/site/index.html
git commit -m "feat: add next steps section — site content complete"
```

### Task 10: Configure GitHub Pages and push to remote

**Files:**
- Create: `.github/workflows/pages.yml` (optional — GitHub Pages can also be configured directly from repo settings)

- [ ] **Step 1: Create a GitHub repository**

```bash
gh repo create sc-request --public --description "SC Request redesign proposal and implementation plan"
```

- [ ] **Step 2: Set remote and push**

```bash
git remote add origin https://github.com/<YOUR_ORG_OR_USER>/sc-request.git
git push -u origin main
```

- [ ] **Step 3: Enable GitHub Pages in repo settings**

In the GitHub repo settings → Pages → Source: Deploy from branch → Branch: `main`, Folder: `/docs/site`

- [ ] **Step 4: Verify site is live**

Wait ~60 seconds, then open: `https://<YOUR_ORG_OR_USER>.github.io/sc-request/`

Expected: Full presentation site loads with all 8 sections and sidebar navigation working.

- [ ] **Step 5: Commit final state**

```bash
git add .
git commit -m "feat: complete presentation site — ready for stakeholder review"
git push
```
