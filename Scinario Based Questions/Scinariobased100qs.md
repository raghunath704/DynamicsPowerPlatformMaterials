# Top 100 Situational Interview Questions & Answers
## Dynamics 365 CRM + Power Platform (Simple English)

**How to use this guide:** Each question is a real-life "what would you do" situation. The answer gives you the thinking, the steps, and the words to say. Use the STAR method when you speak: **S**ituation, **T**ask, **A**ction, **R**esult.

## Contents
| # | Section | Questions |
| --- | --- | --- |
| 1 | Dataverse & Data Modelling | Q1–Q14 |
| 2 | Model-Driven Apps, Forms & Views | Q15–Q26 |
| 3 | Canvas Apps & Power Apps | Q27–Q38 |
| 4 | Power Automate | Q39–Q52 |
| 5 | Security, Privacy & Compliance | Q53–Q62 |
| 6 | ALM, Solutions & DevOps | Q63–Q72 |
| 7 | Integration & APIs | Q73–Q80 |
| 8 | Plugins & Custom Code | Q81–Q86 |
| 9 | Power Pages / Portals | Q87–Q90 |
| 10 | Reporting & Power BI | Q91–Q93 |
| 11 | Licensing, Governance & Adoption | Q94–Q96 |
| 12 | Project, Stakeholder & Behavioural | Q97–Q100 |
| — | Quick Revision Sheet | End |

---

# Section 1 — Dataverse & Data Modelling (Q1–Q14)

### Q1. The client says "We need 60 custom fields on the Account table." What do you do?
**Answer:**
I would not just add them. First I ask *why* each field is needed and who uses it.

My steps:
1. Run a workshop and list every field with its purpose, who fills it, and who reads it.
2. Check if standard out-of-the-box fields already cover the need (many do — for example `Account Number`, `Industry`, `Revenue`).
3. Group the remaining fields. If a group of fields belongs to a repeating "thing" (like multiple bank accounts or multiple certifications), that is a **separate related table**, not extra columns.
4. If fields are only used by one team or one process, consider a **separate table with a 1:1 or 1:N relationship**, or use form tabs/sections plus business rules to hide them.
5. Warn them about the impact: wide tables slow down forms, views, and Advanced Find, and they make integrations harder.

**Result I would state:** "In my last project we cut 60 requested fields down to 22 real fields plus 2 child tables. Form load time improved and users stopped complaining about clutter."

---

### Q2. Users complain a view is very slow and sometimes times out. How do you fix it?
**Answer:**
Slow views almost always come from bad filtering or too many columns.

Steps:
1. Check how many rows the table has. Over a few hundred thousand rows, unfiltered views will crawl.
2. Look at the view's filter. Filters on **non-indexed** columns, `contains` (leading wildcard) searches, and `OR` conditions across many tables are the usual culprits.
3. Reduce the columns shown — every lookup column costs an extra join.
4. Remove or reduce **linked entity** filters. More than 5–6 joins in FetchXML is a red flag.
5. Add filters that use indexed fields: primary key, owner, status, created on, and any field marked searchable.
6. Ask Microsoft support (or use the Power Platform admin center) to check if a **custom index** is needed on a heavily filtered column.
7. If the view is really a report, move it to Power BI instead.

**Result:** "I reduced one view from 25 columns and 4 joins to 9 columns and 1 join. Load time went from 40 seconds to under 3 seconds."

---

### Q3. The business wants to store 10 million rows of transaction data in Dataverse. Is that a good idea?
**Answer:**
I would question it, because Dataverse is a business application database, not a data warehouse.

What I would ask:
- Do users need to **act** on each row (open it, edit it, run a workflow)? If yes, Dataverse is fine.
- Do they only need to **read and report** on it? Then Power BI on top of Azure SQL, Fabric, or Dataverse long-term retention is better and much cheaper.

My recommendation:
1. Keep "hot" data (last 12–24 months) in Dataverse.
2. Move older data out using **Dataverse long-term retention** or Azure Synapse Link / Fabric Link.
3. Use **elastic tables** in Dataverse if the data is high volume, simple, and does not need complex relationships or rollups.
4. Always talk about storage cost — Dataverse database storage is charged per GB and is expensive compared to Azure storage.

---

### Q4. A customer wants a field that shows the total value of all open opportunities on an account. How do you build it?
**Answer:**
There are four ways, and I pick based on how fresh the number needs to be.

1. **Rollup column** — easiest, no code. But it refreshes on a schedule (about every hour) and has limits: it only works one level down, cannot cross a many-to-many relationship, and has a max of 100 rollup fields per entity by default.
2. **Calculated column** — instant, but it cannot aggregate child rows, only do maths on the same row or a parent lookup.
3. **Real-time plugin** — if the number must be exact the second an opportunity is saved. More code, more maintenance, and a risk of slowing down saves.
4. **Power Automate flow** — good middle ground, but it is asynchronous and can fail silently, so add error handling.

**My default answer:** Start with a rollup column. Only move to a plugin if the business proves they need real-time accuracy.

---

### Q5. You need a relationship where one Contact can belong to many Accounts and one Account has many Contacts. How?
**Answer:**
That is a **many-to-many (N:N)** relationship. I have two choices:

**Option A — Native N:N relationship.** Quick to set up. Dataverse creates a hidden intersect table. The problem: you cannot add any extra fields to that link, and you cannot report on it easily.

**Option B — Manual N:N using a custom "junction" table** (for example `Account Contact Role`). This table has a lookup to Account, a lookup to Contact, plus extra fields like Role, Start Date, End Date, Is Primary.

**What I recommend:** Almost always Option B. In real projects the business eventually asks "when did this relationship start?" or "what is their role?" — and native N:N cannot answer that. The junction table also gives you security, auditing, and reporting.

---

### Q6. Someone deleted 5,000 records by mistake. What now?
**Answer:**
First, stay calm and stop the bleeding.

Steps:
1. **Stop the source** — disable the flow, bulk delete job, integration, or user account that caused it.
2. Check **Auditing**. If auditing was on for that table, the audit log tells you what was deleted, by whom, and when — but audit does **not** restore data.
3. Check if the records are in the **recycle bin** (Dataverse has restore for recently deleted records if the feature is enabled in the admin center).
4. If not, request a **restore to a point in time**. Dataverse keeps backups: system backups for 7 days (up to 28 for some plans) and manual backups for longer. Restore goes to a **new environment**, not over the top of production, so we then copy the missing records back with a data migration tool (Data Migration Utility, XrmToolBox, or a dataflow).
5. Re-import from the source system if the data came from an integration.

**Prevention I would put in place:** remove Bulk Delete privileges from normal users, add a confirmation step in flows, turn on auditing everywhere, and use "deactivate" instead of "delete" as the standard business practice.

---

### Q7. The client insists on using Option Sets (Choices) for a list of 500 products. Good or bad?
**Answer:**
Bad. I would push back politely and explain why.

Problems with a 500-value choice field:
- Only an **admin** can add a new value — the business cannot manage it themselves.
- Every change is a **solution change**, so it needs a deployment through Dev → Test → Prod.
- You cannot store extra information about each item (price, category, active/inactive).
- The dropdown becomes unusable in the UI.
- Reporting on the label instead of a real record is painful.

**My recommendation:** Create a **Product table** (or use the out-of-the-box Product Catalog) and use a **Lookup** field. Then the business can add products themselves, deactivate old ones, and store price, category, and description.

**Rule of thumb I share:** Choices are good for small, stable lists (under ~20 values) that rarely change — like Status, Priority, or Rating.

---

### Q8. How would you handle a requirement for multi-language data (English + Arabic + French)?
**Answer:**
I separate two things: **UI language** and **data language**.

**UI language** is easy: enable the language packs in the environment, and each user picks their language in Personal Settings. All labels, field names, views, and system messages translate automatically. For custom labels I export the **translation file** (a spreadsheet) from the solution, fill in the translations, and import it back.

**Data language** is the hard part. Dataverse does not translate the *content* users type. Options:
1. Add extra columns — `Description_EN`, `Description_AR`. Simple but ugly and does not scale.
2. Create a related **Translation table** with columns: Parent record, Language, Translated text. This scales to any number of languages.
3. Use **Azure Translator** through a Power Automate flow to auto-translate on save, then let a human review it.

I also remind the team about **Right-to-Left (RTL)** layout for Arabic and Hebrew, and about date/number formats.

---

### Q9. A user asks: "Why can't I make a calculated field that uses a field from a child record?"
**Answer:**
Because calculated columns can only look at:
- fields on the **same record**, and
- fields on a **parent record through a lookup** (one hop up).

They cannot look **down** to child records, and they cannot aggregate.

What I tell the user:
- To sum or count child records, use a **Rollup column** — that is exactly what it is for.
- If the rollup limits get in the way (more than one level deep, N:N, or needing real-time), use a **plugin** or a **Power Automate flow** writing into a normal number field.
- If it is only needed for reporting, do it in **Power BI** and do not store it in Dataverse at all.

I always finish with: "Let's understand the real reporting need first — sometimes we don't need to store the number at all."

---

### Q10. The data migration keeps failing with duplicate records. What is your approach?
**Answer:**
Steps I follow:
1. **Define what "duplicate" means** with the business first. Is it same email? Same name + postcode? Same tax number? Get this in writing.
2. **Clean the data in the source or in a staging area**, not in Dataverse. It is far faster and cheaper to fix a spreadsheet or a SQL staging table.
3. Use an **Alternate Key** in Dataverse on the natural key (for example, `Account Number` or `Email`). Then use **Upsert** instead of Create — Dataverse will update the existing record instead of creating a copy.
4. Turn **off** duplicate detection rules, plugins, real-time workflows, and flows during the bulk load to speed things up — then turn them back on and run a duplicate detection job.
5. Load in the right **order**: parent records first (Accounts), then children (Contacts), then transactions (Opportunities), then activities.
6. Keep a **legacy ID column** on every record so you can trace back to the source and re-run safely.

**Result:** "This approach let me re-run a failed load three times without creating a single duplicate."

---

### Q11. Business wants the Account "Name" field to be auto-generated like ACC-0001. How?
**Answer:**
I would use the **Autonumber column type** in Dataverse. It is built in, it handles concurrency safely, and it supports a prefix, a seed value, and a minimum length — for example `ACC-{SEQNUM:5}`.

Important points I mention:
- Autonumber is generated **on create only**, and it does not fill in for existing records — those need a one-time data update.
- If the requirement includes the **year or date** in the number (`ACC-2026-0001`), autonumber supports date placeholders like `{DATETIMEUTC:yyyy}`.
- If the number must reset each year, autonumber cannot do that natively — I would use a **plugin with a counter table plus a lock** to avoid duplicates under load.
- I would **never** use a Power Automate flow that reads the last record and adds 1 — two users saving at the same time will get the same number.

I also make the field read-only on the form so users cannot overwrite it.

---

### Q12. How do you decide between a Lookup, a Choice, and a Text field?
**Answer:**
My simple test:

**Use a Lookup when:**
- The item is a real "thing" with its own life (a Product, a Vendor, a Branch).
- The list changes often or is managed by the business.
- You need extra data about the item, or security on it.
- The list is long.

**Use a Choice when:**
- The list is short (under ~20), stable, and controlled by IT.
- It drives logic — business rules, flows, and views filter on it easily.
- Examples: Status Reason, Priority, Rating, Yes/No-plus-Maybe.

**Use Text when:**
- The value is genuinely free-form and unique per record — names, notes, addresses.
- Never use text for something you will filter or group by. It creates typos and useless reports.

I always add: "The wrong choice here is the most expensive mistake to fix later, because you have to migrate the data."

---

### Q13. The customer wants file attachments up to 500 MB. What do you advise?
**Answer:**
Dataverse can do it, but it is expensive and I would look for a better home for the files.

Facts I share:
- Notes/attachments and File columns consume **Dataverse File storage**, which is billed per GB.
- The default max file size is 32 MB and can be raised (up to 128 MB for File columns, and larger with chunked upload), but performance suffers.

My recommendation:
1. Store files in **SharePoint** using the out-of-the-box Dynamics 365–SharePoint document management integration. Users still see the documents inside the record, but the storage is SharePoint (much cheaper) and users get versioning, co-authoring, and Office integration.
2. For very large or streaming files, use **Azure Blob Storage** with a link stored in Dataverse.
3. If files must be secured exactly like the record, note that SharePoint security is folder-based and does **not** match Dataverse row-level security — so tell the client this trade-off up front.

---

### Q14. The team asks whether to use Business Units or a custom "Region" field for data separation. What do you say?
**Answer:**
This depends on whether it is a **security** need or a **reporting** need.

