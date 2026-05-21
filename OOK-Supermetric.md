---
name: ook-report
description: >
  Full workflow skill for updating the One of a Kin (OOK) ad performance dashboard (index.html).
  Use this skill whenever the user uploads ad export files (Meta CSV, TikTok XLSX, Snap XLSX,
  Google CSV) and asks to update the OOK report, add a new day's data, insert daily ad
  performance, or update the dashboard. Also triggers on: "update the report", "add data for
  [date]", "process today's files", uploads of platform exports alongside index.html.
---

# OOK-Supermetric.md — OOK Ad Report Master Skill (Updated May 14 2026 — added RULES 11/12 + Snap attribution-window change to 1_DAY__7_DAY)

## Overview
Single-file HTML/JS dashboard tracking Meta, TikTok, Snap, Google across GCC markets.

**File:** `index.html` → `https://3rdscreen.github.io/ook-dashboard`  
**Working path:** `/home/claude/index.html` → deliver to `/mnt/user-data/outputs/index.html`  
**Data objects:** `const D={...}` (daily platform totals) · `const GRANULAR={...}` (ad/adset level)

---

## ═══════════════════════════════════════
## NON-NEGOTIABLE RULES — READ EVERY TIME
## ═══════════════════════════════════════

### RULE 1 — NEVER FABRICATE DATA
- Only insert numbers from actual uploaded export files
- New ads: data ONLY from the day they first appear in an export
- Spend-only entries (sv=0, conv=0, atc=0, roas=0, cpo=0) are acceptable when no conv data exists
- **NEVER** proportionally distribute sv/conv/roas/atc/cpo from D object totals
- **NEVER** invent ATC, conv, or SV — these must come from the export

### RULE 2 — SPEND MUST ALWAYS MATCH THE UPLOADED EXPORT EXACTLY
**This is non-negotiable.** Ibra has explicitly emphasized: the spend in the uploaded document must equal the spend in the dashboard to the cent.

For every day being inserted or updated, compute per-platform totals from the export and verify:
```python
# Meta (already USD)
meta_spend = m[m['Amount spent (USD)'] > 0]['Amount spent (USD)'].sum()  # all rows, sales + followers

# TikTok (AED → USD)
tt_spend = tt[tt['Primary status']=='Active']['Cost'].sum() / 3.67

# Snap (USD already)
snap_spend = snap['Amount Spent'].sum()

# Google (AED → USD, filter Campaign status == 'Enabled' to exclude total rows)
g_spend = g[g['Campaign status']=='Enabled']['Cost_clean'].sum() / 3.67
```
Then validate the D object's new `"Apr N"` entry has `spend` matching these values. If off by even a cent, stop and fix.

Also validate the sum of per-geo spends equals the top-level total, and sum of per-ad spends in GRANULAR equals the top-level total.

TikTok UAE spend has been wrong on multiple dates — always recompute from export, never trust stored geo values.

### RULE 3 — JSON PARSE + RE-SERIALIZE FOR D OBJECT EDITS
```python
d_start = content.find('const D={')
i = d_start + len('const D=')
# brace-count to find d_end
d_obj = json.loads(content[i:d_end])
# modify d_obj entries
new_d = json.dumps(d_obj, separators=(',',':'))
content = content[:i] + new_d + content[d_end:]
```
Never use string replacement for D object multi-field edits — format uncertainty causes silent failures.

### RULE 4 — PLATFORM STATUS MAPS ARE FULLY INDEPENDENT
- `TIKTOK_AD_STATUS` → built from TikTok export `Primary status` column ONLY
- `SNAP_AD_STATUS` → built from Snap export ONLY
- `META_AD_STATUS` → built from Meta export ONLY
- **NEVER cross-use.** isTTAdActive must never check Meta/Snap status.

### RULE 5 — NO DUPLICATE JS VARIABLE DECLARATIONS
Before adding ANY `const`/`let` to a JS function, scan the **entire function** for existing declarations of the same name. Duplicate `const` = `Uncaught SyntaxError` = entire dashboard broken. Common collisions: `const last`, `const ds`, `const p`, `const i`.

