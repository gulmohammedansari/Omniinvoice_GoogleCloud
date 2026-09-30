# Firestore Export Commands

Complete guide for exporting data from Firestore to Excel format.

## Prerequisites

1. Navigate to backend directory in Cloud Shell:
```bash
cd ~/gme-invoiceapp-backend
```

2. Ensure xlsx package is installed (only needed once):
```bash
npm install
```

## Export Commands

### 1. Export All Tenants to Excel

Exports all tenant data + control data (tenants & users) to separate Excel files.

```bash
npm run export-to-excel
```

**Output:**
- Creates timestamped folder: `exports/export_YYYY-MM-DDTHH-mm-ss/`
- `Control_Tenants_Users.xlsx` - All tenants and users
- One `.xlsx` file per tenant with all their business data

### 2. Export Single Tenant to Excel

Export only one specific tenant's data.

```bash
npm run export-to-excel -- --tenant=T-GME-PRIMARY
```

**Replace `T-GME-PRIMARY` with your tenant ID.**

### 3. Custom Output Directory

Specify where to save the exported files.

```bash
npm run export-to-excel -- --output=./backups
```

### 4. Combined Options

Export single tenant to custom directory:

```bash
npm run export-to-excel -- --tenant=T-GME-DINERS --output=./backups
```

## Excel File Structure

### Control_Tenants_Users.xlsx
Contains 2 sheets:
- **Tenants** - All tenant profiles
- **Users** - All user accounts

### Per-Tenant Files (e.g., T-GME-PRIMARY_G_M_ENTERPRISES.xlsx)
Contains 8 sheets:
1. **Invoices** - All invoices
2. **Challans** - Delivery challans
3. **Quotations** - Price quotations
4. **Certificates** - Service certificates
5. **CreditDebitNotes** - Credit/debit notes
6. **AMC_Contracts** - AMC contracts
7. **Buyers** - Customer database
8. **Items** - Product/service catalog

## Output Example

```
exports/export_2026-09-27T04-30-00/
├── Control_Tenants_Users.xlsx
├── T-GME-PRIMARY_G_M_ENTERPRISES.xlsx
└── T-GME-DINERS_Diner_s_Fire_Safety.xlsx
```

## Download Files from Cloud Shell

### Option 1: Using Cloud Shell File Browser
1. Click the **⋮** (three dots) menu in Cloud Shell
2. Select **Download file**
3. Enter path: `gme-invoiceapp-backend/exports/export_2026-09-27T04-30-00/Control_Tenants_Users.xlsx`

### Option 2: Using gsutil (Cloud Storage)
```bash
# Upload to Cloud Storage bucket
gsutil -m cp -r exports/export_2026-09-27T04-30-00 gs://YOUR-BUCKET-NAME/

# Generate download URL (valid for 1 hour)
gsutil signurl -d 1h service-account-key.json gs://YOUR-BUCKET-NAME/export_2026-09-27T04-30-00/Control_Tenants_Users.xlsx
```

### Option 3: Download All as ZIP
```bash
# Create zip archive
cd exports
zip -r export_2026-09-27T04-30-00.zip export_2026-09-27T04-30-00/

# Download the zip file using Cloud Shell download
# File path: gme-invoiceapp-backend/exports/export_2026-09-27T04-30-00.zip
```

## One-Liner Commands

### Export all data and show summary:
```bash
cd ~/gme-invoiceapp-backend && npm run export-to-excel
```

### Export specific tenant:
```bash
cd ~/gme-invoiceapp-backend && npm run export-to-excel -- --tenant=T-GME-PRIMARY
```

### Export to backups folder:
```bash
cd ~/gme-invoiceapp-backend && npm run export-to-excel -- --output=./backups
```

## Comparison: Excel vs Google Sheets Export

| Feature | Excel Export | Sheets Export |
|---------|-------------|---------------|
| **Command** | `npm run export-to-excel` | `npm run export-to-sheets` |
| **Output** | Local .xlsx files | Google Sheets (cloud) |
| **Download** | Manual download needed | Already in Drive |
| **Offline** | ✅ Yes | ❌ No |
| **Sharing** | Email attachments | Share link |
| **Size Limit** | None (local disk) | Google Drive quota |
| **Use Case** | Backup, offline analysis | Live fallback, collaboration |

## Troubleshooting

### Error: "Cannot find module 'xlsx'"
```bash
cd ~/gme-invoiceapp-backend
npm install xlsx
```

### Error: "ENOENT: no such file or directory"
Make sure you're in the backend directory:
```bash
pwd
# Should show: /home/itransform007/gme-invoiceapp-backend
```

### Permission denied when creating exports folder
```bash
mkdir -p exports
chmod 755 exports
```

## Automated Scheduled Exports

To automate exports using Cloud Scheduler:

1. Create a Cloud Function that calls the export script
2. Set up Cloud Scheduler to trigger it daily/weekly
3. Upload results to Cloud Storage
4. Send notification email with download links

*(Implementation available upon request)*

## Notes

- Export is **read-only** - does not modify Firestore data
- Each export creates a **new timestamped folder**
- Old exports are **not automatically deleted** - clean up manually
- Excel files include **auto-sized columns** for readability
- JSON fields (items, ship-to) are **preserved as-is** in cells

---

**Last Updated:** 2026-09-27  
**Backend Version:** 0.1.0