**Use Business Units when** users must not *see* each other's data. Business Units plus security roles give real row-level security. Each user belongs to one business unit, and roles can be scoped to User / Business Unit / Parent-Child / Organization. Modern Dataverse also supports **owning business unit at the record level** and **matrix data access**, so a user can work across units without moving.

**Use a Region field when** everyone can see everything but you just want to filter views, charts, and reports.

What I warn about: Business Units are **structural**. Changing the hierarchy later is painful and can cascade record ownership changes. So I always ask: "Will this structure be the same in 3 years?" If the org restructures often, I prefer flatter business units plus **Teams** (Owner Teams and Access Teams) for flexible sharing.

---

# Section 2 — Model-Driven Apps, Forms & Views (Q15–Q26)

### Q15. Sales users say the Opportunity form takes 15 seconds to load. How do you troubleshoot?
**Answer:**
I would measure before changing anything.

Steps:
1. Open the form with the browser **Network** and **Performance** tabs (F12), and use the built-in **Performance Insights** / Monitor tool for model-driven apps.
2. Common causes and fixes:
   - **Too many fields.** Move rarely used fields to a second tab. Tabs that are collapsed by default do not load until opened.
   - **Too many sub-grids and quick-view forms.** Each one is a separate query. Move them to tabs.
   - **Synchronous JavaScript** on OnLoad that calls the Web API. Make calls asynchronous, or use `async/await` correctly, and never use `async: false`.
   - **Too many business rules** running on load.
   - **Real-time (synchronous) plugins** on Retrieve — these block the form.
   - **iFrames and embedded canvas apps** loading up front.
3. Then re-measure and show the before/after numbers to the business.

**Result:** "On one project moving 4 sub-grids to lazy-loading tabs and making 3 JS calls async took the form from 14 seconds to 3 seconds."

---

### Q16. Different teams want completely different Opportunity forms. How do you handle it?
**Answer:**
Dataverse supports **multiple main forms per table**, and I control who sees which form using **form security roles**.

Steps:
1. Create one form per audience — for example "Opportunity – Sales", "Opportunity – Service", "Opportunity – Management".
2. In form properties, assign the **security roles** allowed to see each form.
3. Set the **form order**. The first form a user has access to becomes their default.
4. If they need to switch, the form selector at the top lets them.

**But first I challenge the requirement.** Often 80% of the form is identical and only a few sections differ. In that case one form plus **business rules** or **JavaScript** to show/hide sections is much cheaper to maintain than three forms that all need the same change every time.

I also mention that multiple forms multiply testing effort — every change has to be applied and tested three times.

---

### Q17. A manager wants a field to be mandatory only when Status = "Closed Won". How?
**Answer:**
Three options, from simplest to most powerful:

1. **Business Rule** (no code) — scope it to the form or to the entity. Condition: Status = Closed Won → Action: Set Business Required on the field. This is my first choice because any admin can maintain it. Note: business rules scoped to "Entity" run on the server too, so they protect against API updates.
2. **JavaScript** on the OnChange of Status: `formContext.getAttribute("fieldname").setRequiredLevel("required")`. Use this when the logic is too complex for a business rule (for example, checking data from another table).
3. **Plugin on Pre-Validation/Pre-Operation** — use this when the rule must be enforced **everywhere**, including imports, integrations, and Power Automate. Throw an `InvalidPluginExecutionException` with a friendly message.

**What I actually recommend:** business rule for the user experience, plus a plugin if data integrity really matters, because forms can always be bypassed by the API.

---

### Q18. Users say "I can't find anything" in the app. What do you do?
**Answer:**
This is usually a navigation and search problem, not a data problem.

My approach:
1. **Watch them work** for 30 minutes. Do not rely on the ticket description.
2. Fix the **sitemap**: group areas logically, remove tables users never touch, and rename items to business language (not "Account" if they say "Customer").
3. Turn on and configure **Dataverse search** (relevance search). Then set the **Quick Find** view columns properly — search only looks at columns in the Quick Find view's "Find Columns", and it returns results only from searchable tables.
4. Create **personal views** for common questions, or better, **system views** so everyone benefits.
5. Add **charts and dashboards** so the answer is visible without searching.
6. Set sensible **default views** per table.
7. Add **recently viewed** and **pinned** items training — many users do not know these exist.

---

### Q19. The client wants the Account form to look completely different — like a custom web page. How far can you go?
**Answer:**
I would first ask what problem they are really solving. "It looks like Dynamics" is not a business problem; "our users take 12 clicks to log a call" is.

What I can do inside a model-driven app:
- Rearrange sections, use multi-column layouts, use tabs and headers.
- Use **modern form styling** and the new look.
- Add **PCF (Power Apps Component Framework)** controls to replace individual fields or whole datasets with custom React/TypeScript UI.
- Embed a **canvas app** on the form for a fully custom area that still reads and writes Dataverse.
- Use **custom pages** in the model-driven app for a fully bespoke screen with model-driven navigation.

What I avoid:
- Unsupported DOM manipulation with jQuery. It breaks on every Microsoft update and is not supported.

**How I close:** "We can get 90% of the look you want in a supported way. Let's list the top three pain points and solve those, rather than rebuilding the whole form."

---

### Q20. A user needs to see 20 columns in a view but the screen is too small. What do you suggest?
**Answer:**
Options I would offer:

1. **Reduce the columns.** Ask which ones they act on versus just look at. Usually 6–8 are enough.
2. Use the **editable grid** control with column widths tuned, and let users freeze the first column.
3. Add a **quick view** or expand-on-hover so extra data appears without extra columns.
4. Give them **Excel export** or a "Open in Excel Online" for wide analysis — Excel is better than a grid for 20 columns.
5. Build a **Power BI report** if this is really analysis, not day-to-day work.
6. Use **multiple views** — "My Open Cases – Short" and "My Open Cases – Full".
7. Let them create **personal views** so they can tune it themselves.

**The message:** a CRM grid is for working on records, not for reading a spreadsheet.

---

### Q21. How would you implement a wizard-like, step-by-step process for a new customer onboarding?
**Answer:**
I would use a **Business Process Flow (BPF)**.

Design:
1. Define the stages with the business — for example: Lead Captured → Documents Collected → Credit Check → Approved → Onboarded.
2. In each stage, add the **steps (fields)** that must be filled before moving on, and mark the critical ones as required.
3. A BPF can **span multiple tables**, so the process can start on Lead and continue on Account and then on a custom Onboarding table.
4. Use **stage gating** plus **business rules** so users cannot skip ahead.
5. Add **workflows / flows triggered on stage change** to create tasks, send emails, or notify approvers.

Things I mention as an experienced person:
- A BPF is stored as its own table, so you can report on "how long did each stage take" — very valuable to the business.
- Users can have **multiple concurrent BPFs** on a record, and you can have up to 10 activated BPFs per table.
- If the process is very complex or needs branching per country, consider several simpler BPFs rather than one giant one.

---

### Q22. The business wants a button on the form that calls an external API. How do you build it?
**Answer:**
Steps:
1. Add a **command bar button** using the modern **command designer** (or Ribbon Workbench for older/complex cases).
2. Choose the action type:
   - **Run JavaScript** — the classic approach. The JS calls the API. But calling a third-party API directly from the browser causes CORS and secret-management problems.
   - **Run a Power Fx formula** — good for simple things like navigation or patching a field.
3. My preferred pattern for a real external API:
   - Button → JavaScript → calls a **custom API / custom action** in Dataverse → a **plugin** calls the external service, using credentials stored securely (Azure Key Vault via environment variables, or a secure config).
   - Or button → triggers a **Power Automate flow** through a Dataverse custom API or the flow's HTTP trigger.
4. Set the button's **visibility rules** using enable/display rules — for example, only show for a specific security role or when Status = Active.
5. Show the user progress and errors: use `Xrm.Utility.showProgressIndicator()` and a clear error dialog.

**Why the plugin route:** secrets never reach the browser, it works for the API and for bulk operations too, and it is fully supported.

---

### Q23. Users keep entering data in the wrong format (phone numbers, tax IDs). How do you prevent it?
**Answer:**
Layered approach — I never rely on just one layer:

1. **Field-level configuration**: use the right data type (Phone, Email, URL), set max length, and set the correct format so the platform validates.
2. **Business rules** for simple checks that the user sees immediately.
3. **JavaScript** on OnChange with a regular expression for pattern checks, and `setNotification()` to show a friendly inline message rather than a popup.
4. **Plugin on Pre-Operation** so the rule also applies to imports, integrations, and Power Automate — this is the only layer that truly cannot be bypassed.
5. **Duplicate detection rules** where relevant.
6. Add **help text / descriptions** on the field so the user knows the expected format before typing.
7. Clean the existing bad data with a one-time script, otherwise reports stay wrong.

**Key point I make in interviews:** UI validation is for user experience; server validation is for data integrity. You need both.

---

### Q24. How do you show a Contact's related Cases on the Account form?
**Answer:**
Out of the box, an Account form shows Cases where the **Customer** is that Account. If they want cases raised by *any contact of that account*, the standard sub-grid will not do it, because there is no direct relationship.

Options:
1. **Sub-grid with a related view** — works only if there is a direct relationship.
2. Create a **custom view on Case** with a linked-entity filter (Case → Contact → Parent Customer = current account) and point a sub-grid at it — but sub-grids filter on a relationship, so this needs the view to be filtered dynamically.
3. **Best supported approach:** add a sub-grid showing all Cases and filter it in **JavaScript** using `setFilterXml` on the grid control, building FetchXML that walks Case → Contact → Account.
4. Alternative: a **PCF control** or an embedded **canvas app** that queries exactly what is needed.
5. Simplest of all: add a `Account` lookup on Case and populate it with a plugin/flow when the case is created, then use a normal sub-grid. This is often the cleanest long-term answer.

---

### Q25. Management wants a dashboard showing team performance. Model-driven dashboard or Power BI?
**Answer:**
My decision rule:

**Use a model-driven (Dynamics) dashboard when:**
- The data is only in Dataverse.
- Users need to **click through to the record** and act on it.
- Security must follow Dataverse row-level security automatically.
- It needs to be free (no extra licence).

**Use Power BI when:**
- You need data from multiple systems (ERP, finance, web analytics).
- You need historical trends, year-over-year, or complex DAX calculations.
- You need large data volumes and fast aggregation.
- You need the report on mobile, in Teams, or emailed on a schedule.

**What I usually deliver:** both. Operational dashboards inside the app for daily work, and a Power BI report embedded as a **Power BI dashboard/tab** in the app for management. For Power BI I always plan **Row-Level Security (RLS)** carefully, because Power BI does not inherit Dataverse security automatically.

---

### Q26. A user says a record "disappeared" from their view. How do you investigate?
**Answer:**
I work through the most likely causes in order:

1. **Is it a filter?** The view might filter on Owner = me, or on Status = Active. If someone changed the owner or deactivated the record, it drops out. Check with Advanced Find / the new modern search with no filters.
2. **Was it deactivated?** Check the "Inactive" view.
3. **Was it deleted?** Check the **Audit history** on the table and the audit summary view — audit records the delete and who did it.
4. **Was it reassigned or shared differently?** If the record moved to another Business Unit or another owner, the user may have lost read access. Check the record's owner and the user's security role scope.
5. **Was it merged?** Merged records get deactivated and linked to the master.
6. **Did a flow or plugin change it?** Check the flow run history and plugin trace log.

Then I tell the user exactly what happened and what we will do to prevent it — that trust matters more than the fix.

---

# Section 3 — Canvas Apps & Power Apps (Q27–Q38)

### Q27. A canvas app takes 30 seconds to open. How do you make it fast?
**Answer:**
Slow canvas apps are almost always caused by loading too much data at start-up.

My checklist:
1. **Reduce OnStart work.** Move anything not needed immediately to `App.OnVisible` of the first screen, or to `Screen.OnVisible` of the screen that needs it.
2. Use **`Concurrent()`** to run independent data calls at the same time instead of one after another. This alone often cuts start-up in half.
3. **Do not load whole tables into collections.** Use delegable queries so the server filters, and only pull the rows the user sees.
4. Turn on the **"Delayed load"** and **"Named formulas"** app settings. Named formulas (`App.Formulas`) calculate only when used, which is much faster than `Set()` in OnStart.
5. Reduce the **number of controls per screen** — under 300 is a good target. Galleries with heavy nested containers are slow.
6. Turn off **unused connectors and data sources**; each one adds start-up cost.
7. Compress images, use SVG where possible, and avoid large media files inside the app.
8. Use the **Monitor tool** in Power Apps Studio to see exactly which call is slow.

