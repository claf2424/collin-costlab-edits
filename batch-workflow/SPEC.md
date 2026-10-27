# Batch workflow: Start Batch to compounding record

A clickable prototype of a new production workflow inside the **Batches** tab. It takes a technician from a scheduled batch through every step of compounding to a locked batch, logging who did each action and when, and ends with a compounding record that exports to PDF.

- `prototype.html` is the full clickable app with sample data. Open it in a browser. It saves to the browser's localStorage; use **Reset demo data** (sidebar footer) to start over.
- `compounding-record-example.html` is a fully completed sterile batch (lot TC-260930-01) rendered in the compounding record layout, including both appendices. Treat it as the target for the PDF export.

The prototype is a single file that mirrors the real app's screens, Tailwind classes and costing math (`lib/costing.ts`). It is a spec to implement against `grantajensen/costlab`, not code to merge. Existing costing, snapshot and completion logic stays; this reorganizes it into a guided flow.

**Good batches to try in the prototype**

| Lot | State | Shows |
| --- | --- | --- |
| KL-261023-01 | Scheduled | Start Batch from the beginning (non-sterile cream) |
| TC-261021-01 | In progress, paused in Bottling | Bottling, QA, Labeling (sterile vial) |
| MC-261022-01 | In progress, paused in Mixing | Mixing timers |
| MC-261016-01 | Quarantine | Attaching lab paperwork and entering results |

Use **Acting as** on the workflow screen to switch users. Pharmacist verification PIN in the demo is `1234`.

---

## 1. Entry point

On the batch detail page (`src/app/(app)/batches/[id]/page.tsx`, `src/components/batch-status-actions.tsx`):

- **Mark in progress** becomes **Start Batch**. It sets status `in_progress`, sets `manufactured_date` to today if blank, starts the batch clock, starts step 1, logs the event, and routes to the workflow screen.
- An in-progress or quarantined batch shows **Open workflow** instead.
- **Mark complete** moves off the batch page and onto the workflow screen (see 4).
- The batch detail page's standalone Actual yield form is replaced by a read-only summary once a batch has a workflow, because yield is entered in Bottling.

## 2. Status model

`draft` → `scheduled` → `in_progress` → `quarantined` → `in_progress` → `complete`
plus `failed` and `cancelled`.

- `quarantined` already exists in the `batch_status` enum (migration 0001). The prototype labels it "Quarantine".
- Add `failed` to the enum.
- Entering quarantine: when the tech clicks **Mark samples sent** in QA (after tracking is recorded). Records who and the send time, shows as **Quarantined - out for testing**, and pauses the clock. Creating a shipping label or entering tracking alone does not change status.
- Leaving quarantine: when QA is marked complete with every lab result passing. Status returns to `in_progress`.
- `failed`: a pharmacist rejects the batch after any lab test fails. Closes the batch like cancel; record and log are kept.
- `cancelled`: Cancel on the workflow screen, reason required.

## 3. Batch clock

Total processing time is the sum of active segments. Paused time is never counted and the clock never resets.

- Start / Pause / Resume button and a live `HH:MM:SS` clock on the workflow screen.
- **Auto-pause** (keeps time) when the user navigates away from the workflow screen, switches browser tab, or closes the window. The lab requisition page counts as part of the workflow and does not pause.
- **Auto-resume** when anyone starts a section or does any work inside one (takes a reading, checks a step, records a value). Expanding or collapsing a tile does not count.
- Pausing also stops any running mixing timers.
- Quarantine pauses the clock.
- In the real app, segments must be written server side (start/stop timestamps), not derived from the browser, so a closed tab still has a correct end. Suggest closing an open segment on `visibilitychange`/`pagehide` with `navigator.sendBeacon`, plus a server-side sweep for segments left open past a threshold.

## 4. Workflow screen

New route, for example `src/app/(app)/batches/[id]/run/page.tsx`.

- Header: status, sterility, lot, formula, size, client, room and hood, **Acting as** (demo only; real app uses the signed-in user).
- Clock bar: Start/Pause, clock, who started, pause count, **Compounding record**, **Cancel**, **Complete batch**.
- Progress bar: "N of 5 sections done" and the outstanding requirements.
- Five section tiles in order. Active tile is amber, awaiting verification is violet, done tiles collapse to a green bar with a check on the right and show who completed it, when, and how long it took.
- Activity panel: every event newest first, with per-user action counts.
- **Complete batch** requires all five sections done plus the existing completion rules in `lib/batch-rules.ts` (sterility classified, actual yield recorded, sterile 250 unit cap). It freezes the cost snapshot exactly as today, stops the clock, and opens the compounding record.

