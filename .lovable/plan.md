# Diagnostic Laboratory + Imaging Centre dashboard

A new clinic type, "Diagnostic Laboratory & Imaging Centre", that reuses everything the dental dashboard already has for finance, staff, marketing, website and shop, and adds a full lab, radiology and pharmacy workflow on top.

## What you get

### Lab
- **Lab Overview** — Tests Today, Pending, In Progress, Completed, median turnaround time with a count of jobs breaching the 24h target, recent activity feed, scientist workload table (open vs completed per scientist) and a bar chart of most requested tests.
- **Test Forms** — one searchable, date-filtered table of every form with status tabs and counts, CSV export, print, and open. "Pending" and "Completed" are the same page opened pre-filtered.
- **New Test Form** — auto serial `MV-YYYY-00001`, patient details (linked to an existing patient record or created on the fly), referring doctor/institution, specimen, billing party (Patient / Clinic / Company), clinical notes, and test picking grouped by category. Categories and tests come from the database; if empty, the built-in printed menu is used (Haematology, Chemical Pathology, Medical Microbiology, Urinalysis, Microscopy/C&S, Semen Analysis, Serology — Widal/HIV/Hep/VDRL, Hormones & Fertility, Viral Load).
- **Result Entry** — opens a form by serial, shows the right result fields per test (analyte, unit, reference range) from a test-template file, flags out-of-range values, auto-saves a local draft and restores it if newer than the saved version, Save Draft (Pending to Processing), Mark Completed, print, and a live preview of the report. An admin panel adds Approve (locks the form), Reopen with a reason and Delete with a reason — every one of those is written to an append-only audit log.
- **Results Search** — search completed/processing results across serial, patient name or test, with a preview of key values.
- **Scientists** — senior staff only; creates real login accounts through a secure server function and edits role/status.
- **Manage Tests** — lab admins manage categories and tests (name, unit, reference range, input type, options, order, active), plus price so tests can be invoiced.
- **Lab Settings** — senior-only preferences (SLA hours, report header/footer, approval required).

### Imaging (Scan) dashboard
- Overview stats; Register (patient plus scan intake with auto MRN and scan serial); Patients with search and per-patient scan history; Scans with modality, body part, urgent flag, status, findings/impression/recommendation, reported-by then approved-by workflow, and image upload to private storage served via signed links; Appointments with status; and an activity log of every scan action.

### Pharmacy
- Drug catalogue with stock, batch and expiry; dispensing against a patient (and against a lab/scan visit); low-stock and expiring-soon alerts; dispensed items flow into invoices.

### Retail arm
- The existing Shop management and public shop pages carry over unchanged for the retail side.

### Patient-facing
- A public `/results` page: enter a serial, no login, see and print or download a branded PDF of the report. Scans use the same page with their own serial. Only safe fields are exposed, through a locked-down database function.

### Money
- Test forms, scans and pharmacy dispenses each create invoice lines against the existing invoicing, so billing, payments and revenue reports keep working exactly as they do today.

## Lifecycle

```text
Booking -> Test Form (Pending) -> Result Entry (Processing, drafts)
        -> Completed -> Approved (locked) -> Patient PDF via serial
        -> Reopen/Delete by lab admin -> audit log
```

## Technical notes

- New clinic type value `diagnostic` added to the `clinic_type` enum, a `diagnostic` entry in `clinicTypeConfig` with its own nav groups (Lab, Imaging, Pharmacy, plus the unchanged Patient Care / Finance / Marketing / Reports / Administration groups), page permissions added to `roleAccess.ts`, and the type marked available in the clinic-type picker.
- New tables (all org-scoped, RLS via `has_org_access`, explicit grants): `test_categories`, `lab_tests`, `test_forms`, `test_form_items`, `test_results`, `result_audit_log` (append-only, no update/delete policies), `lab_scientists` view over `staff`/`org_members`, `scan_patients`, `scans`, `scan_images`, `scan_appointments`, `scan_activity_log`, `pharmacy_drugs`, `pharmacy_dispenses`, `pharmacy_dispense_items`, `lab_settings`. Serial generation via a `generate_lab_serial(org_id)` database function using a per-org yearly counter.
- Security-definer RPCs `get_public_result(serial)` and `get_public_scan(serial)` returning only safe columns for approved/completed records, granted to `anon`.
- Edge function `manage-scientist` (service-role, JWT validated in code, senior-role checked) to create/update lab user accounts, mirroring the existing `manage-staff-user`.
- Private storage bucket `scan-images` with signed URLs.
- Realtime enabled on `test_forms` and `scans` so tables and the notification bell update live; subscriptions inside `useEffect` with cleanup.
- Test field templates live in `src/config/lab/testTemplates.ts` (analyte, unit, reference range, input type per test) with the printed menu as fallback data.
- New hooks under `src/hooks/lab/*` and `src/hooks/scan/*` following the existing React Query pattern; pages under `src/pages/dashboard/lab/*`, `src/pages/dashboard/scan/*`, `src/pages/dashboard/pharmacy/*`; public page `src/pages/PublicResultsPage.tsx` at `/results`.

## Build order

1. Database migration (tables, enum value, RLS, grants, serial function, public RPCs, realtime, storage bucket) and seeding of the default test menu.
2. Clinic type config, nav, role access, routes.
3. Lab: overview, forms list, new form, result entry with templates, results search, manage tests, scientists, settings.
4. Imaging: overview, register, patients, scans, appointments, activity log.
5. Pharmacy: catalogue and dispensing.
6. Public results page and PDF, plus invoice linking from forms, scans and dispenses.