**Result:** "By moving 6 OnStart calls into Concurrent() and switching to named formulas, I took an app from 28 seconds to 5 seconds."

---

### Q28. Your gallery only shows 500 records but the table has 50,000. What is happening?
**Answer:**
This is the **delegation limit**. Canvas apps only pull a maximum number of rows (default 500, can be raised to 2,000) when a formula is **not delegable** — that means Power Apps cannot push the filter to the server, so it downloads a page of data and filters locally.

How I fix it:
1. Find the blue warning triangle in Studio — it tells you exactly which function is not delegable.
2. Replace non-delegable functions with delegable ones. For Dataverse, `Filter`, `Search`, `LookUp`, `Sort`, `StartsWith`, `=`, `<`, `>`, `And`, `Or` are delegable. `Search()` with wildcards in the middle, `CountRows` on large sets, `Sum` over a filtered set, `in` on text, and `First()` inside a filter are common problems.
3. Avoid comparing to a **collection** inside a filter — that always breaks delegation.
4. If the logic cannot be delegated, do the heavy work elsewhere: a **Dataverse view**, a **Power Automate flow**, or a **custom API**, and return only the result to the app.
5. Redesign the screen so the user filters first (by branch, by date range) instead of scrolling through 50,000 rows. No human reads 50,000 rows anyway.
6. Raise the row limit to 2,000 only as a temporary help, never as the fix.

---

### Q29. The client wants one app that works on phone, tablet, and desktop. How?
**Answer:**
I would build a **responsive canvas app**, or honestly consider whether a model-driven app is a better fit (it is responsive out of the box).

For a responsive canvas app:
1. In App Settings, turn **off** "Scale to fit" and "Lock aspect ratio", and turn **on** orientation change support.
2. Use **containers** — especially horizontal and vertical layout containers — instead of absolute X/Y positions.
3. Use `Parent.Width` and `Parent.Height` in sizing formulas, never fixed pixel numbers.
4. Create **breakpoints** using `App.Width` (for example, small under 600, medium under 900, large above) and switch layouts based on that.
5. Test on real devices, not just the Studio preview.

What I tell the client honestly: a truly great phone experience and a great desktop experience are usually **two different designs**, not one stretched design. Sometimes two apps sharing the same components and data is the better answer.

---

### Q30. Users say "the app lost my data when the internet dropped." How do you handle offline?
**Answer:**
Steps:
1. Enable **offline mode**. Canvas apps in the Power Apps mobile player support offline with Dataverse using offline profiles; model-driven apps also support mobile offline through an offline profile in the admin centre.
2. Define an **offline profile** that syncs only the data the user needs (their records, last 30 days) — never the whole table.
3. In the app, use `Connection.Connected` to detect the state and show a clear banner: "You are offline. Changes will sync when you reconnect."
4. Use `SaveData()` / `LoadData()` to store pending changes locally, and a `Patch` loop on reconnect to push them.
5. Plan for **conflicts**: if two people edited the same record, decide the rule (last write wins, or flag for manual review) and agree it with the business.
6. Warn about limits: offline does not run server-side plugins, business rules scoped to entity, or flows until sync happens.

**The honest advice I give:** offline is expensive to build and test. If users are offline only occasionally, a simple "save as draft locally" is often enough.

---

### Q31. How would you share reusable UI (like a header and a menu) across 10 canvas apps?
**Answer:**
I would use a **component library**.

Steps:
1. Create a **Component Library** in the environment.
2. Build the header, navigation menu, loading spinner, and confirmation dialog as components with clear **custom input and output properties**.
3. Import the library into each app. When the library updates, apps show an "update available" prompt.
4. Package the component library inside a **solution** so it moves through Dev → Test → Prod properly.

Things I mention from experience:
- Components cannot directly access the app's data sources unless you pass data in as a property — this is by design and keeps them reusable.
- Component library updates are **not automatic**; each app owner must accept the update, so plan a communication process.
- For truly complex UI, a **PCF code component** may be better and can be used in both canvas and model-driven apps.

---

### Q32. A canvas app writes to SharePoint but sometimes records go missing. What do you check?
**Answer:**
My investigation order:
1. **Is the Patch inside an error handler?** Wrap writes in `IfError()` and log the failure. Silent failures are the number one cause of "missing" records.
2. **Check delegation** — a `LookUp` used before the Patch may be reading only the first 500 rows and finding nothing.
3. **Check SharePoint throttling.** SharePoint lists throttle heavily, and lists over 5,000 items hit the **list view threshold**, which breaks filters on non-indexed columns.
4. **Check column types.** SharePoint choice, lookup, and person columns need a specific record structure in Patch; getting it slightly wrong fails silently in some versions.
5. **Check permissions** — the connection runs as the signed-in user, so a user without contribute rights fails.
6. **Check for a race condition** — a `Patch` followed immediately by a `Refresh` and a navigate can look like data loss.

**My bigger recommendation:** if this is real business data with volume, move from SharePoint to **Dataverse**. SharePoint is a document store, not a database, and it has no transactions, no proper relationships, and no row-level security model like Dataverse.

---

### Q33. The business wants a "Submit for Approval" button in a canvas app. How do you design it?
**Answer:**
Design:
1. Button `OnSelect` → **Patch** the record to set Status = "Submitted" and record who submitted it and when.
2. Trigger a **Power Automate flow**. Two ways:
   - Call the flow directly from the app with the Power Apps trigger (`FlowName.Run(...)`) — good when you need a response back.
   - Or trigger the flow from the **Dataverse "When a row is modified"** trigger with a filter on Status — better, because it also works if the status changes from anywhere else.
3. In the flow, use the **Approvals** connector — "Start and wait for an approval". Choose the type (Approve/Reject, Everyone must approve, First to respond).
4. Handle both outcomes: on approve, update the record and notify; on reject, capture the **comments** and put the record back to Draft with the reason visible.
5. Add a **timeout / escalation** branch so an approval does not sit forever.
6. In the app, disable the button after submit and show clear status so users do not click twice.

**Important detail:** approvals need the approver to be licensed and to have the Approvals app. Also, approval history is stored in Dataverse in the Approvals tables, which is good for reporting.

---

### Q34. How do you handle errors gracefully in a canvas app?
**Answer:**
My standard pattern:
1. Turn on the **formula-level error management** app setting so `IfError()` and `Errors()` work properly.
2. Wrap every write in `IfError(Patch(...), Notify("Could not save: " & FirstError.Message, NotificationType.Error))`.
3. Never show raw technical errors to users. Map them to friendly messages.
4. Log failures — write to a Dataverse "App Log" table or send to **Application Insights** (canvas apps support Application Insights via the instrumentation key).
5. Use **`Notify()`** for transient messages and a modal container for blocking errors.
6. Disable the Submit button while a save is running (`Set(varSaving, true)`) to stop double submissions.
7. Validate **before** submitting so most errors never happen: check required fields, formats, and business rules in the app.

**What I say in interview:** "Users forgive a system that fails clearly. They do not forgive a system that fails silently."

---

### Q35. Someone built a canvas app in their personal environment and now it is business critical. What do you do?
**Answer:**
This is a classic **shadow IT / citizen developer** situation. I handle it in two parts: rescue, then govern.

**Rescue:**
1. Take a backup — export the app package immediately.
2. Move the app into a **solution**, and move it to a proper **Dev environment** owned by IT.
3. Identify all data sources and connections, and replace personal connections with **connection references** and **environment variables**.
4. Reassign ownership from the individual to a **service account or security group**, so the app does not break when they leave.
5. Set up Dev → Test → Prod with a pipeline.
6. Review security, licensing, and DLP compliance.

**Govern:**
1. Deploy the **Centre of Excellence (CoE) Starter Kit** to get visibility of every app, flow, and maker.
2. Set **DLP policies** so business data cannot mix with personal connectors.
3. Restrict who can create environments.
4. Create a simple "promote my app to production" process so makers ask instead of hiding.

**How I frame it to leadership:** "The maker did a good thing — they solved a real problem. Our job is to make the safe path the easy path."

---

### Q36. The app needs to show data from SQL Server on-premises. How?
**Answer:**
Steps:
1. Install the **on-premises data gateway** on a server that can reach the SQL Server and has a stable connection. For production I always install a **cluster** of at least two gateways for high availability.
2. Register the gateway in the Power Platform tenant, and give the right people gateway admin rights.
3. Create a **SQL Server connection** in Power Platform using the gateway, with a dedicated service account that has least-privilege access (usually read plus specific write to specific tables or, better, only to stored procedures).
4. In the app, connect to **views or stored procedures**, not raw tables. Views let the DBA control what is exposed and can be tuned.
5. Watch **delegation** — the SQL connector supports delegation for many functions, but not all.
6. Consider **virtual tables** in Dataverse instead, so SQL data appears as a Dataverse table and can be used in model-driven apps, flows, and views without copying data.

**What I raise as a risk:** the gateway is a single point of failure and needs patching, monitoring, and a documented owner.

---

### Q37. How do you decide between a canvas app, a model-driven app, and a Power Page?
**Answer:**
My simple rule:

**Model-driven app** — when the process is **data-first**: lots of tables, relationships, security roles, business process flows, and internal users who need to work with records all day. Fastest to build, responsive by default, and inherits Dataverse security. Example: a sales or service system.

**Canvas app** — when the process is **task-first and pixel-perfect**: a specific job like a site inspection, a stock count, or a guided form, often on mobile, often mixing several data sources. You control every pixel but you build everything yourself.

**Power Pages** — when the users are **external**: customers, partners, suppliers, or the public. It is a website with anonymous or authenticated access, and it has its own security model (web roles and table permissions), not Dataverse security roles.

**In real projects** I often use all three together: a model-driven app for staff, a canvas app embedded inside it for a specialised task, and a Power Page for customers to self-serve.

---

### Q38. A canvas app works for the maker but fails for everyone else. What is wrong?
**Answer:**
Nine times out of ten it is **sharing and permissions**.

My checklist:
1. Was the **app shared** with the users or a security group? Sharing the app is separate from sharing the data.
2. Do users have access to the **data**? Dataverse needs a security role granting privileges on those tables. SharePoint needs list permissions. SQL needs the connection permission.
3. Were the **connections shared**? For some connectors the maker must share the connection or the user must create their own.
4. Do the users have the right **licence** — for example, premium connectors (Dataverse, SQL, custom connectors) need Power Apps premium licensing or a Dynamics 365 licence.
5. Is a **DLP policy** blocking a connector for those users' environment?
6. Did the maker hard-code their own user ID, environment ID, or a personal SharePoint path?
7. Was the app **published** after the last change? Unpublished changes are only visible in Studio.

**How I prevent it:** always test with a real test user who has only the target security role — never test as an admin.

---

# Section 4 — Power Automate (Q39–Q52)

### Q39. A flow runs 10,000 times a day and keeps getting throttled. What do you do?
**Answer:**
Steps:
1. **Understand the trigger.** A "when a row is modified" trigger with no filter fires on every field change, including changes made by the flow itself — that causes loops.
2. Add **trigger filters**: filter rows (an OData condition), select specific columns to watch, and set the scope correctly (User / Business Unit / Organization).
3. Add a **trigger condition** so the flow only starts when it truly needs to.
4. Reduce actions per run. Every action counts against Power Platform request limits.
5. Use **batching** — instead of a loop with 1,000 individual updates, use a single action that handles the set, or call the Dataverse Web API `$batch` endpoint via an HTTP action or a custom API.
6. Set **concurrency control** on the loop, but be careful — higher concurrency finishes faster but throttles harder.
7. Add **retry policy** with exponential backoff on the actions that fail.
8. If the volume is genuinely huge, move the logic to a **plugin** or an **Azure Function** — code is far more efficient than a flow at scale.
9. Check licensing: request limits are per user per 24 hours, and **Power Platform Requests** add-ons or a service principal can help.

---

### Q40. A flow silently fails and no one notices for a week. How do you prevent that?
**Answer:**
Monitoring and alerting is the answer.

What I put in place:
1. **Configure run-after (error handling)** on key actions: add a parallel branch that runs on "has failed" or "has timed out", and send a Teams message or email to a support channel.
2. Use a **Scope** for the main logic plus a Scope for error handling, so one handler covers many actions. Use `result()` to get the failure detail.
3. Turn on **"Send email on flow failure"** notifications — but do not rely on them alone, because they go to the flow owner only.
4. Own flows with a **service account or a security group**, not an individual, so alerts do not disappear when someone leaves.
5. Build a **monitoring dashboard**: the CoE Starter Kit has flow analytics, or query run history through the Power Platform admin API into Power BI.
6. Log key steps to a Dataverse **Integration Log** table with a correlation ID, so support can trace a transaction end to end.
7. Add **Application Insights** for the environment.