### Step 1: Ingredients + Measuring

- Rows are the batch recipe lines whose component type is Ingredients (packaging and lab consumables excluded). These already come from the MFR via `batch_recipe_lines` (migration 0013).
- Columns: Ingredient (API badge, manufacturer, weigh or measure-by-volume) · Target with ± tolerance range · Scale/device picker plus Scan · Actual measured · Lock.
- Weight lines: actual comes from the connected scale (**Read scale**), rounded to the device's readability. Volume lines (mL, L): tech types the measured volume.
- Lock (check mark) is enabled only when a device is selected and the reading is in tolerance. Out-of-tolerance readings show red and cannot be locked. A locked row stores the tolerance in effect at the time.
- Tech can clear a locked row and re-measure until review.
- **Mark ready for review** when every row is locked.
- Pharmacist verification: pharmacist name plus PIN. Only `pharmacist` role. The verifier cannot be anyone who measured on that batch. Pharmacist can **Return for re-measure**.
- On verification the tile collapses green and Mixing opens and starts.
- Tolerance defaults in the prototype: API ±2%, other ingredients ±5% (Settings → Pharmacy). These are placeholders; the pharmacy sets real values.

### Step 2: Mixing

- One line per mixing instruction, copied from the MFR when the batch is created (same pattern as recipe lines).
- Each line: number, instruction, equipment/settings detail, optional timer tag (Mix, Heat, Rest with minutes).
- Timed steps have Start / Stop / Reset, a progress bar and a "time reached" alert. A timed step can only be checked off once its timer reaches the set time and is stopped.
- Checking a step logs user, time, and actual timed duration. Steps can be unchecked while active.
- **Mark Mixing complete** when every step is checked.

### Step 3: Bottling

- **Packaging required**: batch recipe lines of type Packaging (labels excluded, they belong to Labeling), with manufacturer, SKU and expected quantity.
- **Filled with** (required): the filling machine or method, from the pharmacy's filling equipment list (for example Ohaus Repeater Pump #1, Manually filled (syringe)).
- **In-process fill checks** (optional, + Add check, unlimited): Verified on (scale or measuring device from the devices list), measured mL or g, note. Target is the formula's fill size ± fill tolerance (default ±3%, Settings → Pharmacy). Out-of-tolerance checks are saved, flagged red, and require a note. Capsule formulas (fill in `ea`) record a weight with no target.
- **Yield** (required): Finished packages and Sent for testing/QA. Shows Expected (batch size ÷ fill size) and Dispensable (finished minus testing/QA) with variance. Testing/QA cannot exceed finished; sterile 250 cap applies.
- Recording yield sets `batches.actual_yield` = finished packages (cost per unit uses total made, not dispensable). Confirm with Collin if dispensable is preferred.

### Step 4: QA

