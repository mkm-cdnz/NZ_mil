# NZ Defence‑Industry Map (Sheets‑backed)

> Purpose: Help **Future Me** remember what this is, how it works, and how to update it.

**Last updated:** 2025-10-22

## NZ_mil (proof of concept)
Explorations &amp; visualisations of materiel & dual-use products produced in Aotearoa. Try it out [here!](https://storage.googleapis.com/web-visualisations/NZ_mil/nz_defence_industry_sheets_map.html)
- Uses Google Sheets as a headless CMS

---

## What this is

A single‑file web app that maps **New Zealand defence‑industry** companies and capabilities.  
Data is curated in **Google Sheets** (SSOT), geocoded with **Google Apps Script**, and rendered with **Leaflet** + **PapaParse**.  
Static hosting is on **Google Cloud Storage (GCS)** under the existing *web‑visualisations* project.

### Why
- Quickly see geographic spread and **who does what** (naval sustainment, radios, munitions, etc.).
- Click through to **official company sites** from popups.
- **Low‑friction updates**: add a row in Sheets → coordinates are auto‑filled → page reflects new data.

---

## Live URLs (fill these in)

- **Public page:** [https://storage.googleapis.com/web-visualisations/NZ_mil/nz_defence_industry_sheets_map.html](https://storage.googleapis.com/web-visualisations/NZ_mil/nz_defence_industry_sheets_map.html)  
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
function onOpen() {
  SpreadsheetApp.getUi().createMenu('Data')
    .addItem('Geocode new rows', 'geocodeEmptyLatLng')
    .addToUi();
}

function geocodeEmptyLatLng() {
  const sh = SpreadsheetApp.getActive().getSheetByName('companies');
  const values = sh.getDataRange().getValues();
  const header = values[0].map(String);
  const col = n => header.indexOf(n);

  const iAddress = col('address'), iCity = col('city');
  const iLat = col('lat'), iLng = col('lng');
  const iStatus = col('geocode_status'), iSrc = col('geocode_source'), iVerified = col('verified_on');

  const g = Maps.newGeocoder().setRegion('nz');

  for (let r = 1; r < values.length; r++) {
    const row = values[r];
    if (!row[iLat] && !row[iLng]) {
      const q = [row[iAddress], row[iCity], 'New Zealand'].filter(Boolean).join(', ');
      let lat = '', lng = '', status = 'FAILED', src = '';
      try {
        const res = g.geocode(q);
        if (res && res.results && res.results.length) {
          lat = res.results[0].geometry.location.lat;
          lng = res.results[0].geometry.location.lng;
          status = 'OK'; src = 'google_maps_geocoder';
        }
      } catch (e) { status = 'ERROR'; }
      row[iLat] = lat; row[iLng] = lng;
      row[iStatus] = status; row[iSrc] = src; row[iVerified] = new Date();
      sh.getRange(r + 1, 1, 1, row.length).setValues([row]);
    }
  }
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

## Troubleshooting

- **Map shows but no markers** → CSV didn’t load (link not public, or opened via `file://`).  
- **Rows > 0 but Markers = 0** → `lat`/`lng` missing or non‑numeric; re‑run geocoder; check `geocode_status`.  
- **CSV intermittently unavailable** → Use the `export?format=csv&gid=...` URL or republish the “Publish to web” link.

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
