# GME InvoiceApp — Handoff Document

**⚠ Read §11 first if you're picking this up after the Cloud Run migration.** Sections 1-10 below describe the *original* Google Apps Script system and predate that migration — still accurate as history and as the live fallback system, but the app's actual production direction is now the Cloud Run backend described in §11.

**Scope of this project:** `F:\Claude Code` is now **two separate git repositories**, not the git-free single folder this document originally described: the root folder (this file's location) is a public repo for the GitHub Pages frontend (`index.html`/`super-admin.html`); `backend/` inside it is a **separate, private** repo (`.gitignore`'d out of the root repo entirely) for the Cloud Run backend, pushed to `https://github.com/gulmohammedansari/gme-invoiceapp-backend`. Two different remotes, two different histories — see §11 for why they're split and what that means for making changes.

**Test tenant used throughout development:** "Diners Fire Engineers" (`T-GME-DINERS` in the new Firestore schema)

**See also:** [SECURITY_AUDIT.md](SECURITY_AUDIT.md) — a full audit; all 7 findings now have fixes applied (XSS escaping across the whole app including `super-admin.html`, login moved off GET, real server-side logout, upgraded password hashing with transparent migration, generic pre-auth error messages, an activated-but-still-public `LOGIN_SECRET`/`API_SECRET` pair, and `Role` enforcement on the two genuinely admin-level actions). Two things still need action from you, not from code: (1) set the `API_SECRET` Script Property to match `LOGIN_SECRET` in index.html — see Finding 4's note for the exact value and why it's a very weak control regardless; (2) the XSS fix was done via careful manual reading, not an automated scan — treat it as thoroughly-checked, not provably exhaustive. Read that file before touching auth/session/login code or adding any new place that renders sheet-sourced text.

## 1. What this app is

A multi-tenant Google Apps Script + HTML invoicing app for small fire-safety/engineering businesses. Each tenant gets their own login, their own business-data spreadsheet, and their own branding (logo/signature/stamp/watermark). A separate control spreadsheet holds tenant/user/session records shared across all tenants.

## 2. File map

| File | Role |
|---|---|
| [Gscript.txt](Gscript.txt) | The entire Google Apps Script backend (paste into the Apps Script editor as `Code.gs`). Single-file `doGet`/`doPost` router. |
| [index.html](index.html) | The entire frontend SPA — served as the Apps Script web app's HTML output. All tenant-facing pages (Invoices, Quotations, Challans, Credit/Debit Notes, AMC, Certificates, Settings) live in this one file. |
| [super-admin.html](super-admin.html) | Standalone portal (separate URL/deployment target) for managing tenants and their users. Not tenant-facing. |
| `HANDOFF.md` (this file) | You are here. |

**No build step.** These are plain files — edit them directly, then paste/redeploy into Apps Script.

## 3. Architecture essentials

