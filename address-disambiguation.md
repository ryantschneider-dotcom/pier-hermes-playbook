# Hermes Agent Guide: When the Address Does Not Match — Parcel Resolution Workflow

**For:** Hermes (PIER Commercial AI Agent)
**Author:** Manus (PIER Research Agent)
**Updated:** June 2026
**Purpose:** Explains the full step-by-step process for finding the correct parcel record when a county portal returns "no property found" or the address does not match.

---

## Why This Happens

Commercial properties fail address lookups far more often than residential properties. The reasons are:

**Address format mismatches.** The county tax assessor may store the address as `6301 ABERCORN ST` while Buildout lists it as `6301 Abercorn Street`. The portal search is often exact-match or prefix-match only — a single period, abbreviation difference, or extra space causes a "not found" result.

**Suite and unit numbers.** A building with multiple suites (e.g., `6301 Abercorn St Suite 100`) may be stored at the parcel level without any suite number. Searching with the suite number returns nothing; searching without it finds the parent parcel.

**Highway and route designations.** A property on `US Highway 17` may be stored as `US HWY 17`, `HWY 17`, `US-17`, `STATE ROUTE 17`, or just `17 HWY` depending on the county. There is no standard.

**Rural route and lot numbers.** Properties outside city limits often have no street address at all in the assessor database. They are stored only by parcel ID, subdivision name, or legal description (e.g., `LOT 4 PINE RIDGE SUBDIVISION`).

**Mailing address vs. situs address.** The county stores two addresses: the **situs address** (where the property physically is) and the **mailing address** (where the tax bill goes). Some portals default to searching the mailing address, which may be a PO Box or the owner's home address in another state.

**New construction / recently platted parcels.** A parcel that was recently subdivided or platted may not yet have a street address assigned. It exists in the system only by PARID or legal description.

**Annexed or re-addressed parcels.** Properties that were annexed into a city, or that went through a county-wide address standardization project, may have a new address in the assessor system that does not match older records.

---

## The Resolution Workflow — 8 Steps in Order

Work through these steps in sequence. Stop as soon as you find the parcel.

---

### Step 1: Try Address Variations First

Before doing anything else, try these address format variations in the county portal search:

| What to try | Example |
|---|---|
| Abbreviate the street type | `Abercorn St` instead of `Abercorn Street` |
| Remove the street type entirely | Search just `6301 Abercorn` |
| Remove suite / unit number | `6301 Abercorn St` not `6301 Abercorn St Ste 100` |
| All caps | `6301 ABERCORN ST` |
| Try just the street number | Enter `6301` in the street number field and `Abercorn` in the street name field separately if the form has split fields |
| Try without directional prefix | `Main St` instead of `N Main St` or `North Main St` |
| Try with directional prefix spelled out | `North Main Street` instead of `N Main St` |
| Try highway variants | `US 17`, `HWY 17`, `US HWY 17`, `STATE RD 17` |

**Most county portals have a wildcard or partial-match option.** Look for a "begins with" or "contains" toggle near the search field. If available, use "contains" and search just the distinctive part of the street name.

---

### Step 2: Search by Owner Name Instead of Address

If address search fails, switch to **owner name search**. Every county portal has this option.

**How to find the owner name:**
- Check the Buildout listing — it sometimes lists the seller/owner name.
- Check the listing agent's name and the company name on the listing.
- Search Google for `"[address]" "owner" OR "deed" site:[county].gov` or similar.
- Check LoopNet or CoStar — they often show the ownership entity name.

**What to search:**
- Try the full legal entity name: `SAVANNAH CARDIOLOGY LLC`
- Try just the key word: `SAVANNAH CARDIOLOGY`
- Try the individual's last name if it is a personal ownership: `SCHNEIDER`

**Important:** Owner names in county records are almost always the **legal entity name exactly as it appears on the deed** — not the DBA or trade name. A property owned by `COASTAL GEORGIA PROPERTIES LLC` will not be found by searching `Coastal Georgia Real Estate`.

---

### Step 3: Use Google Maps to Get the Precise Coordinates

This is the most reliable method when address search completely fails. Google Maps almost always has the correct pin location for a commercial property, even when the county portal does not recognize the address.

