# 🧾 Ledger Pro — Business Finance Tracker

> Track expenses and income for freelancers and small businesses. Google Sheets sync, CSV export, receipt photos, 6 languages.

**Live App:** https://farboddavani-cmyk.github.io/expense-tracker/  
**Play Store:** `com.farboddavani.ledger`  
**Version:** 1.0.0

---

## 📁 Project Structure

```
ledger-pro/
├── index.html              ← Complete single-file PWA
├── manifest.json           ← PWA manifest
├── sw.js                   ← Service worker (offline caching)
├── icons/
│   ├── icon-192.png        ← PWA icon (192×192)
│   ├── icon-512.png        ← PWA icon (512×512)
│   ├── icon-512-maskable.png ← Maskable icon for Android
│   └── feature-graphic.png ← Play Store feature graphic (1024×500)
├── PRIVACY_POLICY.md       ← Privacy policy
├── PLAY_STORE_LISTING.md   ← Play Store copy & metadata
├── README.md               ← This file
└── .github/
    └── workflows/
        └── deploy.yml      ← GitHub Pages auto-deploy
```

---

## 🚀 Deploy to GitHub Pages

### Option 1: GitHub Actions (Recommended)

1. Push this repo to `github.com/farboddavani-cmyk/expense-tracker`
2. Go to **Settings → Pages → Source → GitHub Actions**
3. Push any commit to `main` — it deploys automatically
4. Live at: `https://farboddavani-cmyk.github.io/expense-tracker/`

### Option 2: Manual

```bash
git init
git add .
git commit -m "Initial release"
git remote add origin https://github.com/farboddavani-cmyk/expense-tracker.git
git push -u origin main
# Then enable Pages in repo Settings → Pages → Deploy from branch: main / root
```

---

## 📱 Package for Play Store (PWABuilder)

1. Open https://www.pwabuilder.com
2. Enter your GitHub Pages URL
3. Click **Package for Stores → Android**
4. Configure:
   - Package ID: `com.farboddavani.ledger`
   - App version: `1`
   - Display mode: `standalone`
5. Download the Android project zip

### Add Custom Android Files

After generating with PWABuilder, add the Kotlin files from `android/`:

#### `TrialManager.kt`
```kotlin
package com.farboddavani.ledger

import android.content.Context
import android.content.SharedPreferences

class TrialManager(context: Context) {
    private val prefs: SharedPreferences = 
        context.getSharedPreferences("ledger_prefs", Context.MODE_PRIVATE)
    private val TRIAL_DAYS = 10L

    fun initTrial() {
        if (prefs.getLong("trial_start", 0L) == 0L) {
            prefs.edit().putLong("trial_start", System.currentTimeMillis()).apply()
        }
    }

    fun getDaysLeft(): Long {
        val start = prefs.getLong("trial_start", System.currentTimeMillis())
        val elapsed = (System.currentTimeMillis() - start) / 86400000L
        return maxOf(0L, TRIAL_DAYS - elapsed)
    }

    fun isTrialActive() = getDaysLeft() > 0
    fun isLicensed() = prefs.getBoolean("licensed", false)
    fun hasAccess() = isLicensed() || isTrialActive()

    fun setLicensed(licensed: Boolean) {
        prefs.edit().putBoolean("licensed", licensed).apply()
    }
}
```

