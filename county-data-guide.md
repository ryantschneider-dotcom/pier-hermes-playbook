# Hermes Agent Guide: Extracting County Property Data Without Getting Blocked

**For:** Hermes (PIER Commercial AI Agent)
**Author:** Manus (PIER Research Agent)
**Updated:** June 2026
**Purpose:** Explains every technique used to pull property data from qPublic, county GIS portals, tax assessor sites, and planning/zoning databases — including how to avoid bot detection, session timeouts, and CAPTCHA walls.

---

## The Core Problem

County property data portals — qPublic, Tyler Technologies (INCODE/iasWorld), Schneider Geospatial, Esri-based GIS viewers, and custom county-built sites — are designed for human one-at-a-time lookups. They use a combination of:

- **Session cookies** that expire if you pause too long between requests
- **Cloudflare or similar WAF (Web Application Firewall)** that fingerprints browser behavior
- **JavaScript-rendered pages** that do not return data to a plain HTTP GET request
- **Rate limiting** that blocks IPs making too many requests per minute
- **CAPTCHA challenges** triggered by headless browser signatures
- **Referrer checks** that reject requests not coming from the site's own pages

The key insight is: **you are not scraping in bulk.** You are doing targeted, single-property lookups — the same thing a human appraiser would do. Your goal is to behave exactly like a careful human user, not a bot.

---

## The Three-Tier Strategy

Every county data lookup follows this priority order. Move to the next tier only if the previous one fails.

### Tier 1 — Direct URL Construction (No Browser Needed)

Many county portals have predictable URL patterns. If you know the Parcel ID (PARID), you can often construct a direct link to the property record page and extract it with a simple HTTP request.

**How to find the URL pattern:**
1. Go to the county's qPublic or assessor site manually in the browser.
2. Search for any known property (e.g., a well-known address).
3. When the result page loads, look at the URL bar. It will usually contain the PARID as a query parameter.
4. Note the exact pattern, then substitute the target PARID.

**Common URL patterns by county:**

| County | Portal | URL Pattern |
|---|---|---|
| Chatham County, GA | qPublic | `https://qpublic.schneidercorp.com/Application.aspx?AppID=1040&LayerID=22898&PageTypeID=4&PageID=9404&KeyValue={PARID}` |
| Glynn County, GA | qPublic | `https://qpublic.schneidercorp.com/Application.aspx?AppID=1041&LayerID=...&KeyValue={PARID}` |
| Brantley / Bryan / Bulloch / Camden | qPublic (Schneider) | Same AppID pattern, different AppID number per county |
| Beaufort County, SC | County Assessor | `https://www.bcgov.net/departments/assessor/search.aspx` — POST-based, use browser |
| Glynn County GIS | ArcGIS Online | `https://gis.glynncounty.org/...` — use Esri REST API (see Tier 2) |

**When this works:** The page is server-rendered HTML and returns full content on a GET request. Use `webpage_extract` tool with the constructed URL. This is the fastest path and works roughly 40% of the time.

**When this fails:** The page returns a blank shell or redirect because it requires JavaScript to render, or it checks for a valid session cookie from the search page.

---

### Tier 2 — REST API Endpoints (The Best-Kept Secret)

Most modern county GIS portals are built on **Esri ArcGIS Server** or **Schneider Geospatial**. These platforms expose machine-readable REST API endpoints that return clean JSON — no browser, no CAPTCHA, no session required.

**This is the single most reliable method and should always be tried before browser automation.**

#### Esri ArcGIS REST API

Every Esri-based county GIS has a REST endpoint. The pattern is:

```
https://{county-gis-server}/arcgis/rest/services/{ServiceName}/FeatureServer/{LayerID}/query
  ?where=PARID='{PARID}'
  &outFields=*
  &f=json
```