**Process:**
1. Search the property address in Google Maps.
2. Confirm the pin drops on the correct building or lot.
3. Right-click the pin → select "What's here?" — Google shows the precise latitude/longitude coordinates.
4. Copy the coordinates (e.g., `31.9876, -81.1234`).

Now use those coordinates to find the parcel in the county GIS system (see Step 4).

---

### Step 4: Use the County GIS Map Viewer to Click the Parcel

With the coordinates from Step 3, go to the county's GIS map viewer and navigate to that exact location.

**Process:**
1. Open the county GIS viewer in the browser.
2. Use the search bar in the GIS viewer — most support coordinate search. Enter the lat/long from Google Maps.
3. The map will center on that location.
4. Click directly on the parcel polygon.
5. A popup or sidebar will appear showing the **PARID** (Parcel ID), owner name, and acreage.
6. Copy the PARID — this is your key to unlock all other data.

**If the GIS viewer does not support coordinate search:**
- Use the zoom controls to navigate to the general area.
- Use Google Maps satellite view side-by-side to orient yourself.
- Click on the correct parcel polygon.

**County GIS viewer URLs for PIER's coverage area:**

| County | GIS Viewer URL |
|---|---|
| Chatham County, GA | `https://www.chathamcounty.org/Government/Departments/InformationTechnology/GIS` |
| Glynn County, GA | `https://gis.glynncounty.org/` |
| Bryan County, GA | `https://www.bryancountyga.org/government/departments/gis` |
| Effingham County, GA | `https://www.effinghamcounty.org/` → GIS section |
| Camden County, GA | `https://www.camdencountyga.gov/` → GIS |
| Beaufort County, SC | `https://www.bcgov.net/departments/gis/` |
| Jasper County, SC | SC Revenue and Fiscal Affairs GIS: `https://gis.sc.gov/` |
| Bulloch County, GA (Statesboro) | `https://www.bullochweb.com/` → Assessor |
| Liberty County, GA (Hinesville) | `https://www.libertycountyga.com/` → Tax Assessor |

---

### Step 5: Use the Esri ArcGIS REST API with Coordinates

Once you have coordinates from Google Maps, you can query the county's ArcGIS parcel layer directly using a spatial query — no address needed at all.

```python
import requests

# Coordinates from Google Maps (longitude first in Esri queries)
lat = 31.9876
lon = -81.1234

# Esri spatial query — find parcel containing this point
url = "https://maps.chathamcounty.org/arcgis/rest/services/Parcels/FeatureServer/0/query"
params = {
    "geometry": f"{lon},{lat}",
    "geometryType": "esriGeometryPoint",
    "spatialRel": "esriSpatialRelIntersects",
    "outFields": "*",
    "f": "json",
    "inSR": "4326"  # WGS84 coordinate system
}

response = requests.get(url, params=params, timeout=20)
data = response.json()

if data.get("features"):
    parcel = data["features"][0]["attributes"]
    print(f"PARID: {parcel.get('PARID')}")
    print(f"Owner: {parcel.get('OWNER')}")
    print(f"Address: {parcel.get('SITEADDR')}")
    print(f"Acreage: {parcel.get('ACREAGE')}")
```

This bypasses the address problem entirely. You are asking "what parcel is at this GPS coordinate?" instead of "what parcel has this address?"

---

### Step 6: Search GSCCCA (Georgia) or County Register of Deeds (South Carolina) by Address

The deed recording system is maintained separately from the tax assessor and often has better address matching.

**Georgia — GSCCCA:**
1. Go to `https://www.gsccca.org/`
2. Click "Search" → "PT-61 Real Estate Transfer Tax"
3. Search by address or grantee name
4. The deed record will show the legal description and PARID

**South Carolina — County Register of Deeds:**
- Beaufort: `https://www.bcgov.net/departments/register-of-deeds/`
- Jasper: `https://www.jaspercountysc.gov/`

Search by grantor/grantee name or street address. The deed will contain the full legal description and tax map number (which is the SC equivalent of PARID).

---

### Step 7: Try Zillow or Realtor.com as a PARID Lookup Tool

This sounds counterintuitive, but Zillow and Realtor.com often display the county Assessor Parcel Number (APN) on their property detail pages — even for commercial properties that are not actively listed.