- **Multi-tenancy:** one control spreadsheet has `Tenants`, `Users`, `Sessions` sheets. Each tenant additionally has their *own* Google Sheet (its ID stored in the tenant's `Spreadsheet ID` column) holding their actual business data (Invoices, Quotations, Challans, AMC, Certificates, etc.).
- **Per-request tenant context:** `applyTenantContext(session)` (Gscript.txt) sets a global `ACTIVE_SPREADSHEET_ID` and replaces the global `COMPANY` object from the session's cached snapshot, on every request. `getSheet(name)` then resolves to the right spreadsheet automatically.
- **Sessions are sheet rows, not CacheService** (CacheService proved unreliable — could fail to persist for an immediate read-back). `validateSession(token)` re-reads fresh from the sheet every request; sliding expiration is applied on every valid use.
- **Session-cache staleness is a recurring theme.** `COMPANY` is a snapshot cached in the session at login time. Any settings change that should take effect immediately must call `updateSessionCompany(token, COMPANY)` — this is already wired for `handleSaveCompanySettings`, but keep it in mind if you add more editable company fields.
- **`ensureHeaders(sheet, expectedHeaders)`** — the standard pattern for schema migrations. Non-destructive: appends any missing header as a new trailing column, never touches existing data. Used for `INVOICE_HEADERS`, `TENANT_HEADERS`, `AMC_HEADERS`, `CERTIFICATE_HEADERS`. **Always read/write sheet rows by header name** (`headers.forEach((h,j)=>{obj[h]=row[j]})`), never by fixed column index — positional arrays drifting out of sync with an evolving header list has caused real bugs this session.
- **No PDF library exists in Apps Script.** The only way to produce a PDF is: write a self-contained HTML string to a temp Drive file, call `.getAs('application/pdf')` on it, then trash the temp file. This conversion is done by Google Drive's own HTML-to-PDF converter, which has real, confirmed quirks (see §6).

## 4. Features implemented this session

### AMC Contracts
- Image + GPS location capture per contract.
- Google Calendar reminder auto-created/updated on save, landing **on the exact expiry date** (not 30 days before) — with a separate 30-day-before **popup notification** on that same event. Manual "Add to Calendar" / "Sync now" button available too.
- Distinct status reporting (`skipped-no-calendar`, `skipped-no-expiry`, `skipped-invalid-date`, `created`, `updated`) so silent failures are distinguishable from "no calendar configured."

### Invoices — WhatsApp & Email sending
- "WhatsApp" and "Email" buttons send the *exact same-looking* PDF as Print/Preview (see §5 — this took several iterations).
- Backend actions: `sendInvoiceToBuyer` (email, always regenerates fresh), `getInvoiceWhatsappLink` (returns a wa.me link with a Drive-shared PDF link, **always regenerates fresh — do not reintroduce caching here**, see §7).
- `findInvoiceRow` / `buildInvoicePayloadFromRow` reconstruct the full print payload straight from the sheet row (including `subTotal`/CGST/SGST/IGST *amounts*, not just rates — the print engine trusts stored figures, never recomputes them).

### GST Type — "No GST" option (Invoice + Credit/Debit Note)
- The existing `CGST_SGST` / `IGST` doc-level dropdown (`inv-gst-type`, `cn-gst-type`) gained a third option, `NO_GST`, for tax-exempt bills.
- `getComputedTotals()`/`getCNComputedTotals()` now also return `cgstRate`/`sgstRate`/`igstRate` (0 for `NO_GST`) alongside the amounts — every save/print/preview call site was switched to read rates from that same object instead of `invState.taxes.*`/`cnState.taxes.*` directly, specifically so a No-GST bill never saves/prints a stale "@ 9%" label next to a ₹0.00 amount. `invState.taxes`/`cnState.taxes` themselves are left untouched, so switching back to CGST+SGST or IGST later restores the originally-configured rates.
- Each line item's own `gstPct` is also forced to 0 when saving under `NO_GST` — otherwise the itemized HSN-wise tax table would still show each item's originally-configured per-item rate even though the doc-level total is ₹0.00.
- `totalsBlock()` (both the client `LedgerPrint` and server `LedgerPrintGas` copies) skips the CGST/SGST/IGST row entirely when all three amounts are zero, instead of printing a misleading "IGST: ₹0.00" line. **Deliberately did NOT** apply the same suppression to the itemized `taxTable()` — pagination's height math (`footerHeight`/`taxBlockHeight`) reserves space for that table based on the document's *kind* (`DOC.tax`) alone, not on whether the amounts happen to be zero, so hiding it at render time only would leave an unexplained blank gap. A No-GST invoice's itemized tax table still shows "0%, ₹0.00" per row — correct data, just not maximally polished. Fixing that properly would mean threading the zero-check into the pagination functions themselves; flagged here rather than risked without live testing.
- Quotations don't have this toggle at all (no `qt-gst-type` exists) — out of scope, not touched.

### Certificates — full feature, added from scratch this session
- `CERTIFICATE_HEADERS` (+ `Certificate PDF URL` column), backend `buildCertificateHtml`/`renderCertificatePdf` (a Drive-safe, full-page port of the browser's `printCertificateHtml()`), `saveCertificatePdfToDrive`, `sendCertificateCopyToBuyer`.
- New actions: `sendCertificateToBuyer`, `getCertificateWhatsappLink` (also always-fresh, same reasoning as invoices).
- Frontend: WhatsApp/Email buttons in the certificates table, `sendCertificateWhatsapp`/`sendCertificateEmailToBuyer`.

### Branding — Logo / Signature / Stamp / Watermark (tenant-provisioned)
- Settings → Branding: upload Logo, Signature, Stamp. Signature/Stamp get automatic background removal (`chromaKeyToTransparentPng` — samples the 4 corner pixels as "background" color, alpha=0 for near-matches). Not applied to Logo (a logo's corners aren't necessarily background).
- **Stamp is auto-darkened 30%** at upload time (`darkenFactor` param on `chromaKeyToTransparentPng`) — stamp ink often photographs faded.
- **Watermark:** its own dedicated upload (Settings → Watermark card), deliberately separate from Logo — not auto-derived from it. Tenant also picks a style: **Tilted** (-22°) or **Straight** (0°), via `WATERMARK_ANGLES` in index.html. The raw uploaded image is kept as-is (`COMPANY.watermarkSrc` / Tenant sheet column `Watermark Image`) so switching styles later recomputes cleanly from the original photo rather than re-processing an already-rotated/faded copy. `makeWatermarkImage(img, angleDeg, opacity)` (renamed from `makeWatermarkFromLogo`) does the actual canvas rotate+fade (7% opacity, padded bounding box so corners aren't clipped), and the result is stored as `COMPANY.watermark` / Tenant sheet column `Watermark` — the ONE field every print engine actually reads. That indirection means the print/PDF code itself never changed when this was redesigned: `COMPANY.watermark` is still just a `background-image` behind every document (invoices, challans, quotations, credit/debit notes, certificates) in both Print and WhatsApp/Email PDFs, regardless of where that image came from.
- Stamp+Signature overlap: **blended to look like one mark** — signature drawn crossing the stamp. Browser engines use `position:absolute` + `transform:translate(-50%,-50%)` (safe in real browsers); Drive-rendered (server) engines use a `vertical-align:middle` + negative-margin technique instead (see §6 for why).
- Images >~a few KB as base64 blow past Sheets' 50,000-char cell limit — all branding images are uploaded to a Drive folder and only a short thumbnail-proxy URL (`https://drive.google.com/thumbnail?id=X&sz=w1000`) is stored. `maybeUploadBrandingImage(value, label)` does this uniformly for Logo/Signature/Stamp/Watermark.
- All darkening/rotation/fading is baked into the PNG's pixels **once, client-side, at upload time** — never a live CSS filter/transform at render time. This is deliberate (see §6) so the same processed asset renders identically in the browser and via Drive's PDF converter.

### Data Export to Excel (Cloud Run only)
- **Tenant export:** Settings → Export Data → "Export My Data to Excel" downloads all business data (Invoices, Buyers, Items, Quotations, Challans, Certificates, AMC Contracts, Credit/Debit Notes) as a single .xlsx file with 8 sheets, one per collection.
- **Super-admin export:** Two options: (1) "Export All Tenants" button (header) downloads a master list of all tenants + users as one .xlsx with 2 sheets; (2) per-tenant "Export" button (table row) downloads that specific tenant's business data (8 sheets).
- Backend: `backend/src/routes/exportData.js` handles `exportMyData` (session-scoped), `exportTenantData` (super-admin, specific tenant), `exportAllTenants` (super-admin, control data only). Uses `xlsx` library; data transmitted as base64 over JSON, no server-side file storage.
- Frontend: `exportMyData()` in index.html; `exportTenantData(tenantId)` + `exportAllTenantsData()` in super-admin.html. Auto-downloads via data URI.
- CLI alternative: `backend/scripts/export/exportToExcel.js` (see [FIRESTORE_EXPORT_COMMANDS.md](FIRESTORE_EXPORT_COMMANDS.md) for usage).

### Super Admin Portal ([super-admin.html](super-admin.html))
- Add/Edit/Delete tenants with full onboarding fields; per-tenant user management (add/edit/delete users, blocks deleting a tenant's last user).
- Export buttons: "Export All Tenants" (header) + per-tenant "Export" (table row) for data backup.
- Gated by a `SUPER_ADMIN_SECRET` Script Property (`isSuperAdminSecretValid`), **not** a session token — it operates outside any tenant's login. `listTenants`/`listTenantUsers` (GET) and `provisionTenant`/`updateTenant`/`deleteTenant`/`addTenantUser`/`updateTenantUser`/`deleteTenantUser` (POST) must stay gated **before** the normal session-token check in the router (this was a real bug once — moving it after the session check always failed with "session expired").

### Dev login bypass
- `DEV_SKIP_PASSWORD_CHECK` (top of Gscript.txt, currently **`false`** — re-enabled/production-safe right now). Flip to `true` only for local testing, and remember to flip it back.

### The big one: server-side print engine port (`LedgerPrintGas`)
- The browser's `LedgerPrint` module (invoice/challan/quotation/credit-debit-note print engine) was **ported line-by-line into Apps Script** as `LedgerPrintGas`, so WhatsApp/Email PDFs are generated by (almost) the same code as Print/Preview, instead of a hand-maintained parallel implementation that kept drifting out of visual sync.
- Wired into `saveInvoicePdfToDrive`, `sendInvoiceCopyToBuyer`, `sendInvoiceEmail` via a shared `renderLedgerPrintPdf(data, COMPANY, kind, fileName)`.
- **The OLD renderer is still in the file, unused** (`buildInvoiceHtml`, `computeInvoiceLineCalc`, `sellerHtml`, `partiesHtml`, `bankBandHtml`, `sigStampCellHtml`, `totalsHtml`, `itemsTableHtml`, `INV_GEO`, `INV_AFM`) — deliberately kept as a rollback fallback since there's no version control. **Do not delete it** until the new engine has been confirmed fully correct in production for a while.

## 5. Bugs found & fixed this session (useful context, don't reintroduce)

- **Numeric-type coercion crashes** — Sheets can return `Invoice No`/`Buyer Phone`/etc. as JS `Number`, breaking `.replace()`/`.trim()`. Always `String(x || '')` before string methods on sheet-sourced values.
- **Double-print-dialog bug** — an `iframe.onload` handler could fire twice (once for the initial blank frame, once after `document.write()`). Fixed everywhere by calling the print logic **synchronously right after `iDoc.write();iDoc.close();`**, waiting only for images via `waitForImagesToLoad(doc, maxWaitMs)` (Drive-hosted signature/stamp/logo/watermark URLs need a real network fetch) — never rely on `onload`.
- **Signature/stamp cut off "Authorised signatory" text** — the bank/sign band has a fixed height budget (`GEO.bankH` = 22mm) shared with two text lines; oversized sig/stamp images (16mm/12mm) overflowed it, and since ancestors use `overflow:hidden`, the text silently got clipped instead of the box growing. Fixed by shrinking to sizes that actually fit the budget (11mm/9mm overlapping, 12mm single). **If you resize these images again, do the arithmetic against `GEO.bankH` first.**
- **Certificate PDF not using the full page** — first attempt at the server-side certificate renderer used content-sized (auto) height instead of the full A4 flex-column layout the browser version uses, so the border box only wrapped its content instead of stretching to the page. Fixed by mirroring the browser's full-page flex layout exactly.
- **WhatsApp PDF caching bug (important, see §7 for the fix)** — `handleGetInvoiceWhatsappLink`/`handleGetCertificateWhatsappLink` used to reuse whatever was already in the `Invoice/Certificate PDF URL` column and only regenerate when empty. This meant ANY later fix (branding, layout, watermark, etc.) would never show up on WhatsApp for an already-tested invoice/certificate — only Email (which always regenerated fresh) would reflect it. **Both handlers now always regenerate fresh on every call.**
- **Router-ordering bug** — super-admin GET actions (`listTenants`, `listTenantUsers`) were briefly placed after the session-token check, so they always failed with "Session expired." They must be gated by the super-admin secret **before** any session check.

## 6. Google Drive's HTML-to-PDF converter — confirmed quirks

This converter is what turns Gscript-generated HTML into the WhatsApp/Email PDFs. It is **not** a full modern browser, and everything server-side is written around its limitations:

- **`position:absolute` + `transform` is unreliable** — confirmed via an actual stray full-page-height vertical line appearing when this was tried for the stamp/signature overlap. Server-side engines use `vertical-align:middle` + a negative margin instead. **Do not reach for `position:absolute`+`transform` in any Gscript.txt-side rendering code without testing carefully** — this is why the watermark feature bakes its rotation into the PNG's pixels via canvas instead of using a live CSS `transform`.
- Flexbox itself is now **confirmed to work fine** (the whole `LedgerPrintGas`/`.inv-page` layout and the certificate's full-page layout both use it successfully) — an older code comment claiming Drive can't do flexbox is outdated; only `position:absolute`+`transform` is the actual problem.
- Doesn't zero default `<img>` borders the way browsers do — always add `border:0` explicitly on server-rendered `<img>` tags.
- `width:100%` + `border` without `box-sizing:border-box` lets the border render outside the specified width, which can get clipped asymmetrically at a page edge (this caused a left/right border-thickness mismatch in the old renderer once).
- Percentage heights are unreliable when the ancestor chain's height isn't fully explicit — prefer fixed mm heights (the `GEO`/`INV_GEO` pattern already used throughout).

## 7. Known-good invariants — please don't regress these

1. **WhatsApp/Email PDF generation must always be fresh**, never read a cached URL column first. (Was a real bug, just fixed — see §5.)
2. **Money/totals are never recomputed at render time** — always read stored figures (`Grand Total`, `CGST Amount`, etc.) directly from the sheet row, never re-derive from line items in the print engines.
3. **Never use `position:absolute`+`transform` in Gscript.txt's HTML-generation code** for anything that needs to render via Drive's converter — bake any rotation/overlap effect into pixels client-side first, or use the negative-margin/vertical-align technique already established.
4. **All new sheet columns go through `ensureHeaders()`**, added to the relevant `*_HEADERS` constant, never a one-off `sheet.appendRow([...])` with a hand-typed list (multiple headers arrays already exist as the single source of truth — `TENANT_HEADERS`, `INVOICE_HEADERS`, `AMC_HEADERS`, `CERTIFICATE_HEADERS`).
5. **Any tenant-editable COMPANY field must be added in *four* places** to actually take effect immediately: the `COMPANY` default object (client AND server), `handleLogin`'s `company` object (built from the Tenants sheet row), `handleSaveCompanySettings` (upload/set-cell/merge-into-COMPANY), and `TENANT_HEADERS`. Missing any one of these is exactly how the "logo missing on WhatsApp" investigation started (that turned out to be the caching bug instead, but the four-places rule is still the thing to check first for "my new company field doesn't show up somewhere").
6. **No `Date.now()`/`Math.random()`/argless `new Date()` assumptions carry over between client and server** — timestamps are computed independently in each engine; this hasn't caused a bug yet but keep it in mind if you add anything date-sensitive to the shared print payloads.

## 8. Explicitly NOT implemented (discussed, not built)

- **Legally-binding PKI Digital Signature (DSC/eSign)** — discussed feasibility only, no code written. Conclusion: **not possible purely within Apps Script** (no PDF-signing library, no access to a physical USB DSC token or HSM). The only real path is integrating a licensed third-party eSign API (Digio, Leegality, NSDL/CDAC Aadhaar eSign, DocuSign, etc.) via `UrlFetchApp` — a materially bigger project (vendor onboarding, per-document cost, OTP-based signer flow, real API integration work), not a quick addition. Worth first confirming whether the tenant's actual documents (standard GST tax invoices) legally require this at all — normally they don't; a visual signature/stamp (already built) is standard and sufficient.

## 9. Deployment reminders (Apps Script, not this repo)

- There is **no CI/CD** — after editing Gscript.txt/index.html locally, you must paste the changes into the Apps Script editor and go **Deploy → Manage deployments → (pencil/edit icon) → Version: "New version" → Deploy**. Saving the script alone does **not** update the live `/exec` URL — this has caused confusion more than once this session (a user testing against a stale deployed version looks identical to a bug that "isn't fixed").
- Local syntax validation (since there's no real GAS test environment available here) has been done via Node: `node -e "const fs=require('fs'); new Function(fs.readFileSync('Gscript.txt','utf8')); console.log('OK')"`. This only catches syntax errors, not runtime/GAS-API issues — always ask the user to test against a real deployment.
- `super-admin.html` is a **separate** page/deployment target from `index.html` — don't assume changes to one automatically apply to the other's copy of any shared-looking code (e.g. `gasPost`/`gasGet` helpers are duplicated, not shared).

## 10. Suggested next steps for whoever picks this up (superseded by §11)

- Confirm the WhatsApp-PDF-caching fix (§5/§7) is deployed and that a previously-stale invoice/certificate now reflects current branding.
- Once the new `LedgerPrintGas` engine has been trusted in production for a while, delete the old unused renderer functions listed in §4 to reduce dead code (holding off only because there's no version control to fall back on).
- If a genuine legal-compliance need for DSC/eSign turns up, scope out a third-party eSign API integration as its own project (see §8) rather than trying to build it into Gscript.txt directly.

These items are all still individually true, but §11 below is the actually-current picture — the whole backend has since been ported off Apps Script.

---

## 11. The Cloud Run migration (current state, read this first)

Everything in §§1-10 describes the *original* system. Since then, the entire backend was ported from Google Apps Script + Sheets + Drive to **Node/Express on Cloud Run + Firestore + Cloud Storage**, following the 12-phase plan in [CLOUD_RUN_MIGRATION_PLAN.md](CLOUD_RUN_MIGRATION_PLAN.md) (gitignored, not in either repo — local reference only). `index.html`/`super-admin.html` did **not** change structurally for this — they still speak the exact same `{action, token, data}` single-endpoint contract, just to a new URL.

### Where everything actually lives now

| What | Where |
|---|---|
| New backend source | `backend/` — a **separate git repo** (its own `.gitignore`'d-out `.git`), pushed to the private `https://github.com/gulmohammedansari/gme-invoiceapp-backend` |
| Backend's own docs | `backend/README.md` (phase-by-phase implementation notes + verification), `backend/DEPLOYMENT.md` (the Cloud Run deployment runbook, written for Cloud Shell) |
| Full migration plan | `CLOUD_RUN_MIGRATION_PLAN.md` (root, gitignored) — phase table, Firestore schema, architecture reasoning |
| Frontend (unchanged repo) | `index.html`/`super-admin.html`, this root folder, public repo, GitHub Pages |
| Old Apps Script backend | `Gscript.txt` (root, gitignored) — **still the live fallback**, see "Cutover status" below |

**Why two repos, not one:** `backend/` was deliberately kept out of the public GitHub Pages repo (it has no secrets in it, but no reason to expose implementation details either) — see the root `.gitignore`. If you're editing backend code, `cd backend` first and treat it as its own repo (`git status` there shows backend-only changes; the root repo's `git status` will never show `backend/` at all).

### Current deployment (as of the last session)

- **GCP project:** `gme-invoiceapp-prod`, region `us-central1` (chosen specifically to land inside Cloud Run/Firestore/Storage's free tier — `asia-south1` would be lower-latency but not free; see `DEPLOYMENT.md`'s "Staying inside the free tier" section).
- **Cloud Run service:** `gme-invoiceapp-backend` → `https://gme-invoiceapp-backend-426403271855.us-central1.run.app`
- **Firestore:** Native mode, `us-central1`, real data migrated in (see below).
- **Storage bucket:** `gme-invoiceapp-prod-files` — **uniform bucket-level access must stay OFF** on this bucket (see "Critical bugs" below for why).
- **Runtime service account** (used by Cloud Run, needs sharing access to each tenant's Calendar for AMC reminders): `426403271855-compute@developer.gserviceaccount.com`
- **Secrets configured:** `SUPER_ADMIN_SECRET` only. `RESEND_API_KEY`/`EMAIL_FROM` deliberately not set yet (email deferred) — every email-sending action already fails soft with a clear "not configured" message rather than breaking anything.
- **`ALLOWED_ORIGIN` is currently `*`** (wide open) — this was intentionally loosened mid-Phase-12 to test the locally-edited frontend files (see "Cutover status"), which run from `file://` and send `Origin: null`. **This must be locked back to the real GitHub Pages origin (`https://gulmohammedansari.github.io`) before/at real cutover** — see the TODO list below. Don't leave it as `*` in real production.

### Data migration: done, for real data

The one-time Sheets → Firestore migration (`backend/scripts/migrate/migrate.js`, run manually from Cloud Shell) has been run successfully against the real production spreadsheets. Migrated: 3 tenants (`T-GME-PRIMARY`, `T-6CC14FE8B8`, `T-GME-DINERS`), 4 users, 31 invoices total, 24 buyers, 2 challans, 2 quotations, 1 credit/debit note, 4 certificates, 7 AMC contracts (`T-GME-DINERS` only). It's idempotent (deterministic doc IDs) — safe to re-run to pick up anything saved through the still-live Apps Script system since, right up until actual cutover. Command (needs `CONTROL_SPREADSHEET_ID`, see `backend/README.md`'s "Data migration" section):
```
CONTROL_SPREADSHEET_ID=<control sheet ID> npm run migrate
```

### Cutover status: DONE — live GitHub Pages now points at Cloud Run

`ALLOWED_ORIGIN` was locked to `https://gulmohammedansari.github.io` and redeployed, then `index.html`/`super-admin.html`'s `googleScriptUrl` change was committed and pushed to the public repo — GitHub Pages now serves the version pointing at the Cloud Run backend. (One wrinkle worth knowing about if you're looking at `git log`: the *identical* `googleScriptUrl` edit had also been made directly on GitHub's web UI in parallel, producing a second, separately-authored commit with the same content change — resolved with an ordinary `git merge`, no conflict since the text matched exactly.)

**What was verified before cutover:** login, dashboard, AMC contract save (including a contract number with slashes, a direct real-world test of the doc-ID fix below), Calendar reminder sync (after sharing), invoice PDF generation. **Still not yet verified — do this against the now-live site, not staging:** Reports page, Super Admin Portal actions, Quotation/Credit-Debit-Note/Certificate saves, a WhatsApp send end-to-end with a real recipient.

**The Apps Script deployment is still intact and paused-but-live** as the rollback path — same reasoning as the old system's "no version control, don't delete the fallback" caution in §4, just applied one layer up now. If something surfaces that the pre-cutover testing missed, revert the `googleScriptUrl` line in both files and push — traffic goes back to Apps Script immediately, no Cloud Run/Firestore state needs undoing.

### Critical bugs found during Phase 12 real-world testing (both fixed — do not reintroduce)

These were **invisible to every unit test** across the whole migration (Phases 1-11) because none of them ever exercised a real Firestore/Storage instance — no live GCP project existed until Phase 12. Both surfaced the moment real data actually flowed through the deployed service. Worth reading in full if you're touching `lib/docId.js`, `lib/storage.js`, or any route handler's `.doc(...)` calls.

1. **Firestore document IDs cannot contain "/".** This app's own auto-generated numbers (invoices, challans, quotations, credit/debit notes, certificates — anything built by `lib/sequentialNo.js`'s `currentFYPrefix()`) are constructed *with* slashes by design (`INV/26-27/001`), and every route handler originally passed that string straight into `.doc(id)`. Depending on the slash count, Firestore either silently misfiled the document at an unintended nested path or threw a hard error. **Fix:** `backend/src/lib/docId.js`'s `toSafeDocId()`/`fromSafeDocId()`, applied at every `.doc()` call site across `invoices.js`/`challans.js`/`quotations.js`/`creditDebitNotes.js`/`certificates.js`/`amc.js`/`buyers.js`/`items.js` and the migration script. **If you add a new document type with an auto-numbered or free-form ID, sanitize it the same way** — this class of bug will recur silently otherwise.
2. **Cloud Storage: per-object ACLs, never bucket-level public IAM.** The bucket must **not** have uniform bucket-level access enabled. `file.makePublic()` (per-object ACL) is what makes each uploaded PDF/photo individually shareable — this was briefly "fixed" (when UBLA broke `makePublic()`) by granting `roles/storage.objectViewer` to `allUsers` at the bucket level instead, which turned out to be a real security hole: that predefined role bundles "read one object" with "list every object in the bucket," with no way to grant just the first — every tenant's invoices/certificates/AMC photos became publicly enumerable, not just individually linkable, for a period during testing. **Never grant a bucket-level public role here** — if `file.makePublic()` ever errors again, the fix is to check `--no-uniform-bucket-level-access` on the bucket, not to reach for bucket IAM.

### Deployment gotcha: `openssl rand | gcloud secrets create` bakes in a trailing newline

Hit this for real with `SUPER_ADMIN_SECRET`: `openssl rand -hex N` always prints a trailing newline, and piping that straight into `gcloud secrets create --data-file=-` stores that newline as part of the secret's actual bytes. Every login then fails with "invalid super-admin secret" even when typing the *exact* value shown back via `gcloud secrets versions access latest` — the comparison is exact-match, and `"...c72b\n" !== "...c72b"`. Diagnosed by piping the secret through `wc -c` (48 expected for a hex-24 secret; 49 means a stray newline). Fixed by adding a clean version: `printf '%s' "<value>" | gcloud secrets versions add SUPER_ADMIN_SECRET --data-file=-`, then forcing a new Cloud Run revision (`gcloud run services update ... --update-secrets=...`) since `:latest` resolves at deploy time, not continuously. `DEPLOYMENT.md`'s Section 5 command was fixed to avoid this for any future secret creation — always generate into a shell variable first, `echo -n`/`printf '%s'` it in, never pipe a command's raw stdout (which usually ends in `\n`) directly into `--data-file=-`.

### Email: switched from Resend to Gmail SMTP (2026-09-05) — real recipients confirmed working

Originally used Resend (`RESEND_API_KEY`/`EMAIL_FROM=onboarding@resend.dev`), but Resend's shared test domain can only deliver to the account's own signup address — any real buyer/customer recipient got a silent 403, confirmed by testing. Verifying a custom domain with Resend would have meant buying a domain, which the user didn't have. **Switched `lib/email.js` to send via Gmail SMTP (`nodemailer`, `service: 'gmail'`) using an App Password** instead — no OAuth/app-verification review, no domain needed, sends to any recipient immediately. Env vars are now `GMAIL_USER`/`GMAIL_APP_PASSWORD` (was `EMAIL_FROM`/`RESEND_API_KEY` — see `DEPLOYMENT.md` Section 6 for the App Password setup steps). Same function signature and fail-soft `{sent, message}` contract as before, so no caller changed. Every buyer/recipient-facing email (`sendInvoiceToBuyer`, `sendCertificateToBuyer`, `sendChallanToBuyer`, `sendQuotationToBuyer`) still sets **Reply-To to the tenant's own `company.email`** so replies reach the real seller, not the shared Gmail sending account.

**Gotcha hit during rollout**: the code change was made locally but not committed/pushed before the first Cloud Shell redeploy, so that redeploy picked up the new env vars but rebuilt the *old* Resend-based code — the app kept showing "RESEND_API_KEY/EMAIL_FROM missing" even with the new vars in place. Fixed by actually committing+pushing (`8a1d80b`), then `git pull` + redeploy in Cloud Shell. Lesson: after any local backend code change, confirm `git push` happened before telling the user to redeploy — env var changes alone don't matter if Cloud Shell's checkout doesn't have the corresponding code yet.

**Verified live** (2026-09-05): real end-to-end send confirmed to a genuine buyer address (`ansarigulmohd2009@gmail.com`, not the sending account) — arrived in Spam initially (expected for a brand-new sending Gmail account with no reputation yet; worth telling users to whitelist/mark-not-spam the first time). Trade-off to keep in mind going forward: sender shows as a personal-looking Gmail address, not a branded domain, and regular Gmail accounts cap around 500 sends/day — fine for current volume.

### Auto seller-copy email on save: ON, goes to the tenant's own email

Every document save (`handleAddInvoice`/`handleSaveChallan`/`handleSaveQuotation`/`handleSaveCreditDebitNote`) auto-fires a seller-copy notification via `sendDocumentSavedNotification()`, faithfully matching `Gscript.txt`'s original `sendInvoiceEmail`/`sendChallanEmail`/etc. behavior. Recipient is `company.email` — **the tenant's own contact email, entered when that tenant was created via `provisionTenant`** (Super Admin Portal) or later changed in that tenant's own Company Settings — never the buyer/customer.

This was briefly turned off (a misunderstanding — it initially looked like it might be emailing the *customer* automatically, which would have been a real problem; once confirmed it was actually going to the tenant's own configured address, the user asked to keep it on) and reverted right back via `git revert`. If this comes up again: confirm which address is actually receiving it (`company.email`, i.e. the tenant's own) before assuming it's wrong — **it never emails the buyer/customer automatically, only `sendInvoiceToBuyer`/`sendCertificateToBuyer`/`sendChallanToBuyer`/`sendQuotationToBuyer`** (all explicit, button-triggered) do that.

### Buyer-facing WhatsApp + Email: now on all 4 document types with buyer contact info (post-migration feature)

Originally only Invoices and Certificates had buyer-facing "WhatsApp"/"Email" send buttons (that's all `Gscript.txt` ever built). Extended to **Challans and Quotations** too, at the user's explicit request — same manual-trigger-only pattern, same backend shape (`getChallanWhatsappLink`/`sendChallanToBuyer`, `getQuotationWhatsappLink`/`sendQuotationToBuyer` in `backend/src/routes/challans.js`/`quotations.js`, wired in `index.js`), same frontend button placement (table rows, plus the view modal for Challans — Quotations have no view modal in the original app, so table-row buttons only). Added `challanPdfUrl`/`quotationPdfUrl` fields (cleared on every save, same invariant as `invoicePdfUrl`) since neither had a PDF-URL column before. **Credit/Debit Notes deliberately excluded** — the user was asked explicitly and said no, not "not gotten to yet."

**Verified live**: both Email and WhatsApp send confirmed fully working for Challans/Quotations — Email to real (non-self) buyer addresses (see "Email: switched from Resend to Gmail SMTP" above), WhatsApp with a real phone number end to end.

### Delete for Invoice/Challan/Quotation/Certificate + a deletion audit log (post-migration feature)

Deleting one of these four document types outright was previously impossible in the app — `Gscript.txt` never had it either. Added at the user's request, with two constraints they set explicitly: a confirm dialog before deleting, and every deletion logged somewhere.

- **Backend**: `handleDeleteInvoice`/`handleDeleteChallan`/`handleDeleteQuotation`/`handleDeleteCertificate` (one per `routes/*.js` file) — **Admin-role-gated**, added to `ADMIN_ONLY_ACTIONS` in `index.js` (same gate `deleteAMCContract`/`saveCompanySettings` already use). Every successful delete calls `lib/deletionLog.js`'s `recordDeletion()`, which writes to a new tenant-scoped `deletionLog` Firestore collection (`docType`, `docNo`, `deletedBy`, plus both a display timestamp and a real ISO `timestampSort` field for correct ordering — `nowIST()`'s locale string doesn't sort correctly as a plain string, unlike every other collection where `timestamp` is only ever displayed, never ordered by).
- **New `getDeletionLog` GET action**, also Admin-only — the **first GET action that ever needed role gating**, so a new `ADMIN_ONLY_GET_ACTIONS` array + a matching check were added to the GET router in `index.js` (mirroring the POST-side `ADMIN_ONLY_ACTIONS` check that already existed).
- **Frontend**: a red "Delete" button (with `confirm()`) on each document type's table row, and on the Invoice/Challan view-modal footers (Quotations/Certificates have no view modal, same as the WhatsApp/Email feature above). A new **Deletion Log** nav page (read-only table: Document Type / Document No / Deleted By / Timestamp).
- **No frontend role-hiding** — deliberately consistent with how Company Settings/AMC-delete already work in this app: the Delete button and Deletion Log nav item are visible to everyone, and a non-Admin who tries either gets the backend's `"This action requires an Admin role."` message via a toast/the page body, rather than the button being hidden. Don't try to add role-based UI hiding here without checking with the user first — it would be a real UX improvement but is a deliberate scope choice made to match the existing app, not an oversight.

**Verified live** (2026-09-05): redeployed, an Admin-user delete confirmed working end to end (document removed, entry appears in the Deletion Log page with correct type/no/user/timestamp).

### Edit for Quotation/Certificate + Admin-only editing on all 4 document types (post-migration feature)

`Gscript.txt` never had an edit path for Quotations or Certificates at all (reprint/view-only) — added at the user's explicit request, mirroring Invoice/Challan's existing edit pattern exactly (rename-aware: matches by the *original* document number, delete+recreate if the number itself changed since Firestore documents can't be renamed in place, preserves the original timestamp).

- **Backend**: `handleUpdateQuotation`/`handleUpdateCertificate` added to `routes/quotations.js`/`certificates.js`, structurally identical to `handleUpdateChallan`. `buildQuotationDoc`/`buildCertificateDoc` both gained an `existingTimestamp` parameter (previously only `buildInvoiceDoc`/`buildChallanDoc` had one) so an edit doesn't reset `timestamp` to "now".
- **Admin-role gating — deliberately retroactive**: the user was asked explicitly whether Admin-only editing should apply just to the new Quotation/Certificate capability or *also* to the already-shipped, already-in-use Invoice/Challan edit (which until now any logged-in user could do) — they chose **all four**. `updateInvoice`/`updateChallan`/`updateQuotation`/`updateCertificate` are all now in `ADMIN_ONLY_ACTIONS`. **This is a real behavior change for existing users**: anyone who isn't an Admin can no longer edit an Invoice or Challan they previously could. If this surfaces as a support question ("I used to be able to edit invoices"), that's expected — not a bug — per this explicit decision.
- **Frontend**: `quotState`/`certState` both gained `editMode`/`originalQuotNo`/`originalCertNo` fields (mirroring `challanState`'s `editMode`/`originalChallanNo`). `editQuotation(quotNo)`/`editCertificate(certNo)` load a saved record into the New Quotation/New Certificate page for editing, mirroring `editChallan` line-for-line. `handleSaveQuotation()`/`handleSaveCertificate()` now check `editMode` first and delegate to new `handleUpdateQuotationToSheets()`/`handleUpdateCertificateToSheets()` functions when true — same dispatch pattern as `handleSaveChallan()`. New "Edit" button added to both tables' row templates.
- **Two small pre-existing gaps filled to make this possible**: the Quotation page's title span and Save button had no `id` to update during edit-mode (unlike Challan/Certificate, which already had them) — added `id="quotation-page-title"` and wrapped the button label in `id="quot-save-btn-label"`. The Certificate page's line-item row template (`addCertItem()`) had no per-field CSS class hooks (unlike Challan's `.chi-desc-*` etc.) — added `.cti-desc-*`/`.cti-qty-*`/`.cti-unit-*`/`.cti-remarks-*` so `editCertificate` can populate saved values into existing rows, matching Challan's approach.
- **No frontend role-hiding**, same deliberate choice as Delete/Company Settings — Edit buttons stay visible to everyone, a non-Admin gets the backend's rejection message instead.

**Not yet verified live** at time of writing — needs a redeploy, then an Admin-user edit test for both Quotation and Certificate (including a rename — changing the document number during edit), plus a non-Admin edit attempt on all four types to confirm the rejection message.

### Old WhatsApp-PDF cleanup on regeneration (post-migration feature)

User noticed the same question the storage-usage check surfaced: every `getInvoiceWhatsappLink`/`getChallanWhatsappLink`/`getQuotationWhatsappLink`/`getCertificateWhatsappLink` call regenerates a **fresh** PDF (deliberate, so branding changes always show up — see that invariant note elsewhere in this file) and uploads it under a new object name, but the *previous* file at the old URL was never deleted — just abandoned in the bucket forever.

**Important scoping finding, confirmed by grepping every route file for `uploadPdfBuffer`**: only these 4 "get WhatsApp link" functions ever upload anything to Cloud Storage. The "send via Email" functions (`sendInvoiceToBuyer`/`sendChallanToBuyer`/`sendQuotationToBuyer`/`sendCertificateToBuyer`) attach the generated PDF buffer directly to the email and never touch Cloud Storage at all — there was nothing to clean up on that path, contrary to how the user's request was originally phrased ("updated & sent via Email/WhatsApp"). Worth remembering if a similar-sounding request comes up again: check `uploadPdfBuffer` call sites before assuming a code path persists anything.

- **`lib/storage.js`** gained `deleteUploadedFile(url)` — parses the object name back out of the public URL format `uploadBuffer()` already returns, deletes it, and fails soft (`{deleted:false, message}`) on any error (foreign URL, already gone, GCS error) rather than throwing — cleanup must never break the WhatsApp-link request that triggered it.
- Each of the 4 `handleGetXWhatsappLink` functions now captures the document's *old* PDF URL before generating the new one, and after successfully updating Firestore with the new URL, deletes the old object (only if one existed and actually differs from the new one). A successful cleanup is recorded in the **same `deletionLog` collection/Deletion Log page** used for document deletions, with `docType` values like `'Invoice PDF (superseded)'` so they're clearly distinguishable from an actual Invoice/Challan/Quotation/Certificate record deletion in that same list.
- All 4 functions gained a `username` parameter (threaded from `session.username` in `index.js`) purely for this log's `deletedBy` field — falls back to `'system'` if somehow absent. Not Admin-gated itself (unlike the Delete/Edit features) — this is routine cleanup triggered by an ordinary WhatsApp-send action any user can already do, not a new destructive capability.

**Verified live** (2026-09-05): confirmed on a genuinely untouched invoice (`INV/26-27/341`) — clicked WhatsApp twice, `gcloud storage ls` on that invoice's folder showed only the newest file after the second click (the first was gone), and two `"Invoice PDF (superseded)"` entries appeared in the Deletion Log at the right timestamps with the correct username.

**Gotcha hit while verifying**: testing on invoices that had been *edited* between WhatsApp clicks (buyer name changed via the Edit feature) made cleanup look broken at first — no old file got deleted. Not a bug: every edit/save resets `invoicePdfUrl` back to `''` (a separate, older "never serve a stale cached PDF" invariant — see the "always regenerate fresh" note elsewhere in this file), so the *next* WhatsApp click after an edit correctly sees "nothing to clean up." Only testing on a document with zero edits in between gives an unambiguous result. Worth remembering next time this needs re-verifying after a change.

### Stamp rendered too small in the "Authorised signatory" block (post-migration fix)

User's actual stamp graphic is wide/oval (like most real company seals), but the box it renders into was a square (`max-height:11mm;max-width:11mm` for Invoice/Challan/Quotation/Credit-Debit-Note; `16mm;16mm` for Certificate) — width was the real constraint, not height, so a wide stamp got squeezed down far more than the available space actually allowed.

**Fix: widened `max-width` only, left `max-height` untouched.** The vertical space in this block is tightly budgeted — a real bug once clipped the "Authorised signatory" text below it when the box was sized too tall (see `bankBlock()`'s big comment in `ledgerPrint.js`/`index.html`) — so changing height at all would risk reintroducing that. Widening only the horizontal cap has zero effect on the vertical footprint, so there's no such risk here. New values: 11mm→**20mm** (Invoice family), 16mm→**28mm** (Certificate). Verified by rendering `ledgerPrint.buildDocument()` locally with a synthetic wide-oval test stamp and inspecting the output in the browser — stamp renders clearly larger, "Authorised signatory" still uncut.

**Four places changed, all doing the exact same "both present" branch** (this app keeps the browser's print engine and the backend's Puppeteer-rendered PDF engine in sync deliberately, so a fresh WhatsApp PDF and a browser Print/Preview always look the same):
- `backend/src/lib/ledgerPrint.js` (`bankBlock()`) + `index.html`'s matching `bankBlock()` — Invoice/Challan/Quotation/Credit-Debit-Note.
- `backend/src/lib/certificateHtml.js` + `index.html`'s `printCertificateHtml()` — Certificate.

Only the "stamp AND signature both present" branch was touched — the "only one of the two" fallback branches were left alone (already reasonably generous, and not what was reported).

**Not yet verified live** — needs a redeploy (backend) + the frontend push to actually take effect; the local render test above only proves the HTML/CSS is correct, not that it looks right in a real Puppeteer-generated PDF or an actual browser print. Confirm by regenerating a WhatsApp PDF (or Print/Preview) for a tenant that has both a stamp and signature uploaded in Company Settings.

### TODO now that cutover has happened

1. ~~Redeploy with `ALLOWED_ORIGIN=https://gulmohammedansari.github.io`~~ — done.
2. ~~Commit + push the `googleScriptUrl` change~~ — done, live.
3. ~~A real WhatsApp send~~ — done, confirmed working (Challans/Quotations). **Remaining to verify against the live site**: Reports page, Super Admin Portal actions, Quotation/Credit-Debit-Note/Certificate saves.
4. ~~Set up email for real recipients~~ — done via Gmail SMTP, confirmed working for real buyer addresses (see "Email: switched from Resend to Gmail SMTP" above). Revisit only if sending volume outgrows Gmail's ~500/day cap.
5. If anything from #3 turns up broken: revert the `googleScriptUrl` line in both files and push (instant rollback to Apps Script), fix, redeploy, re-cut-over.
6. Once confident the new backend has been correct in production for a while, consider actually retiring the Apps Script deployment (pause it, don't delete — no rush on this).
7. Repeat the calendar-sharing step (§11's "Critical bugs" isn't this — see the AMC section above) for any tenant that wants AMC reminders and hasn't been set up yet.

## 12. Apps Script disaster-recovery fallback (built as a separate contingency track, live 2026-09-06)

A second, independent recovery path in case Cloud Run/Firestore itself is ever unavailable —
not a replacement for anything in §11, a fallback *for* it. Full setup/runbook lives in
[backend/APPS_SCRIPT_FALLBACK.md](backend/APPS_SCRIPT_FALLBACK.md); this section is the
what-and-why for whoever picks this up next.

**What it is**: [Gscript_v2.txt](Gscript_v2.txt) (root, gitignored, local-only) — a fork of
the original `Gscript.txt` with every feature added to the Node backend since the migration
ported over by hand (Delete + Deletion Log for all 4 document types, Edit for Quotation/
Certificate, Admin-only gating on edit/delete, buyer-facing WhatsApp/Email for Challans/
Quotations, the wide-stamp fix, PDF-cleanup-on-regeneration). Deployed **twice** as two
separate Apps Script Web Apps from the same code:
- **Read-only viewer** — safe to open any time, rejects every write with a friendly message.
- **Full contingency** — genuinely usable if Cloud Run is down for real; anything created
  here needs manual reconciliation back into Firestore later (accepted gap, not automated).

Distinguishing the two **cannot** use a Script Property (`READ_ONLY_MODE`) the way the first
draft assumed — Script Properties are project-wide, not per-deployment, so both deployments
would see the same value. Fixed by comparing `ScriptApp.getService().getUrl()` against a
hardcoded `READ_ONLY_DEPLOYMENT_URL` constant instead, verified live via a temporary no-auth
`debugDeploymentUrl` diagnostic action (since removed from `Gscript_v2.txt` once confirmed
working).

**Data flow — one-way, Firestore → Sheets**: `backend/scripts/export/exportToSheets.js`
(`runExport()`) reads every tenant's Firestore data and writes it into that tenant's own
fallback spreadsheet (created once, ID remembered on the tenant's Firestore doc as
`fallbackSpreadsheetId`), plus a shared control spreadsheet's `Tenants`/`Users` tabs. A full
overwrite each run, not a merge — see that script's own header comment. Runnable two ways:
- Manually: `npm run export-to-sheets` from Cloud Shell.
- **Automated (live now)**: Cloud Scheduler job `fallback-export-daily` (2 AM IST) calls a
  new secret-gated `runFallbackExport` action on the *existing* Cloud Run service
  (`src/routes/fallbackExport.js`, gated by `FALLBACK_EXPORT_SECRET` in Secret Manager) — no
  second always-on service needed.

**Fallback logins use unique, randomly-generated per-user passwords, never real ones** —
real passwords are bcrypt-hashed (Node) and mathematically cannot convert to or from Apps
Script's own SHA-256×10,000-iteration scheme; a shared single fallback password was
considered and explicitly rejected (user's own security concern: a leaked/guessed password
would expose every user in every tenant) in favor of one unique password per user,
regenerated on every export run and printed only to Cloud Run's own logs — deliberately
never returned in the `runFallbackExport` HTTP response, since a Scheduler job's execution
history is a wider-audience surface than intended for those.

**All fallback spreadsheets live in one Drive folder**, `Itransform Technology Cloud
Storage` (ID in `FALLBACK_DRIVE_FOLDER_ID`), shared as Editor with the runtime service
account — new spreadsheets are placed there automatically by `createSharedSpreadsheet()`;
pre-existing ones had to be moved in by hand (one-time, not scripted).

**Real gotcha, cost real debugging time — worth knowing before touching this again**: a
spreadsheet not explicitly shared with the runtime service account
(`426403271855-compute@developer.gserviceaccount.com`) makes `runExport()` fail with
`GaxiosError: The caller does not have permission` (403) on that specific spreadsheet, even
though the control spreadsheet and secret are all fine — several per-tenant fallback
spreadsheets, created earlier under a personal Google identity during initial testing, hit
exactly this. See `backend/DEPLOYMENT.md`'s matching gotcha entry for the full diagnostic
path (reading the exact failing spreadsheet ID out of Cloud Run logs). A second, unrelated
deploy-time gotcha (`.gcloudignore` vs. the `Dockerfile`'s own `COPY` instructions being
independent gates) is also documented there.

**Not yet done**: the Step 5 end-to-end login test (both deployments) from
`APPS_SCRIPT_FALLBACK.md` — do this before actually trusting the fallback in a real outage.