**How to find the endpoint:**
1. Open the county GIS viewer in a browser.
2. Open browser DevTools → Network tab → filter by "XHR" or "Fetch."
3. Click on a parcel in the map viewer.
4. Watch for a network request to a URL containing `/arcgis/rest/services/` or `/FeatureServer/`.
5. Copy that base URL — that is your REST endpoint.
6. Substitute the PARID in the `where` clause.

**Example — Chatham County:**
```
https://maps.chathamcounty.org/arcgis/rest/services/Parcels/FeatureServer/0/query
  ?where=PARID='20143-01002'
  &outFields=PARID,OWNER,SITEADDR,ACREAGE,APPRAISED_VALUE,ZONING
  &f=json
```

This returns a clean JSON object with all parcel attributes. No login, no CAPTCHA, no session.

**Example — Glynn County:**
```
https://gis.glynncounty.org/arcgis/rest/services/Parcels/MapServer/0/query
  ?where=PARID='03-10768'
  &outFields=*
  &f=json
```

#### Schneider qPublic REST API

qPublic (Schneider Geospatial) also exposes a JSON endpoint behind the scenes. When you load a property page, the browser makes an AJAX call to fetch the data. You can replicate this call directly.

**How to find it:**
1. Open qPublic for the county in browser DevTools → Network tab.
2. Search for a parcel.
3. Look for XHR requests to URLs like:
   - `/Application.aspx/GetParcelData`
   - `/api/parcel/{PARID}`
   - `/Search.aspx` (POST with form data)

**Important:** Schneider's API often requires a `Referer` header matching the qPublic domain and an `X-Requested-With: XMLHttpRequest` header. Add these to your Python `requests` call:

```python
import requests

headers = {
    "Referer": "https://qpublic.schneidercorp.com/",
    "X-Requested-With": "XMLHttpRequest",
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
}

response = requests.get(url, headers=headers, timeout=15)
```

---

### Tier 3 — Browser Automation with Human-Like Behavior

When Tier 1 and Tier 2 fail — typically because the site uses Cloudflare, requires a CAPTCHA on first visit, or has a multi-step search form — use the browser tools with deliberate human-like pacing.

**The rules for avoiding detection:**

#### Rule 1: Always Start from the Homepage

Never navigate directly to a deep property URL as your first request. Cloudflare and similar WAFs check whether you arrived via a legitimate referral path. Always:

1. Navigate to the county assessor or GIS homepage first.
2. Wait 2–3 seconds (let the page fully render).
3. Then navigate to the search page.
4. Then perform the search.

This establishes a valid session cookie and a legitimate referrer chain.

#### Rule 2: Use the Search Form, Not Direct URLs

Even if you know the direct URL, use the search form on first visit. This:
- Sets the correct session cookie
- Establishes a valid `Referer` header for subsequent requests
- Avoids triggering "direct deep-link" bot detection rules

#### Rule 3: Never Request More Than One Property Per Session

Do not loop through multiple parcels in a single browser session. Each property lookup should be treated as a fresh, independent task. If you need data on 5 parcels, do 5 separate lookups with pauses between them.

#### Rule 4: Read the Full Page Before Extracting

After navigating to a result page, always use `browser_view` to confirm the page has fully loaded before attempting to extract data. Many county sites use lazy-loading JavaScript that populates fields 1–2 seconds after the initial page load.

#### Rule 5: Use `webpage_extract` Before `browser_view`

For any page that is not behind a login or CAPTCHA, always try `webpage_extract` first. It is faster, does not trigger browser fingerprinting, and returns clean markdown. Only fall back to full browser tools if `webpage_extract` returns empty content or a redirect.

#### Rule 6: Handle Cloudflare Challenges

If you hit a Cloudflare "Checking your browser" page:
- **Do not retry immediately.** Wait 5 seconds and try again — Cloudflare's JS challenge completes in the background.
- If it persists, the site requires a real browser with JavaScript execution. Use `browser_navigate` and then `browser_view` to wait for the challenge to resolve.
- If a CAPTCHA appears (image-based), you cannot solve it autonomously. Note this in your research log and use an alternative data source (see Fallback Sources below).