**Result:** "We went from finding failures a week late to a Teams alert within 60 seconds, with the record link included."

---

### Q41. How do you handle a flow that must process 50,000 records?
**Answer:**
I would not do it in one flow run.

Design:
1. **Get the rows in pages.** Turn on **pagination** in the List Rows action and set a realistic threshold, and always use `$select` to fetch only the columns needed and `$filter` to reduce rows.
2. Use a **parent–child flow** pattern: the parent lists records and calls a child flow per batch, so each run stays small and one failure does not kill everything.
3. Set **loop concurrency** (for example 20–30) so records process in parallel, and test for throttling.
4. Add **checkpointing** — mark each record as processed so a re-run skips what is done. This makes the job restartable.
5. Consider whether this should be a flow at all. For pure bulk data work, a **console app with the Dataverse SDK using ExecuteMultiple**, an **Azure Function**, or a **dataflow** is much faster and cheaper.
6. Respect the **30-day flow run limit** and the 100 MB message limit.

**What I say:** "Flows are great for events. For bulk, use bulk tools."

---

### Q42. The client wants a flow to run every 5 minutes to check for new emails. Is that right?
**Answer:**
Usually no — I would use an **event-driven trigger** instead of polling.

Why polling is bad: it runs 288 times a day even when nothing happened, burns API requests and licence entitlement, adds delay of up to 5 minutes, and costs money.

Better options:
1. Use the **"When a new email arrives"** trigger — it is a push trigger and fires immediately.
2. For Dataverse, use **"When a row is added, modified or deleted"**.
3. For external systems, use a **webhook** into a flow's HTTP request trigger, or Azure Event Grid / Service Bus.
4. Only use a **recurrence trigger** when the source genuinely cannot push — for example an SFTP folder or a legacy database. Even then, tune the interval to the real business need (hourly is often fine).

**How I close:** "Let's start from what the business actually needs. If the answer is 'within an hour is fine', we save a lot of money."

---

### Q43. A flow needs to call an API that requires OAuth. How do you set it up securely?
**Answer:**
Steps:
1. Build a **custom connector**. In the security tab, choose **OAuth 2.0** and enter the identity provider, client ID, client secret, authorisation URL, token URL, and scope.
2. Never put the client secret in a flow action or a compose step. Store secrets in **Azure Key Vault** and reference them with the Key Vault connector, or use **environment variables of type secret** which point to Key Vault.
3. Prefer **service principal (client credentials)** authentication for system-to-system integration, so the flow does not depend on a person's token.
4. Use **connection references** in the solution so the connection can be re-pointed per environment without editing the flow.
5. Apply the connector to a **DLP policy** group deliberately (business vs non-business).
6. Add **retry with exponential backoff** and handle 401 by refreshing, 429 by honouring the `Retry-After` header.

**Alternative I mention:** if the API is complex, an **Azure API Management** layer in front gives you throttling, caching, logging, and one place to rotate secrets.

---

### Q44. Two flows are updating the same record and overwriting each other. How do you fix it?
**Answer:**
This is a **race condition**. My approach:

1. **Understand the business rule first** — who should win? Last write? Highest priority? A specific source of truth?
2. **Use optimistic concurrency.** Dataverse supports ETags / row version. In the Update action, pass the record version so the update fails if the record changed since you read it, then retry with fresh data.
3. **Serialise the work.** Set loop concurrency to 1, or use a queue (Azure Service Bus) so messages for the same record process in order.
4. **Reduce the overlap.** Have each flow update only the columns it owns, not the whole record. This alone fixes most cases.
5. **Add trigger filters** so each flow only reacts to the columns it cares about, breaking the ping-pong.
6. If it is truly transactional, move the logic into a **plugin**, because plugins run inside the Dataverse transaction and can lock properly.

**What I stress:** the fix is usually a design fix, not a technical trick. Two systems owning the same field is the real problem.

---

### Q45. How do you move flows from Dev to Production without breaking connections?
**Answer:**
The professional way:
1. Put every flow in a **solution** — never edit flows directly in production.
2. Use **connection references** instead of connections embedded in the flow. Then in each environment you point the reference at the right connection.
3. Use **environment variables** for anything that changes per environment: URLs, SharePoint site addresses, email addresses, API keys, feature switches.
4. Export as **managed** for Test and Production; keep unmanaged only in Dev.
5. Use **Power Platform Pipelines** (built in) or Azure DevOps / GitHub Actions with the Power Platform Build Tools for automated deployment.
6. Provide a **deployment settings file** (JSON) that supplies the connection reference and environment variable values, so the import is fully unattended.
7. After import, **turn on** the flows — imported flows can arrive off, especially if the owner has no connection.

**What breaks people:** flows owned by a leaving employee, and hard-coded GUIDs. Own everything with a service principal or service account.

---

### Q46. A flow must wait for a manager approval that can take 3 days. Any concerns?
**Answer:**
Yes, several, and I would design for them:

1. **Flow run duration limits.** A flow can run for up to 30 days, so 3 days is fine — but I would not design anything that waits weeks.
2. **Timeout and escalation.** Use the approval action's timeout, then a branch that reminds the approver, then escalates to their manager. Never let an approval sit forever.
3. **Delegation and absence.** What if the manager is on leave? Build a delegate lookup, or approve by **group** so anyone in the group can act.
4. **Visibility.** Users must be able to see "where is my request?" — so I write the approval status back to the Dataverse record, not just leave it inside the flow.
5. **Resilience.** If the flow is edited or the solution is re-imported while the run is waiting, in-flight runs can be affected. For long processes I prefer storing state on the record and using a separate trigger, rather than one long-running flow.
6. **Reminders** — a scheduled flow that chases pending approvals is usually more reliable than a delay inside the approval flow.

---

### Q47. The finance team wants a flow that posts to an ERP, but the ERP is sometimes down. How do you design it?
**Answer:**
I design for failure, because the ERP *will* be down.

Design:
1. **Retry policy** on the HTTP action — exponential backoff, 4–5 attempts. This handles short blips.
2. **Queue the message.** Write the payload to an **Azure Service Bus queue** or a Dataverse "Outbound Message" table with a status. A second flow (or Azure Function) drains the queue. This decouples CRM from the ERP.
3. **Dead-letter handling** — after N failures, move the message to a dead-letter queue and alert support with the payload and the error.
4. **Idempotency.** Send a unique message ID so if the same message is delivered twice, the ERP ignores the duplicate. This is essential with retries.
5. **Status on the record** so users can see "Pending", "Sent", "Failed" instead of guessing.
6. **Manual re-send button** for support, so they do not need a developer.
7. **Monitoring and alerting** on queue depth — if the queue grows, something is wrong.

**Result:** "With a queue in front, an 8-hour ERP outage caused zero data loss and zero user-visible failures — everything drained automatically."

---

### Q48. What is the difference between a classic workflow, a Power Automate flow, and a plugin? When would you use each?
**Answer:**

**Classic workflow (Dataverse background/real-time workflow)** — legacy, still supported but Microsoft recommends moving to Power Automate. Real-time workflows are still useful because they run **inside the transaction** and can be synchronous with no code.

**Power Automate cloud flow** — my default for most automation. Great for connecting systems, approvals, notifications, and scheduled work. Runs **outside** the Dataverse transaction, so it is asynchronous, can fail independently, and cannot roll back a Dataverse operation. Low code, easy to maintain.

