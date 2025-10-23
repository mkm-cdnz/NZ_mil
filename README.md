# NZ Defence‑Industry Map (Sheets‑backed)

> Purpose: Help **Future Me** remember what this is, how it works, and how to update it.

**Last updated:** 2025-10-22

## NZ_mil (proof of concept)
Explorations &amp; visualisations of materiel & dual-use products produced in Aotearoa. 
- Uses Google Sheets as a headless CMS

 # **Try it out [here!](https://storage.googleapis.com/web-visualisations/NZ_mil/nz_defence_industry_map_v4.html)**

---

## What this is

A single‑file web app that maps **New Zealand defence‑industry** companies and capabilities.  
Data is curated in **Google Sheets** (SSOT), geocoded with **Google Apps Script**, and rendered with **Leaflet** + **PapaParse**.  
Static hosting is on **Google Cloud Storage (GCS)** under the existing *web‑visualisations* project.

### Why
- Quickly see geographic spread and **who does what** (naval sustainment, radios, munitions, etc.).
- Click through to **official company sites** from popups.
- **Low‑friction updates**: add a row in Sheets → coordinates are auto‑filled → page reflects new data.
- I just wanted to experiment with using **Google Sheets as a headless CMS**, okay!?

---

## Live URLs

- **Public page v1:** [https://storage.googleapis.com/web-visualisations/NZ_mil/nz_defence_industry_sheets_map.html](https://storage.googleapis.com/web-visualisations/NZ_mil/nz_defence_industry_sheets_map.html)
- **Public page v3:** [https://storage.googleapis.com/web-visualisations/NZ_mil/nz_defence_industry_sheets_map.html](https://storage.googleapis.com/web-visualisations/NZ_mil/nz_defence_industry_map_v3.html)  
- **Google Sheet (view‑only):** [https://docs.google.com/spreadsheets/d/1XI7fWf5Xc9EshogItq-P6XREqbZr2EGM0W8PYppG5DQ/edit?usp=sharing](https://docs.google.com/spreadsheets/d/1XI7fWf5Xc9EshogItq-P6XREqbZr2EGM0W8PYppG5DQ/edit?usp=sharing)  
- **CSV export URL used by the page:** [https://docs.google.com/spreadsheets/d/e/2PACX-1vRInRJ-k6PPnkTvP0dwO_Rh0WH5CqIgfSYF1zT5KBXY2Scd0ARkHMusJ_LvhxixaRWhqpvS5pFQqBTv/pub?gid=0&single=true&output=csv](https://docs.google.com/spreadsheets/d/e/2PACX-1vRInRJ-k6PPnkTvP0dwO_Rh0WH5CqIgfSYF1zT5KBXY2Scd0ARkHMusJ_LvhxixaRWhqpvS5pFQqBTv/pub?gid=0&single=true&output=csv)  
 
---

## Data model (Sheet tab: `companies`)

Columns (left → right):
```
id,city,address,company,capability,website,category,lat,lng,
source,verified_on,geocode_status,geocode_source,notes
```
- **Manually enter:** `city,address,company,capability,website,category,source,notes`
- **Script fills:** `lat,lng,geocode_status,geocode_source,verified_on`
- **Categories (drives marker colors):** Naval sustainment · Marine propulsion · Boatbuilding · Comms & radios · Aerospace/space · Munitions · Simulation/training · Other

> Keep addresses precise when possible (street number + suburb + city + “New Zealand”).

---

## Geocoding (Apps Script)

Apps Script (Extensions → Apps Script) adds a custom menu **Data → Geocode new rows**. It uses the Maps Service **Geocoder** with `setRegion('nz')` to bias to NZ.

```gs
/**
 * NZ Defence Map — Geocoder
 * Populates lat/lng + status fields for the 'companies' and 'NZDF' tabs.
 * Uses Apps Script Maps Service Geocoder with NZ region bias.
 *
 * Docs:
 *  - Menus & onOpen(): https://developers.google.com/apps-script/guides/menus
 *  - Spreadsheet UI Menu class: https://developers.google.com/apps-script/reference/base/menu
 *  - Maps Service (Geocoder): https://developers.google.com/apps-script/reference/maps/geocoder
 *  - Maps Service index: https://developers.google.com/apps-script/reference/maps/
 *  - Quotas (Geocode calls/day): https://developers.google.com/apps-script/guides/services/quotas
 */

/** Menu: Data → Geocode (Companies / NZDF / Both) */
function onOpen() {
  const ui = SpreadsheetApp.getUi();
  ui.createMenu('Data')
    .addItem('Geocode: companies', 'geocodeCompanies')
    .addItem('Geocode: NZDF', 'geocodeNZDF')
    .addItem('Geocode: both tabs', 'geocodeBothTabs')
    .addSeparator()
    .addItem('Re-geocode selected rows (active sheet)', 'geocodeSelectedRows')
    .addToUi(); // :contentReference[oaicite:1]{index=1}
}

/** Convenience wrappers */
function geocodeCompanies()  { geocodeSheet_('companies'); }
function geocodeNZDF()       { geocodeSheet_('NZDF'); }
function geocodeBothTabs()   { geocodeSheet_('companies'); geocodeSheet_('NZDF'); }

/**
 * Re-geocode only the currently selected rows in the active sheet.
 * Useful if you fixed an address and want to refresh coords/status immediately.
 */
function geocodeSelectedRows() {
  const sh = SpreadsheetApp.getActiveSheet();
  const range = sh.getActiveRange();
  if (!range) return;
  const header = sh.getRange(1,1,1,sh.getLastColumn()).getValues()[0].map(String);
  const col = (name) => header.indexOf(name);

  const iAddress  = col('address'),   iCity = col('city');
  const iLat      = col('lat'),       iLng  = col('lng');
  const iStatus   = col('geocode_status'),  iSrc  = col('geocode_source');
  const iVerified = col('verified_on');

  if ([iAddress,iCity,iLat,iLng,iStatus,iSrc,iVerified].some(idx => idx < 0)) {
    SpreadsheetApp.getActive().toast('Missing one or more required columns (address, city, lat, lng, geocode_status, geocode_source, verified_on).', 'Geocode', 8);
    return;
  }

  const nzBias = Maps.newGeocoder().setRegion('nz'); // bias to New Zealand :contentReference[oaicite:2]{index=2}
  const values = sh.getRange(1,1,sh.getLastRow(),sh.getLastColumn()).getValues();

  // Update only the selected rows’ indexes
  const [r0, , rCount] = [range.getRow(), range.getColumn(), range.getNumRows()];
  let updated = 0;

  for (let r = r0; r < r0 + rCount; r++) {
    if (r === 1) continue; // skip header if selected
    const row = values[r-1];
    const q = buildQuery_(row[iAddress], row[iCity]);
    const {lat, lng, status, src} = geocodeOne_(nzBias, q);
    row[iLat]      = lat || '';
    row[iLng]      = lng || '';
    row[iStatus]   = status;
    row[iSrc]      = src || '';
    row[iVerified] = new Date();
    sh.getRange(r, 1, 1, row.length).setValues([row]);
    updated++;
  }
  SpreadsheetApp.getActive().toast(`Re-geocoded ${updated} row(s) in "${sh.getName()}".`, 'Geocode', 6);
}

/**
 * Core worker: geocode any sheet by name.
 * Requirements: columns named exactly:
 *   'address','city','lat','lng','geocode_status','geocode_source','verified_on'
 */
function geocodeSheet_(sheetName) {
  const ss = SpreadsheetApp.getActive();
  const sh = ss.getSheetByName(sheetName);
  if (!sh) {
    SpreadsheetApp.getActive().toast(`Sheet "${sheetName}" not found.`, 'Geocode', 8);
    return;
  }

  const values = sh.getDataRange().getValues();
  if (values.length < 2) {
    SpreadsheetApp.getActive().toast(`Sheet "${sheetName}" has no data rows.`, 'Geocode', 5);
    return;
  }

  const header = values[0].map(String);
  const col = (name) => header.indexOf(name);

  const iAddress  = col('address'),   iCity = col('city');
  const iLat      = col('lat'),       iLng  = col('lng');
  const iStatus   = col('geocode_status'),  iSrc  = col('geocode_source');
  const iVerified = col('verified_on');

  if ([iAddress,iCity,iLat,iLng,iStatus,iSrc,iVerified].some(idx => idx < 0)) {
    SpreadsheetApp.getActive().toast(
      `Sheet "${sheetName}" is missing one or more required columns (address, city, lat, lng, geocode_status, geocode_source, verified_on).`,
      'Geocode',
      10
    );
    return;
  }

  const nzBias = Maps.newGeocoder().setRegion('nz'); // NZ bias for disambiguation :contentReference[oaicite:3]{index=3}

  const out = [];
  let updated = 0;
  for (let r = 1; r < values.length; r++) {
    const row = values[r];
    const latPresent = !!row[iLat];
    const lngPresent = !!row[iLng];
    const addressPresent = !!row[iAddress];
    if ((!latPresent || !lngPresent) && addressPresent) {
      const q = buildQuery_(row[iAddress], row[iCity]);
      const {lat, lng, status, src} = geocodeOne_(nzBias, q);

      row[iLat]      = lat || '';
      row[iLng]      = lng || '';
      row[iStatus]   = status;
      row[iSrc]      = src || '';
      row[iVerified] = new Date();

      out.push({rowIndex: r+1, rowValues: row});
      updated++;
    }
  }

  // Write back in batches (one row at a time for clarity; still fine at our scale)
  out.forEach(({rowIndex, rowValues}) => {
    sh.getRange(rowIndex, 1, 1, rowValues.length).setValues([rowValues]);
  });

  SpreadsheetApp.getActive().toast(
    `Geocoded ${updated} new row(s) in "${sheetName}".`,
    'Geocode',
    6
  );
}

/** Build a human-readable query string for the geocoder. */
function buildQuery_(address, city) {
  return [address, city, 'New Zealand'].filter(Boolean).join(', ');
}

/** One-shot geocode with basic error handling. */
function geocodeOne_(geocoder, query) {
  let lat = null, lng = null, status = 'FAILED', src = '';
  try {
    const res = geocoder.geocode(query); // Apps Script Maps Service geocoder :contentReference[oaicite:4]{index=4}
    if (res && res.results && res.results.length) {
      lat = res.results[0].geometry.location.lat;
      lng = res.results[0].geometry.location.lng;
      status = 'OK';
      src = 'google_maps_geocoder';
    } else {
      status = 'ZERO_RESULTS';
    }
  } catch (e) {
    status = 'ERROR';
  }
  return {lat, lng, status, src};
}

```

> If ever switching to **Nominatim** (OSM), throttle ~**1 req/sec** and include a descriptive User‑Agent per their usage policy.

---

## Front‑end (nz_defence_industry_sheets_map.html)

- **Leaflet** for mapping and popups.  
- **PapaParse** to fetch/parse the Sheet CSV in the browser.  
- Search box filters by **city / company / category / capability**.  
- Legend explains approximate locations during verification.

**Important:** inside `index.html`, set:
```js
const CSV_URL = "https://docs.google.com/spreadsheets/d/<FILE_ID>/export?format=csv&gid=<TAB_GID>";
```
The app appends a cache‑buster (`?_=${{Date.now()}}`) so edits in Sheets show up on refresh.

---

## Running locally

Do **not** open via `file://` (browsers block cross‑origin CSV). Serve over HTTP:
```bash
python -m http.server 5500
# http://localhost:5500/
```
Or use VS Code “Live Server”.

---

## Deploying to Google Cloud Storage (static)

1) Upload `nz_defence_industry_sheets_map.html` (and assets) to the bucket (e.g., `web-visualisations/nz-defence-industry/index.html`).  
2) Make objects public (grant **Storage Object Viewer** to **allUsers**).  
3) (Nice) Set website config so folder paths load `index.html`:
```bash
gcloud storage buckets update gs://<YOUR_BUCKET>   --web-main-page-suffix=index.html --web-error-page=404.html
```
4) (Optional) Put a HTTPS load balancer + Cloud CDN in front for custom domain + caching.