**Process:**
1. Search the address on Zillow.
2. Scroll to the "Home Details" or "Public Records" section.
3. Look for "APN," "Parcel Number," or "Tax ID."
4. Copy that number — it is the PARID.
5. Go back to the county assessor portal and search by parcel number directly.

Parcel number search almost never fails because it is an exact-match lookup on the primary key of the database.

---

### Step 8: Call or Email the County Assessor's Office

This is the last resort and should only be used when all digital methods have failed. It is also the most reliable — a human assessor staff member can find any parcel in seconds.

**What to say:**
> "Hi, I'm a commercial real estate broker researching a property at [address]. I'm having trouble finding it in your online portal. Could you help me identify the parcel ID for this property?"

County assessor offices are public agencies and are required to provide this information. They are generally helpful and the call takes under 5 minutes.

**Contact information for key counties:**

| County | Phone | Website |
|---|---|---|
| Chatham County Tax Assessor | (912) 652-7271 | `https://www.chathamcounty.org/` |
| Glynn County Tax Assessor | (912) 554-7093 | `https://www.glynncounty.org/` |
| Bryan County Tax Assessor | (912) 653-3880 | `https://www.bryancountyga.org/` |
| Beaufort County Assessor | (843) 255-2400 | `https://www.bcgov.net/` |
| Jasper County Assessor | (843) 717-3620 | `https://www.jaspercountysc.gov/` |

---

## Special Cases

### Multi-Parcel Properties

Large commercial properties — shopping centers, industrial parks, campuses — are often split across multiple parcels. A search for the street address may return only one of several parcels.

**How to identify all parcels:**
1. Find the first parcel using the workflow above.
2. In the GIS viewer, look at the surrounding parcels — do they share the same owner name?
3. Run an owner name search in the assessor portal to find all parcels owned by the same entity.
4. Cross-reference the total acreage across all parcels against the listing's stated acreage.

### Outparcels and Pad Sites

A property described as "outparcel at [shopping center]" will almost never be found by the shopping center's address. It will have its own separate PARID and may have a different street address or no address at all. Use the GIS coordinate method (Steps 3–5) to locate it visually.

### Properties on State Routes / US Highways

For properties whose address is a highway number (e.g., `1234 US Highway 17`), try all of these in the county portal:

- `1234 US HWY 17`
- `1234 HWY 17`
- `1234 US 17`
- `1234 US-17`
- `1234 STATE ROUTE 17`
- `1234 SR 17`

If none work, go straight to the GIS coordinate method.

### Recently Subdivided or Platted Land

If a parcel was recently created through a subdivision plat, it may exist in the GIS system but not yet in the assessor's address database. The GIS viewer will show it as a polygon with a PARID but no situs address. Use the PARID to pull whatever data exists, and note in the research log that the address is not yet assigned in the assessor system.

---

## The Decision Tree (Quick Reference)

```
Address search returns "not found"
        │
        ▼
Try address format variations (abbreviations, no suite, no directional)
        │
        ├── Found? → Pull data, done.
        │
        ▼
Try owner name search
        │
        ├── Found? → Pull data, done.
        │
        ▼
Get GPS coordinates from Google Maps
        │
        ▼
Click parcel in county GIS viewer → get PARID
        │
        ├── Found? → Search by PARID in assessor portal, done.
        │
        ▼
Query ArcGIS REST API with coordinates (spatial intersect query)
        │
        ├── Found? → Extract JSON data, done.
        │
        ▼
Search GSCCCA (GA) or Register of Deeds (SC) by address or owner
        │
        ├── Found? → Get PARID from deed, search assessor, done.
        │
        ▼
Search Zillow / Realtor.com for APN/Parcel Number
        │
        ├── Found? → Search assessor by parcel number, done.
        │
        ▼
Call county assessor office directly
        │
        └── Always works.
```

---

## What to Document When You Find It

Once you locate the correct parcel through any of these methods, record the following before moving on:

1. **The PARID** — the primary key for all future lookups on this property.
2. **The exact address as stored in the county system** — this may differ from the Buildout listing address. Note both.
3. **The method used to find it** — e.g., "Located via GIS coordinate click after address search failed; situs address stored as `6301 ABERCORN ST` not `6301 Abercorn Street`."
4. **The county portal URL** for the parcel record page — bookmark it for future reference.

This documentation protects you if the data is ever questioned and ensures the next agent working on this property can find it instantly.