#### `BillingManager.kt`
```kotlin
package com.farboddavani.ledger

import android.app.Activity
import android.content.Context
import com.android.billingclient.api.*

class BillingManager(
    private val context: Context,
    private val onPurchaseSuccess: () -> Unit
) : PurchasesUpdatedListener {
    private val PRODUCT_ID = "ledger_unlock_full"
    private lateinit var billingClient: BillingClient

    fun init() {
        billingClient = BillingClient.newBuilder(context)
            .setListener(this)
            .enablePendingPurchases()
            .build()
        billingClient.startConnection(object : BillingClientStateListener {
            override fun onBillingSetupFinished(result: BillingResult) {
                if (result.responseCode == BillingClient.BillingResponseCode.OK) {
                    checkExistingPurchases()
                }
            }
            override fun onBillingServiceDisconnected() {}
        })
    }

    fun launchPurchase(activity: Activity) {
        val productList = listOf(
            QueryProductDetailsParams.Product.newBuilder()
                .setProductId(PRODUCT_ID)
                .setProductType(BillingClient.ProductType.INAPP)
                .build()
        )
        billingClient.queryProductDetailsAsync(
            QueryProductDetailsParams.newBuilder().setProductList(productList).build()
        ) { _, productDetailsList ->
            val product = productDetailsList.firstOrNull() ?: return@queryProductDetailsAsync
            val flowParams = BillingFlowParams.newBuilder()
                .setProductDetailsParamsList(listOf(
                    BillingFlowParams.ProductDetailsParams.newBuilder()
                        .setProductDetails(product)
                        .build()
                ))
                .build()
            billingClient.launchBillingFlow(activity, flowParams)
        }
    }

    fun restorePurchases() = checkExistingPurchases()

    private fun checkExistingPurchases() {
        billingClient.queryPurchasesAsync(
            QueryPurchasesParams.newBuilder()
                .setProductType(BillingClient.ProductType.INAPP)
                .build()
        ) { _, purchases ->
            purchases.forEach { purchase ->
                if (purchase.purchaseState == Purchase.PurchaseState.PURCHASED &&
                    PRODUCT_ID in purchase.products) {
                    if (!purchase.isAcknowledged) acknowledgePurchase(purchase)
                    onPurchaseSuccess()
                }
            }
        }
    }

    private fun acknowledgePurchase(purchase: Purchase) {
        billingClient.acknowledgePurchase(
            AcknowledgePurchaseParams.newBuilder()
                .setPurchaseToken(purchase.purchaseToken)
                .build()
        ) {}
    }

    override fun onPurchasesUpdated(result: BillingResult, purchases: List<Purchase>?) {
        if (result.responseCode == BillingClient.BillingResponseCode.OK) {
            purchases?.forEach { purchase ->
                if (purchase.purchaseState == Purchase.PurchaseState.PURCHASED) {
                    if (!purchase.isAcknowledged) acknowledgePurchase(purchase)
                    onPurchaseSuccess()
                }
            }
        }
    }
}
```

#### `MainActivity.kt` additions
```kotlin
// In your PWABuilder-generated MainActivity, add:
private lateinit var trialManager: TrialManager
private lateinit var billingManager: BillingManager

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    
    trialManager = TrialManager(this)
    trialManager.initTrial()
    
    billingManager = BillingManager(this) {
        // Called when purchase is verified
        trialManager.setLicensed(true)
        runOnUiThread {
            webView.evaluateJavascript("window.onPurchaseSuccess && window.onPurchaseSuccess()", null)
        }
    }
    billingManager.init()
    
    // Expose bridge to WebView
    webView.addJavascriptInterface(object {
        @android.webkit.JavascriptInterface
        fun purchaseProduct(productId: String) {
            runOnUiThread { billingManager.launchPurchase(this@MainActivity) }
        }
        @android.webkit.JavascriptInterface
        fun restorePurchase() {
            billingManager.restorePurchases()
        }
    }, "Android")
}
```

---

## 🔧 Google Apps Script Setup

1. Open Google Sheets
2. Go to **Extensions → Apps Script**
3. Replace `Code.gs` with the contents below, then **Deploy → New deployment → Web App**
   - Execute as: Me
   - Who has access: Anyone
4. Copy the deployment URL into Ledger Pro → Settings → Apps Script URL