### RULE 6 — CACHE MUST RESET ON EVERY RENDER
`buildTTAdsHTML` and `buildSnapAdsHTML` must start with:
```js
window.TT_AD_DAILY_CACHE = {};    // in buildTTAdsHTML
window.SNAP_AD_DAILY_CACHE = {};  // in buildSnapAdsHTML
```
Without reset, switching periods (e.g. all-time → last 7 days) leaves stale data in cache, causing hover to show wrong numbers (e.g. 37 sales vs table's 24).

### RULE 7 — HOVER CACHE FROM BREAKDOWN dayAgg ONLY
- `TT_AD_DAILY_CACHE` and `SNAP_AD_DAILY_CACHE` populated inside the breakdown day loop from `dayAgg`
- Same source as the table = hover always matches table
- Also populate in fallback else-branch (when breakdown hidden) using same dayRecs logic
- `getAdDailyData` fallback reads ALL-TIME data — cache must be hit to avoid wrong totals

### RULE 8 — ALWAYS CHECK LAST LINE + JSON VALIDITY
```python
json.loads(content[i:d_end])                          # D object valid ✓
assert content.splitlines()[-1].strip() == '</html>'  # no corruption ✓
```
Use `content.rfind('</script>')` not `find()` — Chart.js CDN tag creates an earlier match.

### RULE 9 — META AD-LEVEL RENDER TRIPLET (new ALL buckets)

When a geo migrates from split (Abaya/Pyjama) to consolidated (ALL) — UAE Apr 27, KU Apr 28, KSA May 7 — **three lines in the Meta ad-level renderer must be updated together**. They are a coupled triplet: the input classifier, the output bucket order, and the day-row filter. Updating one without the others silently hides the geo's data, because data goes into a bucket nobody reads from.

**The three lines (verified May 8 2026):**

| Line | Role | Pattern |
|---|---|---|
| ~1629 | Input: classify ad → `ct` | `const ct = (geo==='Catalogue') ? 'Catalogue' : (geo==='UAE'\|\|geo==='KSA'\|\|geo==='KU / QA / BH') ? 'ALL' : detectCampType(name);` |
| ~1681 | Output: which CT buckets to render per geo | `const campOrder = g.label==='Catalogue' ? ['Catalogue'] : (g.label==='UAE'\|\|g.label==='KSA'\|\|g.label==='KU / QA / BH') ? ['ALL'] : ['Abaya','Pyjama','Catalogue'];` |
| ~1713 | Day breakdown: match ads to day rows | `const nameCT=(adGeo==='Catalogue')?'Catalogue':(adGeo==='UAE'\|\|adGeo==='KSA'\|\|adGeo==='KU / QA / BH')?'ALL':detectCampType(name);` |

All three must list **the same set** of geos in the ALL branch. Any drift = entire geo section disappears from the Ad-level view.

**Failure symptom:** An entire geo (KSA, KU/QA/BH, etc.) is missing from the Meta Ad-level view, even though its data exists in `D.meta.<geo>` and `GRANULAR.meta.adsets["<GEO> ALL - ASC"]`. Cause: line 1629 marks ads with `ct='ALL'` but line 1681's `campOrder` still expects `['Abaya','Pyjama']` → `presentTypes=[]` → `geoSpend=0` → `return` → geo skipped.

**Find the triplet quickly:**
```bash
grep -n "geo==='UAE'\|adGeo==='UAE'\|g.label==='UAE'" /home/claude/index.html
# Should return exactly 3 lines, all with the same set of geos in the ALL branch.
```

**Other render references that already include ALL keys (verify after each migration but don't usually need editing):**
- `META_CAMPS` array (~line 3721) — drill-down config; lists both new and legacy keys
- `aggMetaGeo([...])` calls in metaAdsets (~line 1952, ~line 2103) — already aggregate new + legacy
- Budget Trend chart (~lines 2797-2801) — historical Abaya/Pyjama lines kept for trend continuity (do NOT remove)

### RULE 10 — ACCOUNT-LEVEL RECONCILIATION (catches attribution drift)

Per-ad-sum reconciliation only proves arithmetic — that my pull adds up to itself. It does NOT prove the data is current. Between two pulls of the same date, platforms (especially Google and TikTok) re-attribute conversions and SV; late impressions trickle in for Meta. Without a second cross-check, drift between pulls is invisible until the user spot-checks against Ads Manager.

**Mandatory check before delivery: fire one extra account-level query per platform on TARGET, assert per-ad/per-campaign sum matches account-level total within tolerance.**

```python
# Account-level pull (one row per platform per day)
TOL_SPEND = 5.0     # USD
TOL_CONV  = 1       # absolute
TOL_ATC   = 5       # absolute (Snap commonly has 4-6 unattributed)

# For each platform: pull date-only totals, compare to per-ad sum.
# Meta:    fields=date,cost,offsite_conversions_fb_pixel_purchase,offsite_conversion_value_fb_pixel_purchase,offsite_conversions_fb_pixel_add_to_cart,currency
# TikTok:  fields=date,advertiser_currency,cost,cost_usd,complete_payment,total_complete_payment_rate,web_event_add_to_cart
# Snap:    fields=date,spend,conversion_purchases,conversion_purchases_value,conversion_add_cart,currency
#          (+settings={"action_report_time":"impression","attribution_window":"1_DAY__7_DAY"})
#          (NOTE: from May 14 2026 onward all Snap squads are on 7-day click attribution.
#           Before May 14, only the UAE squad was on 7-day while KSA + KU were on 28-day,
#           but the file pulled May 1-13 with 1_DAY__28_DAY so historical UAE numbers in the
#           file slightly differ from the platform's 7-day truth (~$674 SV net understated
#           over 13 days). Do NOT retroactively repatch — the file represents the pull-time
#           snapshot. From May 14 ingest, use 1_DAY__7_DAY.)
# Google:  fields=Date,Cost,Cost_usd,Conversions,ConversionValue,Currencycode

assert abs(account_spend_usd - my_per_ad_sum_usd) < TOL_SPEND, "DRIFT: re-pull"
assert abs(account_conv      - my_per_ad_conv)    <= TOL_CONV,  "DRIFT: re-pull"
```

**If drift detected:** re-pull the per-ad/per-campaign data fresh, recompute `D.<plat>.<TARGET>` and patch `GRANULAR` ads/adsets/campaigns for that day. Do NOT just adjust top-level `D.<plat>.spend` to match — the per-ad GRANULAR data must agree.

**Confirmed cases:**
- TikTok Apr 30 — 2 conv re-attributed between pulls (in skill history)
- Google May 8 — `PMAX_UAE_Pyjama_EN_goog` gained 2 conv + $1,021 SV between pulls (~30 min apart)
- Meta May 8 — $0.58 late-impression drift across 6 ads

**Snap ATC tolerance specifically:** Snap ad-level rows commonly sum to 4-6 fewer ATCs than account-level (unattributed events). Allow `TOL_ATC=5` for Snap; do NOT re-pull just for ATC delta in this range.

---

### RULE 11 — BACKFILL DRIFT CHECK (prior-7-day re-pull before every delivery)

RULE 10 catches drift on TARGET day only. But TikTok and Snap have 28-day click attribution windows — conversions land 1-7+ days **after** the impression. A day delivered last week is almost always understated by the time of the next delivery. Without a backfill check, the dashboard chronically reports stale ROAS on every prior day.

**Mandatory check before delivery: re-pull account-level for the past 7 days on every platform, compare to file's existing D entries, patch any drift.**

```python
# Pull all of past 7 days in one query per platform (TARGET-7 to TARGET)
# Same field lists as RULE 10 account-level queries.
# Compare each day's row to D.<plat>[day_idx].
# Drift threshold: any conv delta, OR spend delta > $0.10, OR sv delta > $2.
```

**If drift found on a prior day:** re-pull per-ad/per-campaign for that day (use OR'd date filter for batch efficiency: `date == 2026-05-06 OR date == 2026-05-08 OR ...`) and patch D + GRANULAR ads/adsets/campaigns for that day. Same patch flow as RULE 10.

**Confirmed scale of impact (May 1-13, 2026):**
- TikTok: 4 of 13 days drifted, +8 conv, +$1,934 USD SV recovered (~11% of TT period revenue)
- Snap: 6 of 13 days drifted, +19 conv, +$4,113 USD SV recovered (~7% of Snap period revenue)
- Meta: $0.50-$1.50 late-impression spend drift on most prior days (small, but compounds in 7-day rollups)

**Never assume "delivered = final."** Always backfill.

---

### RULE 12 — PRE-DELIVERY LIVE STATUS CHECK (ad active-state matches platform NOW)

The daily export's `ad_status` reflects status during the export window, not current state. An ad that ran all day and was paused at 11pm shows `ACTIVE` in the day's export but is `PAUSED` on the platform now. The dashboard's "Active ads" view must reflect **current platform state**, not "delivered today."

**CRITICAL FIX (the `cost > 0` trap):** A prior version of this rule pulled live status with a `cost > 0` filter and only flipped paused-but-spending ads. This is BROKEN — it can only see ads that spent today. A paused ad with **zero spend today is invisible** to a `cost > 0` pull, so it stays marked `active` in the file forever. This is exactly why stale-active ads (paused Tank ads, dead geo-variants) survived undetected and the user had to catch them from Ads Manager screenshots. **Never derive the active set from a spend-filtered pull.**

**MANDATORY METHOD — per-geo ACTIVE reconciliation (no cost filter):**
For each platform, pull the ads whose **current** status is ACTIVE, scoped per geo/ad-set, then set the file's active set to **exactly** that returned list — nothing more, nothing less. This catches BOTH directions in one shot: stale-active (in file, not on platform) AND missing-active (on platform, not in file).

```python
# Meta — one pull PER GEO (UAE / KSA / KU). Filter on STATUS, not cost:
#   fields=ad_name,ad_set_name,ad_status,cost
#   filter: ad_set_name =@ UAE AND ad_status == ACTIVE      (then KSA, then Ku)
#   date_range_type=last_7_days, compress=true
# TikTok:  fields=ad_name,ad_status,cost_usd     filter: ad_status == AD_STATUS_DELIVERY_OK
# Snap:    fields=ad_name,ad_squad_name,ad_status,spend  filter: ad_status == ACTIVE
# Google:  no ad-level status (campaign-level only)

# Reconcile — the live ACTIVE list IS the truth:
live_active = set(rows_returned_by_status_eq_ACTIVE_pull)   # exclude Followers ad-set ads (user turned off)
for ad in file_status_map:
    file_status_map[ad] = "active" if ad in live_active else "inactive"
# Any live_active ad missing from the map → add it as active.
```

**Exclude Followers ad-set ads** from the active set regardless of platform status — they show ACTIVE and keep spending residually (~$50-110/day) but the user keeps them OFF in the dashboard (standing instruction). Force them inactive.

**Dedupe naming variants BEFORE reconciling:** the same ad can appear under two keys — `Ku_Qa` vs `Ku&Qat`, or single-vs-double-space (`...VO_En_GLO  _Meta` vs `..._En_GLO_Meta`). Use the file's existing key spelling as canonical; delete the stale duplicate rather than carrying conflicting status on two keys. Match on the ad-number prefix + geo, not the full string.

**Why this matters:** "Active ads" view drives the dashboard's most-watched section, and the client compares it directly against their Ads Manager On/Off column. If a paused ad shows there, the user looks bad. The answer to "are the active ads right?" must be "yes, verified per-geo against `ad_status == ACTIVE`" — not "yes, the ones that spent today."

**Confirmed cases:**
- May 13: `Combo_En_GLO  _Meta_UAE_Sales_ASC` had $683 spend then was paused — export said ACTIVE, platform said PAUSED. Flipped to inactive.
- May 20: stale-active ads invisible to the old `cost > 0` check — 522_Tank UAE + 526_Tank KU (Meta, paused, $0 today) and 5 TikTok geo-variants (515 Ku_Qa, 521 UAE, 523 KSA, 523 Ku_Qa, 528 Ku_Qa) stayed `active` in file until the user flagged it from screenshots. Root cause: spend-filtered status pull. Fix: per-geo `ad_status == ACTIVE` reconciliation (this rule). Also removed stale duplicate key `517_Abaya..._Ku_Qa_Sales_ASC` (old naming vs current `Ku&Qat`).
- May 21 verified: Meta UAE 7 / KSA 7 / KU 6 active, all reconciled exactly to user's Ads Manager screenshots via per-geo ACTIVE pull.

---

## STEP 0 — File Date Validation
Check each file matches the expected date before processing:
- TikTok: filename contains date range
- Snap: timestamp in filename
- Meta: `Reporting starts` / `Reporting ends` columns
- Google: header row date range (row 2 after skipping)

Flag mismatch → wait for confirmation.

---

## STEP 1 — Read Files

| Platform | Format | Encoding | Key col |
|---|---|---|---|
| Meta | CSV | UTF-8 | `Amount spent (USD)` |
| TikTok | XLSX | — | `Cost` (AED) |
| Snap | XLSX | — | `Amount Spent` (USD) |
| Google | CSV | **UTF-16, tab-sep, skip 2 rows** | `Cost` (AED) |

---

## STEP 2 — Currency

| Platform | Rule |
|---|---|
| TikTok | ÷ 3.67 |
| Google | ÷ 3.67 |
| Meta | Use as-is (USD already) |
| Snap | Use as-is (USD already) |

---

## STEP 3 — Geo Detection
Check Ad Name first → fall back to Ad Set / Ad Group name.

| Key | Patterns |
|---|---|
| `uae` | `UAE`, `_UAE_` |
| `ksa` | `KSA`, `_KSA_` |
| `ku` | `Ku & Qa`, `KW/QA/BH`, `KUW`, `QAT`, `KWT`, `Ku/Qa/Bh` |
| `cat` | `Catalogue`, `Catalog`, `Retarget`, `GCC` |
| `followers` | `Brand-Social-Followers`, `Brand`, `Awareness` (Meta only) |

**Snap KU special:** KU ads often have NO geo in Ad Name — must check `Ad Set Name` for `KW/QA/BH`.

---

## STEP 4 — Platform Processing

### TIKTOK
- Filter: `Primary status == 'Active'` rows only
- Spend: `Cost` ÷ 3.67 | SV: `Purchase value (website)` ÷ 3.67
- Conv: `Purchases (website)` | ATC: `Adds to cart (website)`
- Geo from Ad Name → Ad Group name

### SNAP
- Filter: Active ads only
- Spend/SV already USD: `Amount Spent`, `Purchases Value`, `Purchases`, `Adds To Cart`
- Geo from Ad Name → Ad Set Name (check Ad Set for `KW/QA/BH`)
- **Snap export scope varies** — sometimes full-account (all 12 ads UAE+KSA+KU in one file, e.g. Apr 16), sometimes UAE-only (4 ads). Always check the row count and Ad Set Names to determine scope before processing.
- When single-file full-account: process all ads in one pass. When multi-file (UAE + separate KSA/KU files): combine before computing totals.
- Same ad name can appear in multiple geos (e.g. `279_Spring0325_...` runs in both KSA and KU). Use `(name, geo)` as the unique dedup key — NEVER just `name`, or you'll collapse rows and lose spend.

### META
See `references/meta.md`. Key points:
- Separate Followers rows before any calculation
- `D.meta.spend` = sales + followers total
- `D.meta.roas` = sales_sv ÷ total_spend
- Geo `spend` = sales-only spend per geo

### GOOGLE
See `references/google.md`. Always use highest of 3 SV methods.

**CRITICAL filter**: `g = g[g['Campaign status'] == 'Enabled']`. The CSV includes 8+ total/summary rows (`Total: Campaigns`, `Total: Account`, `Total: Performance Max`, `Total: Search`, `Total: Shopping`, `Total: Demand Gen`, `Total: In-stream video`, `Total: Display`) which will DOUBLE the spend if not filtered. Filtering by `startswith('Total')` is fragile — use `Campaign status == 'Enabled'` instead.

**GCC PMAX split**: `PMAX_UAE_KSA_KW_QA_Abaya_EN_goog` spans all 3 geos. Split its spend/conv/sv proportionally across UAE/KSA/KU using each geo's **SEM-only spend ratio** (SEM campaigns, not PMAX). Example Apr 16: SEM totals UAE $127.79 / KSA $12.17 / KU $65.80 → UAE gets 62.11%, KSA 5.91%, KU 31.98% of the split campaign's $173.86.

**Conv rounding**: Per-geo conv values after split will be fractional. Round to nearest int while preserving the grand total. Apr 16: UAE 15.73→16, KSA 0.35→0, KU 1.92→2 (sum=18 ✓).

---

## STEP 5 — CPO Sanity Check

| Platform | Valid range |
|---|---|
| Meta | $30–$220 |
| TikTok | $50–$280 |
| Snap | $10–$95 |
| Google | $30–$180 |

CPO=0 with spend>$100 → wrong metric (ATC used instead of purchases). Fix before inserting.

---

## STEP 6 — Update D Object
Use JSON parse + re-serialize (see Rule 3). See `references/d-object-structure.md` for exact formats.

---

## STEP 7 — Update GRANULAR

### Key architecture — TikTok & Snap
- `seenDates` dedup across ALL keys (permanent + dated `_marXX`/`_aprXX`)
- Dated keys hold real daily uploaded data — **NEVER delete them**
- **NEVER create fake permanent keys** with invented data
- New ad = data only from the day it first appears in the export
- ATC must come from the export — never zero it out or invent it

### Snap GRANULAR specifics
- `SNAP_AD_STATUS` keys = base key after stripping `_marXX`/`_aprXX`/`_vN`
- `makeSnapDn` handles `UAE_Snap`/`UAE_Snapchat` (geo before platform) with special regex
- `perfRowSignalOther` always needs geo param `g.label` for `data-geo` hover filtering

### Meta GRANULAR adset keys (exact — 8 keys, verified Apr 16)
```
"UAE Abaya - ASC"
"KSA Abaya - ASC"
"KU Abaya - ASC"
"UAE Pyjama - ASC"
"KSA Pyjama - ASC"
"KU Pyjama - ASC"
"Catalogue Retargeting / Prospecting - GCC"
"Followers GCC - Brand"
```
Budgets were shifted Apr 7 (Abaya→Pyjama in UAE/KU) and Apr 16 (UAE Abaya $500/Pyjama $500, KSA Abaya $600/Pyjama $300, KU Abaya $200/Pyjama $450). Abaya and Pyjama are SEPARATE adset keys — the old "UAE Ongoing - ASC - En" combined key no longer exists.

### Meta export — Ad Set names to GRANULAR key mapping
```python
def map_adset(ad_set_name):
    n = ad_set_name.upper()
    if 'CATALOG' in n: return 'Catalogue Retargeting / Prospecting - GCC'
    if 'FOLLOWERS' in n or 'BRAND-SOCIAL' in n: return 'Followers GCC - Brand'
    if 'UAE' in n and 'ABAYA' in n: return 'UAE Abaya - ASC'
    if 'UAE' in n and 'PYJAMA' in n: return 'UAE Pyjama - ASC'
    if 'KSA' in n and 'ABAYA' in n: return 'KSA Abaya - ASC'
    if 'KSA' in n and 'PYJAMA' in n: return 'KSA Pyjama - ASC'
    if ('KUW' in n or 'KW/QA' in n) and 'ABAYA' in n: return 'KU Abaya - ASC'
    if ('KUW' in n or 'KW/QA' in n) and 'PYJAMA' in n: return 'KU Pyjama - ASC'
```

---

## STEP 8 — Update ALL_DATES
```js
const ALL_DATES = ["Mar 1",...,"Apr 15","Apr 16"]; // append new date
```
Current tail: `...,"Apr 15","Apr 16"]`

---

## STEP 9 — Budget Change Markers (AUTO)
```js
{"date":"Apr 5","platform":"tiktok","direction":"up"}
```
Apply for all 4 platforms whenever spend changes meaningfully vs previous day.

---

## STEP 10 — Validate & Deliver

Full pre-delivery checklist (run every time):
```python
# 1. D object parses
json.loads(content[i:d_end])

# 2. TARGET spend in D matches each platform's per-ad export total
for pk in ['meta','tiktok','snap','google']:
    for rec in d_obj[pk]:
        if rec['d'] == new_date:
            assert abs(rec['spend'] - expected[pk]) < 0.01

# 3. File ends cleanly
assert content.rstrip().splitlines()[-1].strip() == '</html>'

# 4. No multi-commas anywhere
assert not re.search(r',{2,}', content)

# 5. node --check on extracted script block
s = content.find('<script>')
e = content.rfind('</script>')  # rfind not find!
with open('/tmp/eval_test.js','w') as f:
    f.write('(function(){\n' + content[s+8:e] + '\n})();')
# subprocess: node --check /tmp/eval_test.js → returncode 0

# 6. ALL_DATES tail includes new date
# 7. GRANULAR has TARGET entries (should be ~70+ across meta/tiktok/snap/google ads)
# 8. SNAP_AD_STATUS sanity: each new key's base name present and "active"

# 9. ACCOUNT-LEVEL RECONCILIATION (RULE 10) — fire date-only query per platform on TARGET
#    and confirm per-ad sum matches account-level within tolerance.
#    If drift detected, re-pull and patch D + GRANULAR for that day before delivering.
for pk in ['meta','tiktok','snap','google']:
    account = pull_account_level(pk, target)
    assert abs(account['spend_usd'] - d_obj[pk][-1]['spend']) < 5.0
    assert abs(account['conv'] - d_obj[pk][-1]['conv']) <= 1
    # Snap ATC: tolerance 5 (unattributed events expected)

# 10. RENDER TRIPLET CHECK (RULE 9) — three coupled lines must list same geos:
import re
pat = r"(geo|adGeo|g\.label)===['\"](UAE|KSA|KU / QA / BH|Catalogue)['\"]"
hits = [(m.group(1), m.group(2)) for m in re.finditer(pat, content)]
# Each of geo, adGeo, g.label should reference UAE+KSA+KU (3 each = 9 total minimum
# in the ALL branch, plus Catalogue handling). Run `grep -n` and eyeball if uncertain.

# 11. ACTIVE META ADSET BUCKETS REFERENCED — verify each bucket with TARGET spend > 0
#     in GRANULAR.meta.adsets is referenced ≥2x in render code (RULE 13 from prior sessions).

# 12. BACKFILL DRIFT CHECK (RULE 11) — re-pull past 7 days account-level per platform
#     and patch any drift in D + GRANULAR before delivering.
for pk in ['meta','tiktok','snap','google']:
    history = pull_account_level_range(pk, target - 7, target)
    for day, row in history.items():
        d_entry = next((e for e in d_obj[pk] if e['d']==day), None)
        if not d_entry: continue
        if abs(row['spend_usd'] - d_entry['spend']) > 0.10 \
           or row['conv'] != d_entry['conv'] \
           or abs(row['sv_usd'] - d_entry['sv']) > 2.0:
            patch_day(pk, day, row)  # re-pull per-ad and patch D + GRANULAR

# 13. PRE-DELIVERY LIVE STATUS CHECK (RULE 12) — per-geo ACTIVE reconciliation.
#     DO NOT filter by cost — a paused $0-spend ad is invisible to cost>0 and
#     stays stale-active. Pull by ad_status == ACTIVE, scoped per geo, and set
#     the file's active set to EXACTLY the returned list (both directions).
# Meta: one pull per geo. filter: ad_set_name =@ <UAE|KSA|Ku> AND ad_status == ACTIVE
# TikTok: filter ad_status == AD_STATUS_DELIVERY_OK ; Snap: ad_status == ACTIVE
for pk in ['meta','tiktok','snap']:                  # google: no ad-level status
    live_active = pull_live_active_set(pk)           # by STATUS, not cost
    live_active -= followers_adset_ads               # user keeps Followers OFF
    dedupe_naming_variants(status_map[pk], live_active)  # Ku_Qa vs Ku&Qat, spaces
    for ad in list(status_map[pk]):
        status_map[pk][ad] = 'active' if ad in live_active else 'inactive'
    for ad in live_active:                           # missing-active → add
        status_map[pk].setdefault(ad, 'active')
```
Copy to `/mnt/user-data/outputs/index.html` (or `ook.html` per current convention) and call `present_files`.

### STEP 0.3b — Account-level cross-check (per RULE 10)

Run this **during ingest**, not just at validate-time, so attribution drift is caught before any GRANULAR mutation work is wasted.

After per-ad/per-campaign queries return, fire one extra date-only query per platform for TARGET. Compare:

| Platform | Per-ad sum | Account-level | Tolerance |
|---|---|---|---|
| Meta | sum cost across all rows | account `cost` (USD) | $5 spend, 1 conv |
| TikTok | sum cost_usd (DELIVERY_OK rows) | account `cost_usd` | $5 spend, 1 conv |
| Snap | sum spend ACTIVE rows | account `spend` | $5 spend, 1 conv, 5 ATC |
| Google | sum cost_usd campaigns | account `Cost_usd` | $5 spend, 1 conv |

If a platform fails: re-pull per-ad/per-campaign data fresh (the older pull is stale), recompute D entry, patch GRANULAR ads/adsets/campaigns for that day. Patch META_KU_COUNTRY too if Meta drifted.

---

## DASHBOARD JS ARCHITECTURE

### Three render functions — ALL must be updated for any UI change:
| Function | When called |
|---|---|
| `renderPlat(pk, d)` | Single day selected |
| `renderPlatRange(pk, rd)` | Date range selected |
| `renderPlatMTD(pk)` | MTD button active |

### KPI boxes
- `g6` CSS class = 6 boxes: ROAS · SPEND · SALES VALUE · SALES · ATC · CPO
- `g5` CSS class = 5 boxes: Google only (no ATC)
- Conditional: `...(pk!=='google'?[{l:"ATC",...}]:[])` in all 3 render functions

### MTD definition
- MTD = **current calendar month only** (Apr 1–Apr 4, NOT Mar 1–Apr 4)
- Filter: `r.d.startsWith(curMon)` where `curMon = last.replace(/ \d+$/, '')`
- `renderOverviewMTD` and `renderPlatMTD` must both apply this filter
- MTD pill label: `"MTD Apr 1 – Apr 4"`
- toggleMTD: `calPickStart = "${curMon} 1"`

### Hover graph cache
- `buildTTAdsHTML` and `buildSnapAdsHTML` reset cache at start: `window.XX_CACHE = {}`
- Cache populated from `dayAgg` inside breakdown loop (same source as table)
- Also populated in else-branch (breakdown hidden) using same dayRecs logic
- `getAdDailyData(baseName, geo, platform)` — platform param isolates each platform's data

### seenDates pattern (TikTok & Snap merged build)
```js
const ttSeenDates = {};
adNames.forEach(name => {
  if(!isTTAdActive(name)) return;
  const geo = detectTTGeo(name); if(!geo) return;
  const dn = makeTTDn(name);
  const seenKey = `${geo}||${dn}`;
  if(!ttSeenDates[seenKey]) ttSeenDates[seenKey] = new Set();
  const dailyRecs = ds.map(d => {
    if(ttSeenDates[seenKey].has(d)) return null;
    return (ads[name]||[]).find(x=>x.d===d)||null;
  }).filter(Boolean);
  if(dailyRecs.length > 0) {
    dailyRecs.forEach(r => ttSeenDates[seenKey].add(r.d));
    // aggregate and merge...
  }
});
```

---

## ACTIVE ADS REFERENCE (Apr 16, 2026)

### TikTok active ads (from Apr 16 export, 12 ads, $780.96 total)
- UAE: 515, 510, 511
- KSA: 515, 499, 497
- KU/QA/BH: 520, 515, 512
- GCC/Retargeting: 487, 486, 467

### Snap active ads (from Apr 16 export, 12 ads, $699.88 total)
- UAE: 493, 486, 515, 512  (note: 486 was "inactive" in STATUS — had to flip to active)
- KSA: 279, 487, 451, 510  (note: 279_Snap_KSA base was MISSING from STATUS — had to add)
- KU/QA/BH: 501, 502, 279, 508  (note: 279_Snap_KU was "inactive" — had to flip to active)

### Google campaigns (9 enabled, Apr 16 total $885.31)
- PMAX_KSA_Pyjama, PMAX_UAE_Pyjama, PMAX_UAE_KSA_KW_QA_Abaya (the GCC split)
- SEM_UAE_Generic/Brand/Abaya, SEM_KUW-QAT_Generic/Brand, SEM_KSA_Brand

### Meta active ads (Apr 16, 29 ads, $3110.61 total)
- UAE: 502, 506, 510, 511, 512, 513, 515, 516
- KSA: 499, 502, 506, 510, 512, 513, 514, 515
- KU: 498, 506, 510, 513, 514, 517, 518, 519
- Catalogue: EN_All, EN_Men, EN_Women (all _Catalogue suffix — different from old _Prospecting Catalogue variant)
- Followers: 3 adsets (UAE, KSA, KU)
- Bare-name UAE ads (no Meta suffix): 515_Cp_GLO, 518_Cp_GLO

---


---

## GRANULAR STRUCTURE NOTES

### GRANULAR object top-level layout
```
const GRANULAR = {
  meta: { budgets:{...}, adsets:{...}, ads:{...} },
  tiktok: { adsets:{...}, ads:{...} },    // inside GRANULAR, not separate
  snap: { adsets:{...}, ads:{...} },
  google: { campaigns:{...} }
}
```
- `meta` section is at chars ~0 of GRANULAR body
- `tiktok` section found via `content.find('tiktok:{adsets:', 380000)`
- `snap` section found via `content.find('snap:{adsets:', 440000)`
- `google` section found via `content.find('google:{', snap_start)`
- `tiktok ads:{` found via `content.find('},ads:{', tt_start, snap_start)`
- `snap ads:{` found via `content.find('},ads:{', snap_start, google_start)`

### Snap GRANULAR: new daily keys (NOT appending to existing)
Each day's Snap data goes into a **new dated key** `_aprXX` (e.g. `_apr15`), never appended to a permanent key. Insert before the closing `}` of the snap `ads:{...}` block.

### TikTok GRANULAR: append to existing keys
TikTok daily data appends a new `{"d":"Apr 15",...}` entry to the existing permanent ad key array. Find the key, walk to array closing `]`, insert before it.

### TikTok ad name → GRANULAR key: FUZZY MATCHING REQUIRED
TikTok export ad names often **differ slightly** from the stored GRANULAR key (e.g. export says `..._Tiktok_UAE` but key is `..._Tiktok_UAE_Sales`, or export says `..._Tiktok_GCC_Retargeting` but key is `..._Tiktok_GCC_Retarge` — truncated). For 512 Ku there are two candidate keys (`_Ku/Qa/Bh_Sales` and `_Ku & Qa_Sales_ASC`) — pick the one with most recent data (check first entry's `d` field; later = more recently used). Build an explicit export→key map for today's ads rather than relying on exact match.

### Meta GRANULAR ads: 3-bucket insertion pattern
Each Meta ad from today's export falls into one of three buckets:
1. **Existing plain key** — ad name exists as-is in GRANULAR → prepend new `{"d":"Apr N",...}` as first array entry
2. **Append to existing `_apr13` (or earlier) dated key** — ad was introduced recently as a dated key and has since accumulated entries → find the dated key, prepend new entry to its array
3. **Brand new ad today** — create a new `{ad_name}_aprN` key with a single-entry array, insert before closing `}` of meta.ads block

Always verify total ads processed = total unique ad names in export, and sum of per-ad spends = export total spend. This catches bucket-assignment errors immediately.

### Snap STATUS map auto-fix check (CRITICAL)
After building Apr N Snap entries, for each new `_aprN` key compute `makeSnapDn` base name and verify it exists in `SNAP_AD_STATUS` as `"active"`. Three common problems:
- **Base name MISSING from STATUS** → ad won't render at all. ADD it as `active`.
- **Base name marked `inactive`** but spent money today → ad won't render. FLIP to `active`.
- **Duplicate entries** for same base name → duplicates can override each other. Scan for duplicates.

Example from Apr 16 session: `279_Snap_KSA` was missing entirely (added), `279_Snap_KU` was inactive (flipped), `486_Snap_UAE` was inactive (flipped). Without these fixes, 3 ads with real spend would not appear in the dashboard.

### JS syntax validation method
`node --check` does NOT work on `.html` files (ERR_UNKNOWN_FILE_EXTENSION). Instead:
```python
# Extract script block
s = content.find('<script>')
e = content.rfind('</script>')   # rfind not find — Chart.js CDN creates earlier match
with open('/tmp/eval_test.js','w') as f:
    f.write('(function(){\n' + content[s+8:e] + '\n})();')
# Run node — "document is not defined" = browser API only = CLEAN
# Any other error = real JS syntax problem
```

### Finding syntax errors in minified data blobs
Binary search within a single giant line using `node /tmp/snippet.js` on truncated prefixes with depth-aware closing braces. Track `{`/`}` depth char-by-char (respecting strings) to always close with the right number of `}`.

### Known past bugs to watch for
- Unquoted object keys: `{d:"Apr 13",...}` — all data keys must be quoted `{"d":"Apr 13",...}`
- Multiple commas: `}],,,,"key":` — caused by bad string insertion; always check for `re.finditer(r',{2,}', content)`

## FILE BACKUP
If corruption occurs: `https://3rdscreen.github.io/ook-dashboard`

---

## SESSION LOG (most recent first)

### May 14 2026 (PM) — Snap account-wide attribution window flipped to 7-day
**Context:** Until May 13, only the UAE Retargeting squad used 7-day click attribution; KSA and KW/QA/BH squads were on 28-day. The OOK pipeline used `1_DAY__28_DAY` for everything, which matched KSA + KU's settings and over-reported UAE squad slightly (UAE squad over-reported ~+0 conv net but specific days off by 1-2 conv; net $674 SV understated across May 1-13 due to platform reporting quirks).

**Starting May 14 ingest onward**: ALL Snap squads (UAE + KSA + KW/QA/BH) are on 7-day click attribution. Pipeline must use `1_DAY__7_DAY` going forward.

**Do NOT retroactively repatch May 1-13.** The file's existing numbers represent legitimate pull-time snapshots under the squads' settings at the time. Repatching would create more drift confusion than it solves; UAE-squad differences are within $674 net over 13 days.

**Critical**: The `attribution_window` setting in Supermetrics queries OVERRIDES per-squad Ads Manager settings — it forces all squads in the response to use the specified window. So under mixed-squad-attribution (pre-May 14), there's no clean way to pull "match each squad's native setting" in one query. From May 14 the squads are uniform, so `1_DAY__7_DAY` matches Ads Manager exactly for every squad.

### May 14 2026 — Two new rules forced by repeated drift + user pushback
1. **Backfill drift discovered at scale**: May 1-13 audit showed TikTok and Snap chronically understated by 1-7 days of late-attribution conversions. TikTok: 4/13 days drifted (+8 conv, +$1,934 SV). Snap: 6/13 days drifted (+19 conv, +$4,113 SV). Total $6K recovered across 27 hidden conversions. **Mitigation: RULE 11 (backfill drift check) — re-pull past 7 days on every platform before every delivery, patch any drift.**
2. **Active-ad status drift caught by user**: `Combo_En_GLO  _Meta_UAE_Sales_ASC` had $683 spend on May 13 then was paused. May 13 export said ACTIVE, current platform state was PAUSED. Dashboard's "Active ads" view would have shown a paused ad. **Mitigation: RULE 12 (pre-delivery live status check) — pull current ad_status from each platform with `cost > 0` filter, flip any paused-on-platform-but-active-in-file ads to inactive before delivery.**
3. **Confirmed B2B/wholesale pattern on Google SEM_UAE_Brand**: 3 consecutive days (May 10-12) of huge ROAS — $16,549 + $8,496 + $6,481 USD on $98 spend = 321× combined. Verified real-as-reported by account-level Google. By May 13 returned to normal ($337 SV / 12.9×).
4. **Whitespace variants in ad names**: Some Supermetrics queries normalize double-spaces to single-spaces depending on filter clause. When auditing RULE 12, use file's existing key spelling as canonical; do NOT add duplicate variants.

### May 8 2026 — Two bugs caught by user pushback
1. **Google attribution drift**: between first pull and verify pull (~30 min apart), `PMAX_UAE_Pyjama_EN_goog` gained 2 conv + $1,021 SV. First delivered Google: $767.56/10/$2828.07 → truth: $771.39/12/$3106.01. Same drift class as TikTok Apr 30. **Mitigation: RULE 10 + STEP 0.3b account-level cross-check now mandatory.**
2. **Meta ad-level render: KSA and KU/QA/BH disappeared.** First "fix" only updated line 1629 (input classifier). Lines 1681 (campOrder) and 1713 (day-row filter) still expected `['Abaya','Pyjama']` → `presentTypes=[]` → entire geo skipped. **Mitigation: RULE 9 (render triplet) — all three lines must be updated together.**
3. **Confirmed consolidation timeline:**
   - UAE → ALL on Apr 27 (Pyjama merged abaya in)
   - KU → ALL on Apr 28 (renamed in place)
   - KSA → ALL on May 7 (new bucket created, legacy Abaya/Pyjama frozen)
4. **TikTok ad-group rotation:** `KSA - May 26 - Interest 1.0 + iOS` is now the live KSA ad group; `KSA - Jan 26` paused. Same `KSA Interest - IOS` rollup bucket — no render code change needed.

### Apr 17 2026 — Prior session (snapshot of skill before this update)
See git history of `index.html` for granular context.

---

## REFERENCE FILES
- `references/meta.md` — Followers separation, geo, GRANULAR adset keys
- `references/google.md` — SV method selection, campaign filtering
- `references/d-object-structure.md` — Exact D object entry formats
- `references/historical-context.md` — Data pipeline, back-calc method, ROAS ranges