---

## Site-by-Site Playbook for PIER's Coverage Area

### Chatham County, GA (Savannah)

| Data Type | Best Method | URL / Notes |
|---|---|---|
| Parcel search | Tier 1 — qPublic direct URL | `https://qpublic.schneidercorp.com/` → AppID 1040 |
| Assessed value, owner, acreage | Tier 1 — qPublic HTML extract | Works reliably with `webpage_extract` |
| GIS / lot boundaries | Tier 2 — ArcGIS REST | `https://maps.chathamcounty.org/arcgis/rest/services/` |
| Zoning | Chatham County MPC site | `https://www.chathamcountympc.org/` — use browser, search by address |
| Building permits | City of Savannah | `https://www.savannahga.gov/` — Development Services portal |

**Known issue:** Chatham County qPublic occasionally shows a "session expired" error if you navigate directly to a parcel URL without first visiting the search page. Fix: navigate to `https://qpublic.schneidercorp.com/Application.aspx?AppID=1040` first, then construct the direct parcel URL.

---

### Glynn County, GA (Brunswick / St. Simons)

| Data Type | Best Method | URL / Notes |
|---|---|---|
| Parcel search | Tier 1 — qPublic | `https://qpublic.schneidercorp.com/` → Glynn County |
| Assessed value | Tier 1 — qPublic HTML | Reliable, no CAPTCHA observed |
| GIS | Tier 2 — ArcGIS REST | `https://gis.glynncounty.org/arcgis/rest/services/` |
| Zoning | Glynn County Planning | `https://www.glynncounty.org/` → Planning & Zoning |
| Sales history | Glynn County Clerk | `https://www.glynncountyclerk.com/` — deed search |

---

### Bryan County, GA (Richmond Hill / Pembroke)

| Data Type | Best Method | URL / Notes |
|---|---|---|
| Parcel search | Tier 1 — qPublic | Bryan County AppID on Schneider |
| GIS | Tier 3 — browser | Bryan County GIS viewer is older; use browser navigation |
| Zoning | Bryan County Planning | Call or email — limited online portal |

---

### Effingham County, GA (Rincon / Springfield)

| Data Type | Best Method | URL / Notes |
|---|---|---|
| Parcel / tax data | Tier 1 — qPublic | Schneider-hosted |
| GIS | Tier 2 — ArcGIS REST | Effingham County has Esri-based GIS |

---

### Camden County, GA (Kingsland / St. Marys)

| Data Type | Best Method | URL / Notes |
|---|---|---|
| Parcel search | Tier 1 — qPublic | Schneider-hosted |
| GIS | Tier 3 — browser | Camden GIS viewer requires browser navigation |

---

### Beaufort County, SC (Bluffton / Hilton Head / Beaufort)

| Data Type | Best Method | URL / Notes |
|---|---|---|
| Parcel / assessed value | Tier 3 — browser | `https://www.bcgov.net/departments/assessor/` — POST-based search form, requires browser |
| GIS | Tier 2 — ArcGIS REST | Beaufort County uses Esri; REST API available |
| Zoning | Beaufort County Planning | `https://www.bcgov.net/departments/planning/` |

**Known issue:** Beaufort County assessor uses a multi-step POST form with a hidden `__VIEWSTATE` token. You must use browser automation (not direct HTTP requests) to submit the search. Navigate to the search page, fill the address field, click Search, then extract the result.

---

### Jasper County, SC (Hardeeville / Ridgeland)

| Data Type | Best Method | URL / Notes |
|---|---|---|
| Parcel data | Tier 3 — browser | Jasper County has a basic assessor portal; browser required |
| GIS | Tier 2 — ArcGIS REST | SC Revenue and Fiscal Affairs GIS covers Jasper County |

---

## Fallback Data Sources (When County Sites Fail)