**Plugin (C#)** — my choice when I need:
- Logic **inside the transaction** so it can roll back.
- Logic that runs on **every** channel — UI, API, import, flow — with no way to bypass it.
- Complex logic, heavy loops, or performance-critical work.
- Pre-validation to block a save with a clear message.

**How I decide:** "Can a flow do it reliably? Then flow, because more people can maintain it. Does it need to be atomic, guaranteed, or fast at scale? Then plugin."

---

### Q49. A flow's trigger keeps firing in a loop. What is happening and how do you stop it?
**Answer:**
This is the classic **infinite loop**: the flow triggers on "row modified", updates the row, which triggers the flow again.

Fixes:
1. **Trigger column filter** — set "Select columns" to only the columns you care about, so your own update does not re-fire it.
2. **Trigger condition** — add an expression so the flow only runs when a condition is true (for example, `@not(equals(triggerBody()?['statuscode'], 2))`).
3. **Flag column** — set a hidden "Processed" flag and add a trigger condition that skips processed rows. Remember to reset it.
4. **Check who modified it** — skip the run if `modifiedby` equals the flow's own service account.
5. Dataverse has **infinite loop protection** that stops runaway flows, but do not rely on it — it stops the flow entirely and users lose service.
6. If two flows trigger each other, draw the diagram. Cross-flow loops are common and invisible until you map them.

**Prevention:** always set trigger column filters as a standard, not as a fix.

---

### Q50. How do you test a flow before releasing it to production?
**Answer:**
My process:
1. **Unit-style testing in Dev** — use the "Test" button with manual and automatic triggers, and use past run data to replay real payloads.
2. Use **Compose** actions to inspect data at each step during development, then remove or keep them behind a debug flag.
3. Test the **negative paths**: missing data, wrong data type, API down, permission denied, empty result set, and very large result set.
4. Test with a **real user account and role**, not just as admin — flows run in the context of the connection, and permissions differ.
5. Deploy to a **Test environment** with production-like data volume, and run a load test if volume matters.
6. Have the **business do UAT** with a written test script and sign-off.
7. Check the **run history** for warnings, duration, and action count — a flow that works but takes 40 actions per run is a licensing problem.
8. Keep a **rollback plan**: the previous managed solution version, and the ability to turn the flow off quickly.

---

### Q51. Business users keep building their own flows that duplicate existing ones. How do you manage this?
**Answer:**
I treat this as governance, not as a technical problem.

What I do:
1. Deploy the **CoE Starter Kit** so I can see every flow, its owner, its connectors, and how often it runs.
2. Create a simple **catalogue** — a Power Page or SharePoint site listing approved, supported automations so people search before they build.
3. Set up **environment strategy**: personal/productivity environments for experimenting, and managed Dev/Test/Prod for anything business critical.
4. Apply **DLP policies** to stop risky connector combinations.
5. Run a **maker community** — monthly session, templates, standards, naming conventions, and office hours. Makers duplicate work because they do not know what exists.
6. Define a clear **promotion path**: if your flow becomes important, IT will adopt it, put it in a solution, and support it.
7. Use **Managed Environments** features: sharing limits, weekly digest, and solution checker enforcement.

**How I frame it:** "The goal is not to stop people building. It is to make sure the important things are supported and the risky things are visible."

---

### Q52. A flow needs to run as a specific service account, not the person who triggered it. How?
**Answer:**
Options depending on the trigger:
1. **Own the flow with a service account** and create the connections with that account. Then all Dataverse actions run as that account, regardless of who triggered it.
2. Use a **service principal (application user)**. Create an app registration in Entra ID, add it as an **application user** in Dataverse with a security role, then use it for the connection. This is the most robust because it has no password, no MFA problem, and no licence issue.
3. For a Dataverse trigger, note the flow runs under the **connection owner**, not the record's modifier, so the connection identity is what matters.
4. In a canvas app calling a flow, the flow can still run with elevated rights — this is a common way to let users do something they normally could not, in a controlled way.

**Security points I raise:**
- Give the service account **least privilege** — only the roles it needs.
- Document it and monitor it; a service account with System Administrator is a serious audit finding.
- Never use a personal account, because when that person leaves, every flow breaks.

---

# Section 5 — Security, Privacy & Compliance (Q53–Q62)

### Q53. Sales reps must only see their own opportunities, but managers must see their whole team's. How do you configure this?
**Answer:**
This is standard Dataverse security, done with **business units, roles, and hierarchy**.

Steps:
1. Create a **Sales Rep** security role with Read/Write/Create on Opportunity scoped to **User** level (the person icon — own records only).
2. Create a **Sales Manager** role with Read scoped to **Business Unit** or **Parent: Child Business Units** (the two-circle or full circle icons).
3. If managers manage a *team* rather than a business unit, enable **Hierarchy Security** in Settings. There are two models:
   - **Manager hierarchy** — based on the Manager field on the User record. A manager sees direct and indirect reports' data, with a configurable depth.
   - **Position hierarchy** — based on custom positions, useful when the org chart does not match data access.
4. For ad-hoc access, use **Access Teams** (share a specific record with a specific set of people) or **Owner Teams** (a team owns records).
5. Test with real users — I always create test accounts for each role and verify what they can and cannot see.

**What I warn about:** hierarchy security adds overhead to every query, so I only enable it on the tables that need it (you can select specific tables).

---

### Q54. A user has System Administrator rights "temporarily" and it has been 6 months. What do you do?
**Answer:**
I treat this as a real security risk and handle it properly.

Steps:
1. **Find out why** they needed it. Usually one specific privilege is missing from their normal role.
2. **Build a proper role** with just that privilege. Dataverse security is additive — you can give a user a small extra role alongside their main one, which is much better than admin.
3. **Remove System Administrator** and test.
4. If they genuinely need admin for occasional tasks, use **Privileged Identity Management (PIM)** in Entra ID so the role is granted just-in-time, with approval and an expiry, and it is fully audited.
5. Put a **review process** in place: quarterly access review of every user with System Administrator or System Customizer.
6. Turn on **auditing** for security role changes so any future grant is visible.

**How I explain it to the business:** "Every admin account is a possible breach. If we cannot say why someone has it, they should not have it."

---

### Q55. GDPR: a customer asks to be forgotten. How do you handle it in Dynamics?
**Answer:**
This needs a defined, repeatable process — not an ad-hoc delete.

Steps:
1. **Verify the request** and the person's identity, and check whether you have a **legal basis to keep** some data (for example, tax records or an active contract usually must be kept — "right to erasure" is not absolute).
2. **Find all the data.** The person can exist in Contact, Lead, Case, Activities (emails, phone calls, notes, attachments), audit history, marketing lists, Power Pages profile, SharePoint documents, backups, and any downstream system fed by an integration.
3. **Decide delete vs anonymise.** Deleting a Contact can break related records and reporting. Anonymising (replacing name, email, phone with a token like `REDACTED-12345`) usually satisfies the requirement while keeping referential integrity and financial history.
4. **Automate it** — a custom API or flow that anonymises consistently every time, with a log of what was done and when.
5. **Handle audit data** — audit logs may contain the old values, so you may need to delete the audit records for that entity too.
6. **Propagate to integrations** and to any data warehouse, Power BI dataset, or backup exports.
7. **Record the evidence** — regulators want proof that you did it and when.

**Also:** design for privacy in advance — consent fields, retention policies, and data minimisation are much cheaper than retrofitting.

---

### Q56. Your client is in a regulated industry and asks "where is our data stored?" What do you say?
**Answer:**
I would answer precisely and offer to show them documentation.

Key points:
1. Dataverse data lives in the **Azure region of the environment**, chosen when the environment is created. It cannot be moved easily afterwards — you would create a new environment in the right region and migrate.
2. Microsoft offers **regional and sovereign clouds** (for example EU, UK, Germany, US Government GCC/GCC High/DoD, China operated by 21Vianet).
3. The **EU Data Boundary** commitment keeps EU customer data processing within the EU for covered services.
4. Data is **encrypted at rest** (Microsoft-managed keys by default) and **in transit** (TLS). **Customer-managed keys (CMK)** are available for organisations that must hold their own key.
5. There are compliance certifications — ISO 27001, SOC 1/2, HIPAA, FedRAMP — available in the **Microsoft Trust Centre / Service Trust Portal**.
6. Watch out for data leaving the boundary through **connectors, integrations, Power BI datasets, and support diagnostics**. A DLP policy and a documented data flow diagram matter here.

**What I would do:** produce a data flow and data residency document as a project deliverable, and get it signed off by their compliance team early — not at go-live.

---

### Q57. How do you secure a single sensitive field (like Salary) so only HR sees it?
**Answer:**
Dataverse has **Column-level security (field security)**.

Steps:
1. On the column, set **Field Security = Enabled**.
2. Create a **Field Security Profile** (for example "HR – Salary Access").
3. In the profile, grant Read / Update / Create on that specific column.
4. Add the HR **users or teams** to the profile.
5. Everyone else sees `*******` on the form, in views, in Advanced Find, in exports, and through the API — it is enforced server-side, which is the key benefit.

Things I mention:
- System Administrators bypass field security by design — so be careful who is an admin.
- Field security affects **performance slightly** and cannot be applied to some system fields.
- It works for **Dynamics 365 and Dataverse**, but check that any downstream integration account also has the profile, or the field will arrive blank.
- Also remember **audit history** and **Power BI/Synapse exports** may bypass this — secure the data warehouse too.

---

### Q58. A partner needs access to only their own customers' records. How do you design it?
**Answer:**
Two very different designs depending on how they access the system.

**If partners log in to the internal app** (they have Dataverse licences):
1. Create a **Business Unit per partner** (or an Owner Team per partner).
2. Give them a role scoped to **Business Unit** so they see only their unit's records.
3. Assign records to the right business unit / owner via a plugin or flow when created.
4. Use **matrix data access / owning business unit** features if a partner user must work across two units.

**If partners access through a portal** (much more common and cheaper):
1. Build a **Power Pages** site.
2. Use **web roles** and **table permissions** with the "Contact" or "Account" scope — this automatically limits a portal user to records linked to their own contact or their parent account.
3. Use **Entra ID B2C** or an identity provider for authentication.
4. Never rely on hiding pages — always secure with table permissions, because the portal Web API can be called directly.

**Extra care:** I would explicitly test cross-tenant leakage — can Partner A guess Partner B's record ID and open it? That test must be part of UAT.

---

### Q59. Auditing was never turned on and now the client needs to know who changed a record last year. What do you say?
**Answer:**
I would be honest: if auditing was off, that history does not exist and cannot be recovered.

What I can still offer:
1. **`modifiedby` and `modifiedon`** on the record show the last change only.
2. **Plugin trace logs** or **flow run history** may show automated changes — but flow history is kept only 28 days by default.
3. **Backups** — if a system backup from that period still exists (7–28 days retention), a restore into a new environment could show the state then. A year later, almost certainly not.
4. **Downstream systems** — a data warehouse, Synapse Link export, or an integration log may have historical snapshots.

Then I turn it into a forward action:
1. Turn on **auditing** at the org level and choose tables and columns deliberately — auditing everything consumes a lot of log storage and costs money.
2. Set an **audit retention policy** (Dataverse supports retention periods) so you keep what compliance needs and delete the rest.
3. Consider **Microsoft Purview** and the unified audit log for user activity like logins, exports, and admin actions.

**The lesson I state:** auditing must be a day-one decision, driven by compliance, not an afterthought.

---

### Q60. A user exported 20,000 customer records to Excel. Is that a problem, and can you stop it?
**Answer:**
Yes, it can be a serious data-loss risk, and there are several controls.

What I would do:
1. **Investigate first** — was it a legitimate business need? Removing a capability without a replacement just creates a workaround.
2. **Remove the Export to Excel privilege** from roles that do not need it (it is a miscellaneous privilege in the security role).
3. Remove **Bulk Delete**, **Print**, and **Mail Merge** privileges similarly if not needed.
4. Apply **Microsoft Purview sensitivity labels** and DLP so the exported file is encrypted and tracked even outside the system.
5. Turn on **audit logging for export/read activity** through Purview so you get an alert when a large export happens.
6. Use **Managed Environments** and **Customer Managed Keys / IP firewall** to restrict where data can be accessed from.
7. Give a **legitimate alternative**: a Power BI report, a scheduled extract, or an approved API — so people do not need to export.

**How I frame it:** "Blocking export without giving a better option pushes people to screenshots and personal email, which is worse."

---

### Q61. How do teams (Owner Teams vs Access Teams) help with security? Give a real scenario.
**Answer:**

**Owner Team** — the team **owns** the record. Anyone in the team gets access through the team's security role. Use it when a group is jointly responsible: for example, a "Major Accounts Team" owns strategic accounts, and any member can work on them. Owner teams can hold security roles.

**Access Team** — the record stays owned by one person, but specific users are **granted access to that record only**, through an access team template. Use it when you need dynamic, per-record collaboration: for example, a Case where a specialist engineer is brought in for one case only.

**Real scenario I would describe:**
"On a banking project, each Corporate Account was owned by a Relationship Manager. But for a specific deal, a credit analyst, a legal officer, and a product specialist needed access to just that one opportunity. We used an **access team template** on Opportunity with Read/Write/Append-to privileges, and a sub-grid on the form where the RM added the people. No security role changes, no business unit changes, and access was per record — exactly what compliance wanted."

**Trade-off I mention:** access teams create rows in the Principal Object Access (POA) table, which can grow large and affect performance, so I do not use them for millions of records.

---

### Q62. Someone asks you to give "read-only access to everything" for an auditor. How?
**Answer:**
Steps:
1. Create a **new security role** — never copy System Administrator and try to remove things, because you will miss something.
2. Start from a low-privilege base, then grant **Read at Organization level** on the tables in scope. Leave Create, Write, Delete, Append, Append To, Assign, and Share **blank**.
3. Grant the required **miscellaneous privileges**: for example, "Read" on the app, access to relevant dashboards, and Advanced Find if they need it.
4. Decide about **Export to Excel** — auditors usually want it, so grant it but log it.
5. Explicitly **exclude sensitive columns** using field security profiles if the auditor should not see certain data.
6. Give access to the **audit history** privilege (View Audit History / View Audit Summary) if they need change history — this is a separate privilege.
7. Assign the role to a **named account with MFA**, time-limited, and record it in the access register.
8. Test as that user before handing over.

**One more point:** read-only in the app is not read-only everywhere — check Power BI, SharePoint documents, and any exports too.

---

# Section 6 — ALM, Solutions & DevOps (Q63–Q72)

### Q63. A developer made changes directly in production. How do you recover?
**Answer:**
This is a common and serious situation. My approach:

**Immediate:**
1. Do not panic-delete anything. First **document** what changed — use the solution history, audit logs, and the "Solution layers" view on affected components to see the unmanaged layer sitting on top.
2. Assess **risk**: is production working? If yes, we have time. If it is broken, roll back the specific change.

**Recover:**
1. In production, changes made directly create an **unmanaged layer** above the managed solution. That layer overrides the managed one and will block future deployments from taking effect.
2. Recreate the same change properly in **Dev**, inside the correct solution.
3. Deploy Dev → Test → Prod as managed.
4. Then **remove the unmanaged layer** in production (the "Remove active customisations" option on the component) so the managed layer takes control again.
5. Verify carefully — removing an active layer takes effect immediately.

**Prevent:**
1. Lock down production: remove System Customizer from everyone, keep only break-glass admin accounts.
2. Use **Managed Environments** and pipelines with approvals.
3. Add an emergency-change process with a hotfix branch, so people have a fast legitimate route.

---

### Q64. Explain managed vs unmanaged solutions to a non-technical stakeholder.
**Answer:**
I use a simple analogy.

"Think of an **unmanaged solution** as your working draft in Word — you can edit anything, add, remove, restructure. That is your development environment.

A **managed solution** is like a published PDF. It is installed as a finished package. People can use it, but they cannot casually change it. And crucially, if you uninstall it, it cleanly removes everything it added — like taking the PDF off the shelf."

Then the practical points:
- **Dev** is unmanaged. **Test and Production** get managed.
- Managed gives you clean uninstall, version control, and prevents accidental edits.
- Unmanaged in production is dangerous: you can never cleanly remove it, and deleting a component deletes the data with it.
- Managed solutions **layer** — a base solution plus extensions on top, applied in a defined order.

**The business benefit I stress:** "It means we can upgrade you safely, and we can undo a release. Without it, every change is permanent and risky."

---

### Q65. Your solution import fails with a dependency error. How do you fix it?
**Answer:**
Steps:
1. **Read the error properly** — download the import log (it is an XML/Excel file) and find the exact component and dependency that is missing.
2. Common causes:
   - A component (field, option set, web resource) exists in Dev but was never added to the solution.
   - The target environment has an **older version** of a dependent solution.
   - A **publisher prefix mismatch**.
   - A component was **deleted in Dev** but still referenced in the target.
   - Missing **connection references** or **environment variables**.
3. **Fix in Dev** by adding the missing dependency to the solution and re-exporting. Use "Add required components" carefully — it can pull in far more than you want.
4. Check **solution layering** — if an unmanaged layer in the target holds the component, remove it.
5. Use the **Solution Checker** and the "Show dependencies" view before exporting, as a standard pre-release step.
6. If a component must be removed, remember managed solution **deletion requires an upgrade** (use "Stage for upgrade" then "Apply upgrade"), not just an update.

**Prevention:** a clean, automated build from source control catches these in the pipeline, not at 11pm on release night.

---

### Q66. How would you set up ALM for a Power Platform project from scratch?
**Answer:**
My standard setup:

**Environments:**
- **Dev** (one per developer if the team is large, or one shared), unmanaged.
- **Test / QA** — managed, production-like data volume.
- **UAT** — managed, business testing.
- **Production** — managed, locked down.

**Source control:**
- Git repository (Azure DevOps or GitHub).
- Solutions are **unpacked** into source files using the Power Platform CLI (`pac solution unpack`) so you get real diffs and code review, not one giant zip.

**Pipeline:**
1. Export the unmanaged solution from Dev → unpack → commit to a branch → pull request → review.
2. Build pipeline: pack the solution, run **Solution Checker**, and produce a managed artefact.
3. Release pipeline: import to Test, run automated tests, then approval gate, then UAT, then Production.
4. Use **deployment settings files** for connection references and environment variables.

**Standards:**
- One publisher and prefix, agreed naming conventions, and a definition of done.
- Everything in a solution — no orphan components.
- **Power Platform Pipelines** is a great lower-effort alternative for teams without DevOps skills.

**Also:** back up production before every release, and rehearse the rollback.

---

### Q67. The client wants to deploy to production every day. Is that realistic?
**Answer:**
It can be, but only with the right foundation. I would answer with conditions.

**What must be true:**
1. Fully **automated pipeline** — no manual export or import steps.
2. **Automated tests**: unit tests for plugins (FakeXrmEasy or similar), UI tests (Power Apps Test Engine / EasyRepro / Playwright), and integration tests.
3. **Small changes.** Daily deployment works when each change is small. A 3-month batch cannot be deployed daily.
4. **Feature flags** using environment variables, so code can ship turned off and be enabled separately.
5. **Solution Checker** in the pipeline with a quality gate.
6. A tested **rollback** procedure.
7. Business readiness — users need training and comms; changing the UI daily confuses people.

**What I would say:** "Daily *deployment* is achievable. Daily *change to the user experience* is usually not what the business actually wants. Let's separate deploying code from releasing features."

---

### Q68. Two developers are working on the same table and keep overwriting each other. How do you prevent it?
**Answer:**
Steps:
1. **Separate Dev environments per developer** with a shared source control repository. Each developer exports and commits, and Git merges the changes.
2. If a shared Dev environment is unavoidable:
   - Split work by **table or module** so people rarely touch the same component.
   - Use **separate solutions** per feature (all pointing to the same publisher), so exports do not collide.
   - Agree a **check-out protocol** — a simple channel message: "I'm editing the Account form."
3. Understand what merges and what does not. **Forms, views, and sitemaps are XML** and merge badly — a merge conflict there is painful. Plan so two people never edit the same form at once.
4. Do frequent, **small commits** — big-bang merges are where things break.
5. Use **pull requests with review** so someone always sees what changed.
6. Use the **Power Platform CLI** unpack format, which splits components into many small files and makes merges far more manageable.

---

### Q69. How do you handle configuration data (reference data) between environments?
**Answer:**
Configuration data is not part of a solution, so it needs its own process.

My approach:
1. Identify true **configuration data**: countries, product categories, SLA definitions, email templates stored as records, mapping tables, settings.
2. Use the **Configuration Migration Tool** (part of the Dataverse SDK tools). You define a schema of which tables and columns to move, export a data zip, and import it into the target.
3. Use **alternate keys** on those tables so the import **upserts** instead of duplicating — this makes the process repeatable.
4. Store the schema file and the data file in **source control**, alongside the solution, and version them together.
5. Automate it in the pipeline using the **Power Platform Build Tools** "Import data" task or `pac data import`.
6. For per-environment values (URLs, keys, email addresses), do **not** use config data — use **environment variables** in the solution.

**What I warn about:** GUID mismatches. If configuration data was created independently in each environment, IDs differ and lookups break. Fix that early by making Dev the single source of truth.

---

### Q70. What do you do when a managed solution needs a component deleted?
**Answer:**
This is a well-known trap: a normal solution **update** does not delete components — it only adds and changes.

The correct process:
1. In Dev, **remove** the component from the solution and delete it from the environment.
2. Export the solution as **managed** with an incremented version.
3. In the target, import using **"Stage for Upgrade"** then **"Apply Upgrade"** — or simply choose the **Upgrade** option. This is the only path that removes components that are no longer in the new version.
4. Alternatively use **Update** if you deliberately want to keep old components (sometimes safer during a phased release), then do a proper upgrade later.

**Cautions:**
- Deleting a column **deletes the data** in it. Take a backup and export the data first.
- Check dependencies — a field used in a view, form, business rule, flow, or plugin will block the delete until those are updated.
- Upgrade also cleans up the **holding solution** left behind by staging; make sure it completes.
- In production, always test the upgrade in Test first with a copy of production.

---

### Q71. How do you version and document your solutions?
**Answer:**
**Versioning:**
- Use **major.minor.build.revision** (for example `1.4.20260820.1`).
- Increment major for breaking changes, minor for new features, build/revision automatically from the pipeline (date or build number).
- Never re-import the same version number with different content — you cannot tell environments apart afterwards.

**Documentation:**
1. **Release notes** per version: what changed, why, who approved, and rollback steps.
2. **Solution structure doc**: which solution owns which components, and the publisher prefix.
3. **Data model diagram** (ERD) kept current — I use the Power Platform CLI or a tool like XrmToolBox's Metadata Document Generator to produce it automatically.
4. **Configuration decisions log** — why we chose a rollup over a plugin, why we used a junction table. Future developers need the *why*, not just the *what*.
5. **Runbook** for support: common errors, service accounts, integration endpoints, and who to call.
6. Keep it all in the **repo** next to the code, so it versions with the solution and cannot go stale on someone's laptop.

---

### Q72. A release broke production. Walk me through what you do.
**Answer:**
I follow incident management: stabilise, communicate, fix, learn.

**1. Stabilise (first 15 minutes):**
- Assess the blast radius: who is affected and how badly? Is it data corruption (urgent) or a broken screen (annoying)?
- If data is being corrupted, **turn off the flows/plugins** causing it immediately.
- Roll back: re-import the previous managed solution version, or restore from the pre-release backup if needed.

**2. Communicate:**
- Tell users what is happening and give a workaround, in plain language, within minutes. Silence destroys trust.
- Keep stakeholders updated at agreed intervals.

**3. Fix properly:**
- Reproduce in Test, find the real root cause, fix in Dev, test, and redeploy through the normal pipeline. Do not hot-fix production directly unless it is a true emergency, and if you do, back-port it immediately.

**4. Learn:**
- Run a **blameless post-mortem**. Ask "what in our process let this through?" not "who did it?"
- Add the missing test, the missing gate, or the missing monitoring.

**What I would add in an interview:** "The measure of a good team is not that nothing breaks — it is how fast and how calmly you recover, and whether the same thing happens twice."

---

# Section 7 — Integration & APIs (Q73–Q80)

### Q73. You must integrate Dynamics 365 with SAP. Where do you start?
**Answer:**
I start with the business, not the technology.

**Discovery questions:**
1. What data moves, in which direction, and who is the **master** for each field? This is the single most important decision.
2. How fresh must it be — real time, near real time (minutes), or nightly?
3. What volume — 100 records a day or 100,000?
4. What happens on failure? Can we retry? Is duplication acceptable?

**Then the technical design:**
1. Prefer **event-driven** where possible: Dataverse plugin/webhook → Azure Service Bus → integration layer → SAP.
2. Use a **middleware layer** — Azure Logic Apps, Azure Functions, or Azure Integration Services — rather than point-to-point. This gives retries, transformation, logging, and one place to change.
3. Consider the **SAP ERP / SAP OData connector** in Power Platform with an on-premises gateway for simpler cases.
4. Store a **cross-reference / mapping table** so each system's ID is linked (Dataverse GUID ↔ SAP number).
5. Build **idempotency** and a **correlation ID** into every message.
6. Design **error handling and a replay mechanism** from day one.
7. Agree the **data contract** in writing and version it.

**What I would emphasise:** "Most integration projects fail on data quality and ownership, not on the connector."

---

### Q74. An external system needs to create records in Dataverse. What is the best way?
**Answer:**
Options, and how I choose:

1. **Dataverse Web API (OData v4)** with **service principal** authentication — my default. Register an app in Entra ID, create an **application user** in Dataverse, give it a least-privilege security role, and use client credentials. No user licence needed for the app user in most cases, and no password to rotate if you use a certificate.
2. **Custom API / Custom Action** — instead of exposing raw table CRUD, expose a **business operation** like `CreateOrderFromPortal`. This is much better: you control validation, you can change the internal model without breaking the caller, and you keep the contract stable.
3. **Azure Logic Apps / Power Automate with an HTTP trigger** — good for low volume and for teams without developers.
4. **Batch operations** ($batch) when creating many records — far more efficient than one call per record.
5. **Virtual tables** if the data should stay in the other system and just be viewed in Dataverse.

**What I always add:** throttling awareness (Dataverse has service protection limits — handle 429 with the `Retry-After` header), and a monitoring/logging plan with correlation IDs.

---

### Q75. The integration is duplicating records every time it runs. How do you fix it?
**Answer:**
The root cause is nearly always **no natural key and no upsert**.

Fix:
1. Define the **natural key** with the business — the value that uniquely identifies the record in the source system (customer number, email, order ID).
2. Create an **Alternate Key** on that column in Dataverse. Dataverse enforces uniqueness at database level and lets you address records by that key.
3. Change the integration from **Create** to **Upsert** using the alternate key: `PATCH /accounts(accountnumber='ACC123')`. Dataverse creates if it does not exist and updates if it does.
4. Store the **source system ID** on every record for traceability.
5. Make the integration **idempotent** — the same message processed twice must produce the same result.
6. Add a **message ID / dedupe check** so replayed messages are ignored.
7. Clean up the existing duplicates with a **merge** process (Dataverse merge keeps related records), not a delete, so history is preserved.

**Result:** "After adding alternate keys and switching to upsert, the nightly job could be safely re-run any number of times with zero duplicates."

---

### Q76. How do you handle an integration where the other system is very slow (30 seconds per call)?
**Answer:**
I would never let a user wait 30 seconds. My design:

1. Make it **asynchronous**. The user action writes a request record and returns immediately with "Processing…".
2. A **queue** (Azure Service Bus) holds the requests; a worker (Azure Function or flow) processes them at whatever pace the slow system allows.
3. Write the **result back** to the record and notify the user — a Teams message, an in-app notification, or just a status field they can see.
4. Increase **timeouts** appropriately and use **retry with backoff**.
5. **Cache** where possible — if the slow call returns reference data that changes rarely, cache it in Dataverse and refresh on a schedule.
6. **Batch** requests if the API supports it, to reduce the number of slow round-trips.
7. Add **circuit breaker** logic — if the system is failing repeatedly, stop calling it for a while instead of piling up requests.
8. Talk to the other team — sometimes 30 seconds is a missing index on their side, and the real fix is theirs.

**Never do:** call a slow API from a **synchronous plugin**. Dataverse plugins have a 2-minute limit and you will block the user's save and possibly time out the whole transaction.

---

### Q77. What is the difference between synchronous and asynchronous plugins, and when do you use each?
**Answer:**

**Synchronous (real-time)** plugins run **inside** the Dataverse transaction, while the user waits.
- Use when you must **validate and block** a save, or when a change must be atomic (either everything succeeds or everything rolls back).
- Stages: **Pre-Validation** (before the transaction — best for validation and for checking security), **Pre-Operation** (inside the transaction, before the save — best for modifying the incoming data), **Post-Operation** (inside the transaction, after the save — best for creating related records).
- Limit: 2 minutes execution. Anything slow here hurts every user.

**Asynchronous** plugins run **after** the transaction, in the background via the async service.
- Use for anything not needed immediately: sending notifications, calling external systems, heavy calculations, logging.
- The user is not blocked, and a failure creates a **System Job** you can retry.
- The trade-off: it is eventually consistent, and it can fail without the user knowing — so add monitoring.

**My rule:** "Synchronous only when the business breaks without it. Everything else async."

---

### Q78. How would you expose Dynamics data to a public website safely?
**Answer:**
I would never expose Dataverse directly to the public internet.

Safe design:
1. Put an **API layer** in front — Azure API Management or an Azure Function — that the website calls.
2. That layer authenticates to Dataverse with a **service principal** holding a least-privilege role that can only read the specific tables and columns needed.
3. Expose only a **purpose-built contract** (for example `GET /products`), not raw Dataverse entities. Never let the caller pass arbitrary OData.
4. Add **rate limiting, caching, and WAF** protection at the API layer. Caching also protects Dataverse from load and from service protection limits.
5. **Never** put a Dataverse client secret in browser JavaScript.
6. If users need to see **their own** data, use **Power Pages** with table permissions and an identity provider — it is built for exactly this and handles the per-user security properly.
7. Log every request with a correlation ID, and monitor for scraping patterns.

**Security testing I insist on:** attempt to read another user's record by changing the ID in the request. If that works, the design is wrong.

---

### Q79. How do you get Dataverse data into a data warehouse for analytics?
**Answer:**
Options, best first:

1. **Microsoft Fabric Link (Link to Microsoft Fabric)** — the current recommended approach. It gives near real-time, no-copy access to Dataverse data in Fabric OneLake, with no ETL to build and no impact on the Dataverse API limits.
2. **Azure Synapse Link for Dataverse** — exports data continuously to Azure Data Lake in Delta/Parquet format. Great for large volumes and for combining with other sources.
3. **Dataflows** — low-code ETL, good for moderate volumes and for makers.
4. **Data Export Service** — deprecated, do not use for new work.
5. **Direct API extraction** with a custom job — only when you need very specific logic; it consumes API limits and needs careful paging.

Design points I raise:
- Do **incremental** loads using `modifiedon` and a watermark, not full loads.
- Handle **deletes** — Synapse Link and Fabric Link track them; a naive extract does not.
- Decide how to handle **choice/option set labels** (store both value and label) and **lookups** (store both GUID and name).
- Plan **security** in the warehouse separately — Dataverse row-level security does not travel with the data.

---

### Q80. A webhook from Dataverse to an external service keeps failing. How do you debug it?
**Answer:**
Steps:
1. Check the **System Jobs** view in Dataverse — webhooks registered as asynchronous steps show failures there with the error message.
2. Check the **Plugin Trace Log** if the webhook is triggered by a plugin.
3. Verify the **endpoint** is reachable and returns a **2xx quickly**. Dataverse webhooks expect a fast response; long processing must be queued on the receiving side.
4. Check **authentication** — webhook headers, the auth type (HttpHeader or WebhookKey), and whether the key or token has expired.
5. Check **payload size** — very large payloads can fail.
6. Check **firewall / IP allow-listing** on the receiving service; Dataverse calls come from Azure IP ranges.
7. Check whether the receiving service is **throttling** you (429) or returning 5xx — then implement retry and a queue.
8. Test the endpoint independently with a tool like curl or Postman using the exact same payload.

**Better architecture if failures continue:** switch from a direct webhook to **Azure Service Bus**. Dataverse can post to Service Bus natively, and the queue gives you durability, retries, and dead-lettering that a raw webhook does not.

---

# Section 8 — Plugins & Custom Code (Q81–Q86)

### Q81. Your plugin causes a "timeout" error when saving. What do you do?
**Answer:**
Plugins have a hard **2-minute limit** (and synchronous plugins block the user).

My steps:
1. **Find what is slow** using the **Plugin Trace Log** with timing statements, or Application Insights.
2. Common causes:
   - Calling an **external API** synchronously.
   - **Retrieving too much data** — a RetrieveMultiple with no filter or paging.
   - Querying **inside a loop** (the N+1 problem) instead of one query up front.
   - **Recursion** — the plugin updating the record that triggers itself.
3. Fixes:
   - Move the slow part to an **asynchronous** plugin or a flow.
   - Use `ColumnSet` with only the needed columns, never `new ColumnSet(true)`.
   - Use **ExecuteMultiple** for bulk operations.
   - Cache reference data using `IOrganizationServiceFactory` sparingly, or better, in a static read-only structure that is safe for a stateless plugin.
   - Add a **depth check** (`context.Depth > 1`) or a shared variable to stop recursion.
4. Re-measure and prove the improvement.

**Design principle:** synchronous plugins should complete in well under a second. If it cannot, it does not belong there.

---

### Q82. How do you make a plugin work differently in Dev, Test, and Production?
**Answer:**
Never with `if (environment == "Dev")` hard-coded. My options:

1. **Environment variables** in the solution — the plugin reads the value at runtime using the Dataverse SDK. This is the modern, supported approach and the value is set per environment at deployment.
2. **Unsecure and Secure configuration** on the plugin step. Secure config is not exported with the solution, which is good for secrets but means you must set it per environment (and remember to, or the plugin breaks).
3. **Azure Key Vault** for real secrets, accessed through a managed identity — the best option for credentials.
4. A **Configuration table** in Dataverse holding settings as records, read by the plugin with caching. Easy for admins to change without a deployment.

**What I recommend:** environment variables for behaviour and endpoints, Key Vault for secrets, and a config table for values the business needs to change themselves.

**Also:** always log which configuration was loaded (but never log the secret) so support can diagnose "it works in Test but not Prod".

---

### Q83. When would you write a plugin instead of using low-code?
**Answer:**
I default to low-code and only write a plugin when there is a real reason. My reasons:

1. **Transactional integrity** — the logic must succeed or roll back with the record. Only plugins run inside the Dataverse transaction.
2. **Cannot be bypassed** — the rule must apply to the UI, the API, imports, and flows equally.
3. **Performance at scale** — thousands of records where a flow would be too slow or too expensive in API requests.
4. **Complex logic** — heavy loops, recursion, complex algorithms, or calling libraries that low-code cannot.
5. **Blocking validation with a friendly message** at save time (`InvalidPluginExecutionException`).
6. **Auto-numbering with concurrency safety.**

**When I do NOT use a plugin:**
- Notifications, approvals, and cross-system messaging → Power Automate.
- Simple field defaults and show/hide → business rules.
- Anything the client's team must maintain and they have no C# developers.

**The honest point I make:** "Every plugin is a long-term maintenance commitment. Code needs a developer, a build pipeline, and unit tests. Low-code needs an admin. Choose accordingly."

---

### Q84. How do you unit test a plugin?
**Answer:**
Steps:
1. **Write testable code.** Keep the `Execute` method thin — it should just read the context and call a separate class that contains the business logic and takes an `IOrganizationService` as a parameter. Never put logic directly in `Execute`.
2. Use a **mocking framework**: **FakeXrmEasy** is the standard for Dataverse and can simulate the full pipeline, or use Moq / NSubstitute to mock `IOrganizationService`, `IPluginExecutionContext`, and `ITracingService`.
3. Write tests for:
   - The **happy path**.
   - **Missing or null attributes** — a very common cause of `NullReferenceException` in real plugins, because the target only contains changed fields.
   - **Wrong message or stage** — the plugin should exit safely.
   - **Depth / recursion** guard.
   - **Exception handling** — does it throw `InvalidPluginExecutionException` with a friendly message?
4. Run tests in the **build pipeline** so a failing test blocks the release.
5. Add **integration tests** against a real Dev environment for the things mocks cannot prove.

**Key point:** the `Target` entity in a plugin only contains the attributes that changed. Most plugin bugs come from assuming a field is present.

---

### Q85. A plugin needs to call another Dataverse operation. What should you be careful about?
**Answer:**
Several things:

1. **Recursion.** Your update triggers your own plugin again. Guard with `context.Depth > 1` (Dataverse also stops runaway recursion at depth 8 by default) or use `SharedVariables` to mark work already done.
2. **Which service to use.** `CreateOrganizationService(context.UserId)` runs as the calling user (respects their security, which is usually correct). `CreateOrganizationService(null)` runs as the **system user** and bypasses security — powerful and dangerous, so use it deliberately and document why.
3. **Transaction scope.** In Pre/Post-Operation you are inside the transaction, so your extra operations roll back with it — that is good. But long operations extend the transaction and cause locks.
4. **Performance.** Do not query inside a loop. Fetch once, then work in memory.
5. **Exception handling.** Do not swallow exceptions. If something fails and you catch it silently, data ends up inconsistent.
6. **Do not call external HTTP from a synchronous plugin** — use an async step, a webhook, or Service Bus.
7. **Sandbox restrictions** — plugins run in sandbox isolation with no file system access, no registry, and limited network.

---

### Q86. How do you debug a plugin running in production?
**Answer:**
You cannot attach a debugger to production, so I rely on tracing and telemetry.

Steps:
1. **Plugin Trace Log** — call `ITracingService.Trace()` at key points with meaningful, structured messages including a correlation ID. Set the trace setting to "Exception" normally and "All" temporarily when investigating, then turn it back off (it generates a lot of data).
2. **Application Insights** — connect the environment to App Insights for full telemetry, dependency tracking, and performance metrics. This is the modern best practice.
3. **Plugin Profiler** (in the Plugin Registration Tool) — capture a real production execution as a profile, then **replay and debug it locally** in Visual Studio with the exact same context. This is the closest thing to live debugging and it is fully supported.
4. **System Jobs** view for asynchronous plugin failures, with the error and stack trace.
5. **Audit history** on the record to see what actually changed.

**Discipline I insist on:**
- Never write secrets or personal data to trace logs.
- Always include the record ID and the correlation ID so support can join the dots across plugin, flow, and integration logs.

---

# Section 9 — Power Pages / Portals (Q87–Q90)

### Q87. Customers must log in to a portal and see only their own cases. How do you set it up?
**Answer:**
Steps:
1. Create a **Power Pages** site linked to the Dataverse environment.
2. Set up **authentication** — Entra ID B2C is the usual choice for customers, or local authentication for simple cases. Configure the identity provider and invitation/registration process.
3. Each portal user is a **Contact** record in Dataverse, linked to an Account.
4. Create a **Web Role** (for example "Customer").
5. Create **Table Permissions** on Case with the scope set to:
   - **Contact** — the user sees cases where the Customer/Contact field is them.
   - **Account** — the user sees cases for their whole company (via parent customer).
6. Assign the table permission to the web role, and set the privileges (Read, Create, Write, Append, Append To) deliberately.
7. Build the **list** and **form** pages, and attach the table permissions to them.
8. Do **not** rely on hiding links — the portal Web API and the list endpoints can be called directly, so security must be in table permissions.

**Testing I insist on:** log in as Customer A and try to open Customer B's case by changing the ID in the URL. It must fail.

---

### Q88. A Power Pages site is very slow. What do you check?
**Answer:**
1. **Caching** — Power Pages caches server-side, but a change in Dataverse may not appear instantly. Conversely, disabling cache kills performance. Understand and tune it.
2. **Liquid code** — nested loops over `entities` with unfiltered queries are the biggest cause. Use `fetchxml` with filters, paging, and only the columns needed.
3. **Number of records on a list page** — enable paging and sensible page sizes.
4. **Table permissions complexity** — deep permission chains cost query time.
5. **Images and static files** — use the CDN, compress images, and minify CSS/JS.
6. **Web API calls from the browser** — batch them, and avoid calling on every keystroke.
7. Check the **site capacity / tier** and whether it should be scaled up.
8. Use browser dev tools and the **Power Pages diagnostics** to see where time actually goes.

**Then re-test** and show a before/after measurement, ideally from the users' real location and network.

---

### Q89. The business wants anonymous users to submit a request without logging in. Any concerns?
**Answer:**
It is possible, but I would raise these:

1. **Spam and bots.** Add **CAPTCHA** on the form (Power Pages supports this out of the box) and rate limiting.
2. **Data quality** — anonymous submissions cannot be tied to a known customer, so you get duplicates and bad data. I would add a matching/dedupe process on arrival, and route them to a **Lead** or a staging table rather than straight into Contact/Case.
3. **Security** — the anonymous web role must have the absolute minimum permissions: Create on one table, and no Read at all if possible. Never give anonymous users Read on customer data.
4. **Attachments** — anonymous file upload is a malware risk; scan files or disable uploads.
5. **Privacy** — you are collecting personal data, so you need consent text, a privacy notice, and a retention rule.
6. **Confirmation** — send an email with a reference number so the person can follow up, and consider a "magic link" for status checking rather than full registration.

**How I would advise:** allow anonymous submission but write to a quarantined table, then have a flow validate, dedupe, and promote it into the main data.

---

### Q90. How do you deploy Power Pages changes between environments?
**Answer:**
Power Pages configuration lives in Dataverse tables, so it moves as **site data**, not only as a solution.

Options:
1. **Power Platform CLI** — `pac paportal download` to get the site as files, commit to source control, then `pac paportal upload` to the target. This is the standard for DevOps and gives real diffs on web pages, templates, and CSS.
2. **Power Pages in solutions** — newer sites can be added to a solution and deployed like any other component, which is the direction Microsoft is moving.
3. **Configuration Migration Tool** with the portal schema, for the older approach.
4. Custom code (JS, CSS, Liquid) should live in **source control**, not be edited directly in production.

Practical points:
- **Environment-specific values** (site URL, identity provider settings, keys) must be updated after deployment — script this or use environment variables.
- **Site settings** and **content snippets** may be maintained by marketing in production; agree who owns what, otherwise a deployment overwrites their content.
- Always take a **backup / download** of the target before uploading.

---

# Section 10 — Reporting & Power BI (Q91–Q93)

### Q91. Management says "the numbers in the CRM dashboard don't match the Power BI report." How do you investigate?
**Answer:**
This is nearly always a definition problem, not a bug.

My steps:
1. **Agree the definition first.** What exactly is "revenue"? Estimated or actual? Including tax? Which currency? Which date — created, close, or modified? Nine times out of ten the two reports are counting different things and both are "right".
2. **Check the date range and time zone.** Dataverse stores dates in UTC; Power BI may convert differently. A day's difference at month end changes the total.
3. **Check the filters** — one may exclude inactive records, the other may not.
4. **Check the refresh time.** Power BI import mode may be showing yesterday's data. Look at the last refresh timestamp.
5. **Check security.** The dashboard respects Dataverse row-level security automatically; Power BI does not unless RLS is configured. So a manager may see more or fewer rows.
6. **Check currency conversion** — Dataverse stores base currency and transaction currency; using the wrong one causes exactly this.

**Then I fix the root cause:** publish a **single agreed definition** (a data dictionary / semantic model) so both reports use the same logic, and ideally build one shared dataset that both use.

---

### Q92. The client wants a report showing data as it was 6 months ago (point in time). How?
**Answer:**
Dataverse stores the **current** state, not history — so this needs design.

Options:
1. **Snapshot table.** A scheduled job writes a daily/monthly snapshot of key figures into a separate table. Simple, reliable, and reportable. Best when you know in advance what you need.
2. **Audit history** — it records changes, but it is not designed for reporting, is slow to query, and may have been purged. I would not build a report on it.
3. **Data warehouse with slowly changing dimensions (SCD Type 2)** — the proper answer for serious historical reporting. Use **Fabric Link** or **Synapse Link** to land the data, then build history in the warehouse.
4. **Business Process Flow records** — these naturally store stage entry/exit times, so "how long in each stage" is available without extra work.
5. **Status change tracking** — enabling "status change" tracking on a table records when the status changed, useful for lifecycle reporting.

**What I say:** "If the business will ever ask 'what did this look like last quarter?', we must design for it now. Retrofitting history is impossible — the data is already gone."

---

### Q93. How do you handle Power BI security so a regional manager only sees their region?
**Answer:**
Use **Row-Level Security (RLS)** in Power BI. It is not inherited from Dataverse, so it must be built.

Steps:
1. In Power BI Desktop, create a **role** (for example "Regional Manager").
2. Add a DAX filter on the region dimension. For dynamic security use `USERPRINCIPALNAME()`:
   - Build a **user-to-region mapping table** (ideally sourced from Dataverse, so it stays in sync with the actual org structure).
   - Filter: `[UserEmail] = USERPRINCIPALNAME()`, and let the relationship filter the fact table.
3. Publish, then in the Power BI Service assign **users or security groups** to the role.
4. Use **"Test as role"** in Desktop and "Test as user" in the Service to verify.
5. For **DirectQuery over Dataverse**, you can sometimes rely on the underlying security, but check carefully — behaviour depends on the connection type and the credentials used.

**Points I raise:**
- RLS filters rows, it does **not** hide columns or measures — for that use object-level security (OLS) or a separate report.
- Anyone with **workspace admin/member** rights bypasses RLS, so control workspace access tightly.
- Keep the mapping table automated. Manual mapping tables always go stale and become a security incident.

---

# Section 11 — Licensing, Governance & Adoption (Q94–Q96)

### Q94. The client is shocked by the licence cost after go-live. How could this have been avoided?
**Answer:**
Licensing must be designed at the start, not discovered at the end.

What I would have done:
1. **Map every user to a persona** in week one: who needs Dynamics 365 Sales/Customer Service, who needs Power Apps Premium, who only needs to view a report, who is external.
2. Understand what triggers a **premium licence**: Dataverse, custom connectors, on-premises gateway, and premium connectors all require Power Apps/Power Automate Premium or a Dynamics 365 licence.
3. Consider **Power Apps per-app** licensing for users who only use one or two apps, instead of per-user.
4. Use **Power Pages capacity** for external users instead of giving them full licences — this is often the single biggest saving.
5. Watch **storage**: Dataverse database, file, and log storage are billed separately and log storage grows fast when auditing is on.
6. Watch **API request limits** — heavy flows can require Power Platform Requests add-ons.
7. Build a **cost model spreadsheet** early and review it at each design decision, because architecture choices drive cost.

**How I would explain it:** "Every design decision is a cost decision. A plugin instead of a flow, or a portal instead of licensed users, can change the annual bill dramatically."

---

### Q95. Adoption is poor — only 30% of users log in. What do you do?
**Answer:**
I treat this as a change management problem, not a technical one.

Steps:
1. **Measure and segment.** Who is using it and who is not? Use the CoE Starter Kit, Power BI usage reports, and Microsoft 365 usage analytics. Often one team is at 90% and another at 5% — that tells you a lot.
2. **Talk to the non-users.** Do not send a survey — sit with them. The reasons are usually: it is slower than what I did before, I was not trained, it does not fit how I work, or my manager does not use it.
3. **Fix the top three friction points.** Often it is form load speed, too many mandatory fields, or too many clicks.
4. **Show them what is in it for them** — not "management needs the data" but "you stop re-typing this, and your pipeline report writes itself."
5. **Leadership behaviour matters most.** If managers run their team meeting from the CRM dashboard and refuse deals that are not in the system, adoption follows within weeks.
6. **Champions network** — one respected super-user per team beats any amount of formal training.
7. **Quick wins** — deliver one visible improvement they asked for, fast, so they believe feedback is heard.
8. **Retire the old system.** If the spreadsheet still works, people will use the spreadsheet.

**Result I would quote:** "We went from 30% to 85% weekly active users in two months, mostly by removing 9 mandatory fields and getting the sales director to run pipeline reviews from the dashboard."

---

### Q96. A department wants their own environment "so IT doesn't slow us down." How do you respond?
**Answer:**
I would say yes, with structure — refusing just creates shadow IT.

My approach:
1. **Understand the real need.** Usually they want speed of experimentation, not ownership of production.
2. **Give them a sandbox.** Provide a **development/experimentation environment** where they can build freely, with a DLP policy applied and no production data (or masked data).
3. **Define the promotion path.** If something becomes business critical, IT adopts it: solution, source control, pipeline, support, and monitoring. Make this path easy and fast, not a 6-week form.
4. **Set guardrails, not gates:** DLP policies, environment creation restricted to admins, **Managed Environments** with sharing limits and solution checker enforcement, and the **CoE Starter Kit** for visibility.
5. **Agree what "supported" means.** Be explicit: apps in the sandbox have no SLA, no backup guarantee, no support. Many teams then choose the governed route themselves.
6. **Track cost** — environments consume capacity, so assign the cost to their budget. Ownership of cost changes behaviour fast.

**How I frame it to IT:** "Our job is not to be the bottleneck. It is to make the safe path the fastest path."

---

# Section 12 — Project, Stakeholder & Behavioural (Q97–Q100)

### Q97. The client asks for a feature that you know is a bad idea. What do you do?
**Answer:**
I do not simply refuse, and I do not simply comply.

My approach:
1. **Understand the real need.** Ask "what problem are you solving?" and "what happens if we do not do this?" Very often the requested solution is not the only way to solve their problem.
2. **Agree on the goal**, so we are on the same side.
3. **Explain the impact in their language** — not "that breaks the data model" but "that will add 15 seconds to every save and cost about X per year in extra licences, and it will make the month-end report unreliable."
4. **Offer an alternative** that meets the goal. Never say no without an option.
5. **Quantify** where I can: time, cost, risk, maintenance.
6. **If they still insist**, I document the decision, the risks I raised, and their acceptance — professionally, not defensively — and then I build it well. It is their business and their money.
7. **Revisit later** with data. Once the pain shows up in real numbers, the conversation changes.

**What I would say in an interview:** "My job is to make sure they are making an *informed* decision, not to win the argument."

---

### Q98. You are halfway through a project and realise the design will not scale. What do you do?
**Answer:**
Raise it immediately. A problem found halfway is far cheaper than one found at go-live.

Steps:
1. **Verify it properly** before raising the alarm — build a small proof of concept or load test so I bring evidence, not a feeling.
2. **Quantify the impact**: at what volume does it break, and when will we reach that volume? "It works at 50,000 records and we will hit that in 8 months" is a very different conversation from "it might be slow."
3. **Present options with trade-offs**, not just the problem:
   - Option A: fix now — cost, time, risk.
   - Option B: ship as is, fix in phase 2 — what breaks and when.
   - Option C: a partial mitigation that buys time.
4. **Recommend one** clearly, with my reasoning.
5. **Take it to the sponsor**, not just the project manager, because it is a budget and risk decision.
6. **Whatever is decided, document it** in the risk register with the agreed trigger point for revisiting.

**What I emphasise:** "The mistake is not the bad design — it is staying quiet about it. I would rather have a difficult conversation in month three than a failure in month nine."

---

### Q99. Tell me about a time you had to say no to a stakeholder. (How to structure the answer)
**Answer:**
Use **STAR** and keep it factual and calm. Here is a model answer:

**Situation:** "On a customer service implementation, two weeks before go-live the Head of Operations asked us to add a new automated escalation process that would email 200 customers a day."

**Task:** "I was the lead consultant. I had to protect the go-live date and the data quality, without damaging the relationship."

**Action:** "I did three things. First, I asked what outcome they wanted — it turned out they were worried about SLA breaches being missed, not about the emails themselves. Second, I showed the impact: the change touched the SLA engine, needed re-testing of every case type, and would push go-live by three weeks. Third, I offered an alternative — a daily SLA-risk dashboard and a Teams alert for supervisors, which we could build in two days without touching the SLA engine. I put both options in writing with cost and risk."

**Result:** "They chose the dashboard. We went live on time, and the escalation emails were added in phase two six weeks later, properly tested. The Head of Operations later said the dashboard was more useful than the emails would have been."

**Why this works:** it shows you said no to the *solution*, not to the *person*, and you protected the outcome they actually cared about.

---

### Q100. How do you keep your Dynamics 365 and Power Platform skills current?
**Answer:**
This platform changes every wave, so I treat learning as part of the job, not extra.

What I actually do:
1. **Read the Release Plans.** Microsoft publishes two release waves a year (Wave 1 in April, Wave 2 in October) with the full list of what is coming. I read the plan for my areas and flag anything that affects our roadmap.
2. **Enable early access** in a sandbox environment and test upcoming features before they hit production. This has saved projects more than once.
3. **Microsoft Learn** for structured learning and certifications — for example PL-200 (Functional Consultant), PL-400 (Developer), PL-600 (Solution Architect), MB-280 (Customer Experience Analyst).
4. **Follow the product teams and the community** — the Power Platform blogs, community forums, and the MVP community. XrmToolBox and the community tools evolve constantly.
5. **Build things.** I keep a personal developer environment (free through the Power Apps Developer Plan) and rebuild real scenarios in new ways.
6. **Learn adjacent skills** — Azure, Fabric, Copilot Studio, and AI features are now part of almost every solution, so I keep growing there.
7. **Teach.** Explaining something to a colleague is the fastest way to find the gaps in my own understanding.

**What I would close with:** "The biggest risk in this platform is building today what Microsoft is shipping next quarter. Keeping current is how you avoid wasted work."

---

# Quick Revision Sheet

**The 10 phrases that make you sound senior:**
1. "Let me understand the business problem before I propose a solution."
2. "What is the source of truth for this data?"
3. "Is this a security requirement or a reporting requirement?"
4. "That works today — will it still work at 10 times the volume?"
5. "Who will maintain this after we leave?"
6. "Let's start out-of-the-box and only customise where there's real value."
7. "I'd validate that on the server too, because forms can be bypassed."
8. "That's a design decision with a cost — here are the trade-offs."
9. "Let me show you the numbers before and after."
10. "I'd document that decision and the risks, and get sign-off."

**Answer structure to use every time (STAR-T):**
- **S**ituation — one sentence of context.
- **T**ask — what you were responsible for.
- **A**ction — the *steps you took*, in order. This is where the marks are.
- **R**esult — a number if you have one.
- **T**akeaway — what you would do differently or now do as standard.

**Five things to always mention when relevant:**
- Security and licensing impact
- Performance at scale
- Maintainability and who supports it
- ALM (how does this move between environments?)
- What you would monitor after go-live

**Good luck. Practise saying these out loud — the words matter as much as the knowledge.**
