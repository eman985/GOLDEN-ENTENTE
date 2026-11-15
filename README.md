# GOLDEN-ENTENTE
ARES ATLES website: setup
Folder layout on your host (aresatles.earth):
```
index.html                 home page (choose Base or Visor)
assets/                    logo
visor/index.html           surface visor
visor/terrain/             HiRISE imagery + elevation (YOU generate this, step 1)
base/index.html            base / tactical map page (reads ../visor/terrain/)
golden-ares/               assistant (golden-ares.js, config.js)
shared/mission-waypoints.js  the 4 waypoints used by BOTH pages (edit once)
golden-ares-proxy/         tiny server that holds your Gemini key (step 3, deployed to Google Cloud, not your website host)
prepare_terrain.py         makes the terrain/ folder
```
1. Why the HiRISE imagery is not showing, and the fix
The visor loads `visor/terrain/` (meta.json, height.u16, lo/ and hi/ tiles). That folder does not exist until you make it,
which is what the red bar "COULD NOT LOAD THE terrain/ FOLDER" means. It is not in the zip because it is hundreds of MB.
Install GDAL (OSGeo4W Shell on Windows, or QGIS) and Python packages: `pip install pillow numpy`
Cut the sector out of the USGS Mars 2020 mosaics (same box for both):
```
   gdal_translate -projwin 4589047.08 1094803.66 4592603.56 1091839.93 /vsicurl/https://planetarymaps.usgs.gov/mosaic/mars2020_trn/HiRISE/JEZ_hirise_soc_006_orthoMosaic_25cm_Eqc_latTs0_lon0_first.tif sector_ortho.tif
   gdal_translate -projwin 4589047.08 1094803.66 4592603.56 1091839.93 /vsicurl/https://planetarymaps.usgs.gov/mosaic/mars2020_trn/HiRISE/JEZ_hirise_soc_006_DTM_MOLAtopography_DeltaGeoid_1m_Eqc_latTs0_lon0_blend40.tif sector_dtm.tif
   ```
(If the download is slow, download each .tif from the USGS Astrogeology page and use the local file name instead of the /vsicurl/ URL.)
Build the web files: `python prepare_terrain.py sector_ortho.tif sector_dtm.tif`  -> creates a `terrain` folder
Upload that folder so that `https://aresatles.earth/visor/terrain/meta.json` opens in your browser and shows text.
If that address gives a 404, the visor will still show the error bar.
2. Check the pages
https://aresatles.earth/  -> home page
Click Visor: the red bar should be gone and the ground should show HiRISE imagery.
Base: click the EVA-01 marker on the map, then "ENTER VISOR VIEW" (it carries only the location).
3. Turn on Golden Ares with Gemini (the key never goes in the website)
Anything in a web page can be read by anyone, so the key lives on a small proxy that you deploy on Google Cloud
(it bills to your Cloud credits). Without it, Golden Ares still works with its on-board engine for the core questions.
In Google Cloud console, pick the project that has your credits and make sure billing is linked.
Create a Gemini API key for that project (Google AI Studio -> Get API key -> choose this Cloud project), and restrict it to the Generative Language API.
Install the gcloud CLI, then in a terminal:
```
   cd golden-ares-proxy
   gcloud auth login
   gcloud config set project YOUR_PROJECT_ID
   gcloud services enable run.googleapis.com cloudbuild.googleapis.com
   gcloud run deploy golden-ares --source . --region europe-west2 --allow-unauthenticated \
     --set-env-vars "GEMINI_API_KEY=PASTE_KEY_HERE,ALLOWED_ORIGINS=https://aresatles.earth,https://www.aresatles.earth"
   ```
It prints a Service URL such as `https://golden-ares-xxxx.a.run.app`.
Check `https://golden-ares-xxxx.a.run.app/health` shows `"keyLoaded":true`.
Edit `golden-ares/config.js`: `window.GOLDEN_ARES_ENDPOINT = 'https://golden-ares-xxxx.a.run.app/chat';` and re-upload it.
Open the visor or base, click the gold eagle button, ask "Give me a summary of my current health readings".
The small grey line under each answer says "Gemini" or "on-board engine".
If the model name is rejected, set env var GEMINI_MODEL to a current Gemini Flash model (default `gemini-2.5-flash`).
4. How the assistant is attached to each page (what was added)
Both pages got the same three things at the bottom of the file:
`<script src="../golden-ares/config.js">` and `golden-ares.js`
`GoldenAres.init({ page, endpoint, getContext })`
`getContext()`: a read-only function that returns the page's live data (position, habitat distance, vitals, suit, environment, alerts,
observations). Anything a page does not simulate is `null`, and the assistant is told to say "unavailable" rather than guess.
base: the function sits in the same block (reads STATE, PHYSIO_ENGINE, ASTAR_PLANNER).
visor: the visor code is wrapped in a function, so a 12-line read-only hook `window.__aresVisorContext` sits at the end of its script.
To add a new data source later (e.g. real CO2 or science notes), add a field to `getContext()` and nothing else changes.
5. Put the whole site on Google Cloud under aresatles.earth (Firebase Hosting)
Firebase Hosting is part of Google Cloud: it serves the pages and the terrain folder, gives you https, and connects your domain.
The assistant then lives at the same address (`/api/chat`), so no extra settings are needed.
Do step 3 first (deploy the proxy to Cloud Run, service name `golden-ares`, region `europe-west2`).
Put your generated `visor/terrain/` folder inside this folder, next to `index.html`.
In `golden-ares/config.js` set `window.GOLDEN_ARES_ENDPOINT = '/api/chat';`
In a terminal: `npm install -g firebase-tools`, then `firebase login`, then in this folder `firebase init hosting`
(choose your Google Cloud project, say NO to overwriting firebase.json and index.html), then `firebase deploy`.
Firebase needs the pay-as-you-go (Blaze) plan to forward /api to Cloud Run; your Cloud credits cover it.
Firebase console -> Hosting -> Add custom domain -> `aresatles.earth`. It shows DNS records (A and TXT). Add them at the company
where you bought the domain, then wait until it says Connected (minutes to a few hours). https is automatic.
Done: aresatles.earth is the home page, /base/ and /visor/ the two views.
6. Why the base map goes fuzzy, and what changed
The map shows sharp imagery only from your own `visor/terrain/` tiles. Without that folder it falls back to NASA's live HiRISE
tile server and to a very coarse global Mars map (about 230 m per pixel). If the live server is slow, blocked or changes address,
you only see the coarse map stretched up, which is the fuzzy look. Changes in this version:
With `terrain/` present the map uses only the local HiRISE tiles and is locked to the 3 km sector, so no coarse tiles are ever shown.
Without `terrain/` an amber notice appears on the map saying so, instead of silently looking blurry.
A browser style (`will-change`) that could blur tiles after zooming was removed.
The terrain files are now loaded once, not twice.
So the permanent fix is step 1: generate the terrain folder and put it at `visor/terrain/`.
7. Waypoints
`shared/mission-waypoints.js` holds the four waypoints (Three Forks Delta Scarp, Seitah Fractured Bedrock, Kodiak Remnant Mesa Butte, Belva Impact Crater Rim). The base and the visor both read it.
In the visor they are science targets (WP-1 to WP-4) on the compass, radar, maps and AR beacons.
Pick one as the destination and the visor plans a safe route on the real elevation data: green dots on the ground, a dotted line on the radar and maps, and the DISTANCE / ETA readout follows that route. It replans if you leave the path.
Clicking the EVA-01 marker on the base map passes the location, the current waypoint target, and which waypoints are already sampled into the visor.