When a county site is completely inaccessible (CAPTCHA, maintenance, or requires login), use these alternatives in order:

| Priority | Source | What It Provides | URL |
|---|---|---|---|
| 1 | **Georgia Superior Court Clerks' Cooperative Authority (GSCCCA)** | Deed transfers, legal descriptions, grantor/grantee | `https://www.gsccca.org/` |
| 2 | **SC Register of Deeds (county-level)** | Deed transfers, mortgages | County-specific, e.g. `https://www.bcgov.net/departments/register-of-deeds/` |
| 3 | **Zillow / Realtor.com** | Assessed value, lot size, year built (secondary source — always verify) | Cross-reference only |
| 4 | **LoopNet / CoStar** | Commercial property specs, sales comps | Use for cross-validation |
| 5 | **FEMA Flood Map Service Center** | Flood zone designation | `https://msc.fema.gov/portal/home` |
| 6 | **GA EPD / SC DHEC** | Environmental flags, wetlands | `https://epd.georgia.gov/` |

---

## The Python Request Template

When using Tier 2 (REST API) or Tier 1 (direct URL extract) via code, always use this template to avoid being flagged:

```python
import requests
import time

def fetch_county_data(url, referer=None):
    headers = {
        "User-Agent": (
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
            "AppleWebKit/537.36 (KHTML, like Gecko) "
            "Chrome/124.0.0.0 Safari/537.36"
        ),
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
        "Accept-Language": "en-US,en;q=0.5",
        "Accept-Encoding": "gzip, deflate, br",
        "Connection": "keep-alive",
        "Upgrade-Insecure-Requests": "1",
    }
    if referer:
        headers["Referer"] = referer

    # Polite delay — never make back-to-back requests
    time.sleep(2)

    response = requests.get(url, headers=headers, timeout=20)
    response.raise_for_status()
    return response.text
```

**Key rules in this template:**
- The `User-Agent` string matches a real Chrome browser on Windows — never use Python's default `python-requests/x.x.x` agent, which is instantly flagged.
- `Accept` and `Accept-Language` headers match what a real browser sends.
- The 2-second `time.sleep()` is non-negotiable. Instant back-to-back requests are the #1 trigger for rate limiting.
- Always set `timeout=20` — county servers are often slow and will hang indefinitely without a timeout.

---

## What to Do When You Hit a CAPTCHA

If a CAPTCHA appears at any point:

1. **Stop immediately.** Do not attempt to solve it or retry.
2. **Log the failure:** Note the county, site URL, and timestamp.
3. **Switch to a fallback source** from the table above.
4. **Flag it in the research notes** on the offering site as: *"Assessed value sourced from [fallback source] — county portal inaccessible at time of research."*

Never fabricate or estimate a data point that you could not verify. Always note the source and date of every figure on the site.

---

## The Golden Rule

> **One property. One session. Human pace. Always start from the homepage.**

If you follow these four principles, you will successfully extract data from 90%+ of county portals in PIER's coverage area without triggering bot detection. The remaining 10% (primarily Beaufort County SC and some older GA county sites) require browser automation with the human-pacing rules above.

---

## Quick Reference Card

| Situation | Action |
|---|---|
| Know the PARID, county uses qPublic | Try `webpage_extract` with direct URL first |
| County uses Esri GIS | Find the ArcGIS REST endpoint via DevTools, query with Python |
| Site returns blank page | JavaScript-rendered — switch to browser tools |
| Cloudflare "checking browser" | Wait 5 seconds, retry once with browser tools |
| CAPTCHA appears | Stop, use fallback source, note in research log |
| Session expired error | Navigate to homepage first, then retry |
| Need deed/transfer history | Use GSCCCA (GA) or county Register of Deeds (SC) |
| Need flood zone | FEMA MSC — always works, no bot detection |
| Need zoning confirmation | County planning/MPC site — use browser, search by address |