**Visual inspection** (from the MFR). Each item Pass or Fail; Fail requires a note; all must pass. Sterile default items:
- All product free from particulates and precipitates
- All product appears {appearance} in color (filled from the formula's Expected appearance field)
- Fill level consistent across products

**Third-party testing** (from the MFR's Testing list; sterile defaults Endotoxin, Potency, Sterility):
1. Pick the lab, choose tests, set samples per test. Total is compared with units pulled for testing/QA in Bottling.
2. **Create lab submission** (ID `LS-{lot}`) → **8.5×11 requisition**: pharmacy info and contact, lab info and client account, lot, product, form, sterility, compounded date, BUD, batch size, units in batch, tests with samples and container, total samples enclosed, storage line, chain-of-custody signature lines, carrier and tracking.
3. **Create shipping label** (service, weight, cold-pack flag) → 4×6 label and tracking, or enter carrier and tracking manually.
4. **Mark samples sent** when the package leaves. Stores `sent_at` and `sent_by`, sets status to quarantined (label "Quarantined - out for testing"), pauses the clock. The quarantine list's days waiting and the record's "Out for testing" duration count from this time. Lab results can only be entered after this.
5. **Sample labels** (any time during QA): a FOR TESTING ONLY version of the step 5 label, so the units going to the lab are labeled before shipping. Red "FOR TESTING ONLY · DO NOT DISPENSE" banner and footer, product, fill size and form, storage line, lot with barcode, MFD, BUD, lab name and submission ID; no client name. Number of labels prefills with the total samples on the lab submission (or the units pulled for testing/QA in Bottling before a submission exists). Printer picker. Each print job is logged (later jobs marked Reprint) and shown on the compounding record under QA. Dispensing labels in step 5 stay locked until results pass.
6. **Lab results** (after samples are marked sent): upload the lab's paperwork (PDF or image, multiple files). Then each test is marked Pass or Fail; marking requires at least one file attached.
7. Any Fail fails third-party testing: QA cannot complete and a **Reject batch** action closes the batch as `failed`.
- **Mark QA complete** requires: all visual items passed, tracking recorded, samples marked sent, paperwork attached, every test Pass. Completing QA releases quarantine.

### Step 5: Labeling

- Locked until QA is complete with passing results. Labels cannot be printed before then.
- **Labeled with** (required): applicator or "Manually labelled", from the pharmacy's labeling equipment list.
- **Sample label** generated from batch and MFR: pharmacy name, address and phone, product, fill size and form, client, directions/storage line (new MFR field), lot with barcode, MFD, BUD, Sterile and Rx only flags. Layout is a starting template; Collin to confirm required label elements.
- **Print**: number of labels prefilled with finished packages, printer picker (sterile defaults to cleanroom printer). Each print job is logged; later jobs are marked Reprint. Warns if fewer labels than packages.
- **Mark Labeling complete** requires equipment selected and at least one print job.

## 5. Activity log and productivity

Every action writes an event: batch started/paused/resumed (with reason), step started/completed/reopened, each measurement, review requested, verified, returned, mixing step done/undone, timer start/stop/reset, fill equipment set, fill checks added/removed, yield recorded/edited, visual pass/fail, submission created/edited, requisition printed, sample labels printed, shipping label, tracking, samples sent, quarantined, documents attached/removed, lab result pass/fail/cleared, released, labeling equipment set, labels printed/reprinted, completed, cancelled, rejected.

Each event stores batch, step, user, timestamp, type, human-readable label, and a JSON payload. This is the basis for productivity per user (actions per person, time per section, sections completed per person). The existing `audit_events` table can keep its org-level role; a dedicated `batch_events` table is easier to query per batch.

## 6. Master formula additions

On the formula page (`src/app/(app)/formulas/[id]/page.tsx`):
- **Mixing steps** section: add, reorder, remove. Each step: instruction, optional timer type (mix, heat, rest) with minutes, equipment/settings text. Copied to new batches; existing batches keep their own.
- **QA release checks** section: Expected appearance/color text, visual inspection items (text may contain `{appearance}`).
- **Label directions and storage** text field (Dispensing section).
- Third-party tests continue to come from the existing formula Testing list.

## 7. Settings additions

- Pharmacy: measuring tolerances (API %, other %, fill %).
- Locations: Scales and measuring devices (asset tag, name, weight or volume, readability, last calibrated), Filling equipment list, Labeling equipment list.
- Labs (prototype has two sample labs: name, address, phone, client account). Needs a management screen.
- Label printers.

## 8. Compounding record (PDF)

Layout in `compounding-record-example.html`, also on the Compounding record page in the prototype:
header (pharmacy, product, lot, status) · batch facts · production steps with who, when and duration, total processing time · ingredients measured with tolerance, % deviation, device, measured by, pharmacist verification · mixing with set and actual times · bottling (filled with, expected, finished, testing/QA, dispensable) · in-process fill checks · QA (visual inspection, lab submission, lab results, overall, paperwork list) · labeling (labeled with, label image, print log) · components, labor, testing and costs · full activity log · signature lines (compounded by = who completed Measuring, verified by = who completed QA) · **Appendix A** lab requisition · **Appendix B** shipping label copy.

## 9. Suggested data model

Names are suggestions; follow the conventions in `supabase/migrations` (organization_id on every table, RLS via `has_org_role`).

```sql
alter type public.batch_status add value if not exists 'failed';

-- MFR
create table formula_mix_steps (id, organization_id, formula_id, position int, instruction text,
  timer_kind text check (timer_kind in ('mix','heat','rest')), timer_minutes int, detail text);
create table formula_visual_checks (id, organization_id, formula_id, position int, text text);
alter table formulas add column appearance text, add column label_text text;

-- Settings
alter table organizations add column tol_api_pct numeric, add column tol_other_pct numeric, add column tol_fill_pct numeric;
create table measuring_devices (id, organization_id, tag text, name text, kind text check (kind in ('weight','volume')),
  readability_decimals int, last_calibrated_on date, active bool);
create table equipment (id, organization_id, kind text check (kind in ('filling','labeling')), name text, active bool);
create table labs (id, organization_id, name, address, city_state_zip, phone, client_account);
create table label_printers (id, organization_id, name);

-- Run
create table batch_time_segments (id, organization_id, batch_id, started_at timestamptz, ended_at timestamptz,
  started_by uuid, ended_by uuid, end_reason text);
create table batch_steps (id, organization_id, batch_id, step text check (step in ('measure','mix','fill','qa','label')),
  status text check (status in ('todo','active','review','done')), started_by, started_at, review_by, review_at,
  done_by, done_at, verified_by);
create table batch_measurements (id, organization_id, batch_id, recipe_line_id, device_id, target numeric,
  tolerance_pct numeric, actual numeric, locked_by, locked_at);
create table batch_mix_steps (id, organization_id, batch_id, position, instruction, timer_kind, timer_minutes, detail,
  timer_accum_ms bigint, timer_started_at timestamptz, done_by, done_at);
create table batch_fill (batch_id primary key, organization_id, equipment_id, finished int, qa_pulled int, recorded_by, recorded_at);
create table batch_fill_checks (id, organization_id, batch_id, device_id, value numeric, in_tolerance bool, note, by, at);
create table batch_visual_results (id, organization_id, batch_id, text, ok bool, note, by, at);
create table lab_submissions (id, organization_id, batch_id, code text, lab_id, created_by, created_at,
  carrier text, tracking text, label_service text, label_weight numeric, cold_pack bool, shipped_by, shipped_at,
  sent_by uuid, sent_at timestamptz);
create table lab_submission_tests (id, organization_id, submission_id, lab_test_id, samples int,
  result text check (result in ('pass','fail')), result_by, result_at);
create table lab_documents (id, organization_id, batch_id, storage_path text, file_name, size_bytes, content_type, uploaded_by, uploaded_at);
create table label_print_jobs (id, organization_id, batch_id, kind text check (kind in ('dispensing','testing')),
  printer_id, copies int, reprint bool, by, at);
alter table batches add column label_equipment_id uuid, add column quarantined_at timestamptz, add column released_at timestamptz,
  add column cancel_reason text;
create table batch_events (id bigserial, organization_id, batch_id, step text, actor_user_id uuid, type text,
  label text, details jsonb, created_at timestamptz);
```

`lab_documents` needs a Supabase Storage bucket with org-scoped policies; the app does not use Storage yet.

## 10. Simulated in the prototype (needs real integration)

- Connected scale readings (Read scale) and barcode scan of device asset tags.
- Label printing (Zebra/DYMO) and requisition printing.
- Shipping label purchase and tracking (FedEx/UPS account).
- PDF export of the compounding record.
- File storage for lab paperwork (prototype keeps file names; images under 2 MB preview).
- Pharmacist PIN (real app: re-authentication or a per-user signing PIN).

## 11. Open items for Collin

- Real tolerance values (measuring and fill).
- Real labs, accounts and samples required per test.
- Required label elements (patient name, Rx number, warnings).
- Actual yield: total finished packages (current) or dispensable units for cost per unit.
- Whether the "All" filter count on the Batches tab should exclude completed batches.

## 12. Batches tab

- Search bar (lot #, formula or client), filters as you type.
- Status filter chips with counts (no count on Complete): All, Draft, Scheduled, In progress, Quarantine, Complete, Failed, Cancelled. Only statuses with batches show.
- When a search or filter is active, only the matching batch records show, with a match count and Clear.
- Status badge text for quarantined batches: "Quarantined - out for testing"; filter chip "Quarantined".
- With no filter: a **Quarantined - out for testing** table (lot, formula, lab, tests, sent date/time and tracking, days waiting, results in, files attached, Enter results), then My batches, then Batch records.