### `Code.gs`
```javascript
const SHEET_ID = SpreadsheetApp.getActiveSpreadsheet().getId();
const EXP_SHEET = 'Expense Log';
const INC_SHEET = 'Income Log';
const RECEIPTS_FOLDER = 'Expense Tracker Receipts';
const INVOICES_FOLDER = 'Expense Tracker Invoices';

function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);
    const ss = SpreadsheetApp.openById(SHEET_ID);
    
    if (data.type === 'expense') {
      const sheet = ss.getSheetByName(EXP_SHEET);
      let receiptUrl = '';
      if (data.receiptBase64) {
        receiptUrl = uploadFile(data.receiptBase64, data.id + '_receipt', RECEIPTS_FOLDER);
      }
      sheet.appendRow([
        data.date, data.vendor, data.desc, data.category,
        data.amount, data.currency, data.method,
        data.taxDeductible ? 'Yes' : 'No',
        data.notes, receiptUrl, data.id
      ]);
    } else if (data.type === 'income') {
      const sheet = ss.getSheetByName(INC_SHEET);
      let invoiceUrl = '';
      if (data.invoiceBase64) {
        invoiceUrl = uploadFile(data.invoiceBase64, data.invoice + '_invoice', INVOICES_FOLDER);
      }
      sheet.appendRow([
        data.date, data.client, data.invoice,
        data.amount, data.currency, data.status,
        data.notes, invoiceUrl, data.id
      ]);
    }
    
    return ContentService
      .createTextOutput(JSON.stringify({ success: true }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch(err) {
    return ContentService
      .createTextOutput(JSON.stringify({ error: err.message }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

function doGet(e) {
  try {
    const type = e.parameter.type || 'expense';
    const ss = SpreadsheetApp.openById(SHEET_ID);
    const sheetName = type === 'income' ? INC_SHEET : EXP_SHEET;
    const sheet = ss.getSheetByName(sheetName);
    const rows = sheet.getDataRange().getValues();
    
    // Skip header rows (rows 1-3), data starts at row 4 (index 3)
    const headers = rows[2] || [];
    const data = rows.slice(3).map(row => {
      const obj = {};
      headers.forEach((h, i) => { obj[String(h).toLowerCase().replace(/\s+/g,'_')] = row[i]; });
      obj.type = type;
      return obj;
    }).filter(r => r.date || r.vendor || r.client);
    
    return ContentService
      .createTextOutput(JSON.stringify({ success: true, data }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch(err) {
    return ContentService
      .createTextOutput(JSON.stringify({ error: err.message }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

function uploadFile(base64Data, fileName, folderName) {
  try {
    // Strip data URL prefix
    const match = base64Data.match(/^data:([^;]+);base64,(.+)$/);
    if (!match) return '';
    const mimeType = match[1];
    const data = Utilities.base64Decode(match[2]);
    const blob = Utilities.newBlob(data, mimeType, fileName);
    
    // Find or create folder
    const folders = DriveApp.getFoldersByName(folderName);
    const folder = folders.hasNext() ? folders.next() : DriveApp.createFolder(folderName);
    
    const file = folder.createFile(blob);
    file.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW);
    return file.getUrl();
  } catch(err) {
    return '';
  }
}
```

---

## 🌍 Languages Supported

| Code | Language |
|------|----------|
| `en` | English |
| `es` | Español |
| `fr` | Français |
| `fa` | فارسی (RTL) |
| `de` | Deutsch |
| `pt` | Português |

---

## 💳 In-App Purchase

| Property | Value |
|----------|-------|
| Product ID | `ledger_unlock_full` |
| Type | One-time (INAPP) |
| Price | $9.99 |
| Trial | 10 days from first launch |
| Billing Library | Google Play Billing 7.0.0 |

---

## 🏗️ Tech Stack

- **Frontend:** Single-file HTML + CSS + Vanilla JS
- **PWA:** Web App Manifest + Service Worker
- **Backend:** Google Apps Script (your own deployment)
- **Storage:** Google Sheets (cloud) + localStorage (local)
- **Files:** Google Drive (receipts & invoices)
- **Android:** PWABuilder + Custom Kotlin (TrialManager + BillingManager)
- **Billing:** Google Play Billing Library 7.0.0

---

## 📄 License

Copyright © 2025 Farbod Davani. All rights reserved.

---

*Built with Ledger Pro v1.0.0*