**Caching:** For the HTML object, consider `Cache-Control: public, max-age=60`. CSV lives on docs.google.com.

---

## How to add/update data

1. Add a row in `companies` with human‑readable `address` and `capability`.  
2. Run **Data → Geocode new rows** (or set a time‑based trigger).  
3. Confirm `geocode_status = OK` and `lat/lng` present.  
4. Refresh the page; new marker should appear.

---

## Troubleshooting & Caveats

- Initial CSVs were generated via GPT-5 web search
  - LLM was prompted with a goal to search the web
  - LLM was prompted to output results as a CSV
  - The current state of LLMs are _not_ ready to perform accurate, trustworthy research 

---

## Roadmap / Nice‑to‑haves

- Marker clustering plugin when points grow.  
- Category toggles (toolbar).  
- City‑aggregate popups (one marker per city listing companies).  
- Switchable basemap / vector tiles (MapLibre GL).  
- Simple admin page to validate and flag “site‑verified” vs “centroid”.
- Streamlined data ingestion

---

## Credits / Licenses

- **Leaflet** (JS map lib).  
- **PapaParse** (CSV loader).  
- **OpenStreetMap** tiles (if using OSM). Respect tile provider T&Cs.  
- **Apps Script Maps Service** for geocoding.  
- Code: MIT. Data: respect original source licenses.

---

## Notes for Future Me

- Keep capability blurbs **concise and factual**; add a `source` for each row.  
- Many companies may be **dual‑use** (civil + military).  
- If you change column names, update both the **Apps Script** and the **index.html** mapping.
- Consider a small CI to validate CSV schema (headers present, lat/lng numeric).
