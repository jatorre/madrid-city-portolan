# Madrid Geoportal — source investigation (2026-06-05)

Investigation of **https://geoportal.madrid.es/IDEAM_WBGEOPORTAL/** (IDEAM — *Infraestructura de Datos
Espaciales del Ayuntamiento de Madrid*) to scope a Portolan catalog: what's published, how to pull it,
and how to convert it to cloud-native formats (GeoParquet / COG / PMTiles).

## 1. What the geoportal is

An **INSPIRE-compliant municipal SDI**, Esri-based. Access surfaces found:

| Surface | Endpoint | What it gives | Auth |
|---|---|---|---|
| **DCAT-AP feed** | `…/IDEAM_WBGEOPORTAL/dcat` (RDF/XML, ~1.7 MB) | Machine-readable catalog: titles, descriptions, keywords, themes, identifiers, **and the 64 datasets that have direct downloads** | none |
| **CSW catalog** | `…/IDEAM_WBGEOPORTAL/csw` (Esri Geoportal Server 2.0.2) | Full ISO 19115 metadata for all records | none (CQL not supported; use OGC filter / GetRecords) |
| **Direct file server** | `https://geoportal.madrid.es/fsdescargas/IDEAM_WBGEOPORTAL/<CAT>/<file>` | ZIP/`.z` archives of **Shapefiles**, plus KML/KMZ/CSV/PDF | none (dir listing 403, but files fetch fine) |
| **Raster GeoServer** | `https://servpub.madrid.es/georaster/<WORKSPACE>/ows` | WMS per workspace ("Raster Oficial del Ayuntamiento de Madrid"); GetMap supports `image/geotiff` output | none (workspace-scoped; global caps = "No workspace specified") |
| **Restricted GeoServer** | `https://servpub.madrid.es/geoserver/…` | — | **403** "Servicio restringido" |
| **Open-data portal** (sibling) | `https://datos.madrid.es/` | CKAN-style; some geoportal datasets link out to CSVs here | none |

**CRS:** uniformly **ETRS89 / UTM zone 30N (EPSG:25830)** — confirmed in the `.prj` of downloaded Shapefiles.

## 2. Catalog profile (the full ~817–967 records)

The DCAT header reports **967 datasets**; ~817 dataset blocks parse cleanly. The catalog is **dominated by
archival imagery metadata**, most of which is *not* directly downloadable (viewable via WMS/fototeca only):

| geoportal "categoría" | count | nature |
|---|---|---|
| vuelos (aerial flights) | 237 | historical aerial photo runs (scanned) |
| ortofoto | 209 | orthophoto mosaics (raster, WMS) |
| satelite | 65 | satellite imagery products |
| cartografia_base | 59 | base cartography |
| cartoteca | 54 | scanned historical maps |
| cambios_urbanos | 52 | urban-change raster pairs |
| mapas_vegetacion | 43 | vegetation maps |
| elevaciones | 38 | DTM / elevation |
| teledeteccion | 22 | remote sensing |
| planeamiento | 15 | urban planning (PGOUM) |
| estadistica | 12 | district/neighborhood statistics |
| callejero | 7 | street directory (vector) |

→ **753 records are metadata-only; only 64 datasets expose a direct download.** The 64 are the realistic
cloud-native conversion targets. The 753 imagery records are mostly WMS-served rasters — convertible to COG
only if/when Madrid gives us the source rasters (or we harvest via WMS/WCS, which is lossy/bounded).

## 3. The 64 directly downloadable datasets (87 distributions)

Almost all are **ESRI Shapefiles** zipped (`.zip`) or LZ-compressed (`.z`), EPSG:25830. A few KML/KMZ/CSV/PDF.

| Categoría | Fmt | Dataset | Archivo |
|---|---|---|---|
| AGENCIA_TRIBUTARIA | zip | Categorías fiscales del municipio de Madrid | `CATEGORIA_FISCAL.zip` |
| ALUMBRADO | zip | Plan de mejora de accesibilidad del alumbrado público | `PLAN_MEJORA_ACCESIBILIDAD.zip` |
| ALUMBRADO | zip | Unidades luminosas e infraestructuras (farolas) | `Iluminacion.zip` |
| BIENESTAR_SOCIAL | zip | Centros de día municipales y concertados | `A0_Plano_CentrosDeDia.zip` |
| BIENESTAR_SOCIAL | z | Red de Espacios de Igualdad | `Plano_Espacios_Igualdad.z` |
| BIENESTAR_SOCIAL | zip | Rutas WAP (Walking People) | `Rutas_WAP.zip` |
| BIENESTAR_SOCIAL | zip | Tarjetas Familia | `Tarjeta_Familia.zip` |
| CALLEJERO | zip | Callejero oficial. Geometrías de rotulación | `ROTULACION_SHP.zip` |
| CALLEJERO | zip | Callejero oficial. Subviales vigentes | `SUBVIALES_SHP.zip` |
| CALLEJERO | zip | Callejero oficial. Viales vigentes | `VIALES_SHP.zip` |
| CARTOGRAFIA | zip | Red Geotécnica de Madrid | `Geopuntos.zip` |
| CARTOGRAFIA | csv/z | Red Topográfica de Madrid (RTM) | `RTM.csv`, `Vertices_RTM.z`, `Visuales_RTM.z` |
| CULTURA | zip | Paisaje de la luz | `Pasiaje_Cultural.zip` |
| ELEVACIONES | zip | Modelo Digital del Terreno (MDT) 2025, 1 m | `MDT_1m.zip` |
| ELEVACIONES | z | MDT 2019 (**already COG**) | `MDT2019_COG.z` |
| ESTADISTICA | zip/z | Edad de la población (2020–2025, por distrito/barrio/sección) | `2020…2025_edad_poblacion.z` |
| ESTADISTICA | zip | Nivel de estudios de la población (2020–2025) | `2020…2025_estudios.zip` |
| JMDISTRITO | zip | El Rastro de Madrid (puestos) | `PuestosRastro.zip` |
| MALLAS | zip | Mallas cartográficas (cartografía / parcelario / PGOU97) | `Malla_*.zip` |
| MOBILIARIO_URBANO | zip | Bancos, Mesas, Papeleras, Áreas caninas, Áreas infantiles | `Bancos.zip`, … |
| MOVILIDAD | zip/z | Carriles por calle, Parquímetros SER, ZBEDEP Plaza Elíptica | `numero_carriles_calle.z`, `SHP_ZIP.zip`, … |
| OBRAS | zip | Túneles centralizados (BIE, hidrantes, columnas secas, SOS, gálibos, …), Accesibilidad aceras, Operación Asfalto | `*_TUNELES.zip`, … |
| OFICINAS_REGISTRO | zip | Oficinas de Registro | `OFICINAS_REGISTRO.ZIP` |
| PLANEAMIENTO | zip/pdf | PGOUM 85 (CRS/DSU/RGS), PG 1963, SIPLAM, Ordenanza 1972 | `PGOUM85_*.zip`, `PG63.zip`, `SIPLAM.zip` |
| POIS | zip | Puntos de interés de la ciudad | `POIS.zip` |
| SATELITE | zip | Isla de Calor Urbano (2020–2024) + variables multicriterio | `OT20xx_ICU.zip`, `Variables_ICU_20xx.zip` |
| VIVIENDA | z | Plan Adapta (subvenciones) | `PLAN_ADAPTA.z` |
| VUELOS | zip/z | Gráficos de vuelo (índices de fotogramas 2007/2011) | `GRAFICO_VUELO*.zip` |
| datos.madrid.es | csv | Callejero oficial — numeración (vigente / histórica) | `213605-*-callejero…csv` |
| externo | pdf/zip | ZBEDEP boletines (BOCM/BOE), Memoria de Madrid, luces de Navidad | — |

*(`.z` files are LZ-compressed single files, not standard ZIP — need `uncompress`/`gzip -d` handling, not `unzip`.)*

## 4. Best download path per data type

1. **Vector (the bulk)** → direct Shapefile ZIPs from `fsdescargas`. No service throttling, daily/periodic refresh.
   The DCAT gives us the exact URL + update frequency for each.
2. **Raster, official products** (orthophotos, satellite, vegetation) → `servpub.madrid.es/georaster` **WMS**,
   workspace-scoped. GetMap can emit `image/geotiff`, but that's reprojected/bounded — only good for small AOIs.
   For full COGs we want the **source rasters**; ask Madrid for them, or harvest the `_COG`/MDT files that already exist.
3. **Elevation** → `MDT_1m.zip` (2025) and `MDT2019_COG.z` are downloadable directly. The 2019 one is *already a COG*.
4. **Tabular** (statistics, callejero numbering) → CSV from `fsdescargas` / `datos.madrid.es`.
5. **The 753 archival imagery records** → only WMS/fototeca today; need source files from Madrid for clean COGs.

## 5. Conversion plan → cloud-native (proven locally)

Local toolchain present: **GDAL 3.12.2, DuckDB 1.5.3, mc**. Missing: `tippecanoe`, `pmtiles` (install when we do tiles).

**Vector → GeoParquet** — end-to-end test passed on `Bancos.zip` (76,822 points, EPSG:25830 → WGS84,
GeoParquet in <0.5 s, re-read in DuckDB = 76,822 rows):

```bash
ogr2ogr -f Parquet out.parquet in.shp \
  -t_srs EPSG:4326 -lco COMPRESSION=ZSTD -lco GEOMETRY_ENCODING=WKB
```

(Decision pending: reproject to WGS84/CRS84 vs. keep native EPSG:25830 and record `crs` in STAC.)

**Raster → COG:**

```bash
gdal_translate in.tif out_cog.tif -of COG \
  -co COMPRESS=DEFLATE -co PREDICTOR=2 -co OVERVIEWS=AUTO
```

**Vector tiles → PMTiles** (for big line/polygon layers like the callejero): `tippecanoe -o x.pmtiles …`.

## 6. Mapping to this Portolan repo

This repo publishes a **static Iceberg-REST + STAC + OGC API-Records** catalog to object storage (no server).
Per dataset we author `datasets/<id>.json` with a `representation` of `iceberg` / `raquet` / `remote-geoparquet`.
Proposed fit:

- **Vector SHP → GeoParquet** → load as **Iceberg** tables (or `remote-geoparquet` if we skip Iceberg initially).
- **Rasters → COG / raquet** → `raquet` representation (`read_raquet`).
- Keep Madrid attribution: `data_provider` = "Ayuntamiento de Madrid", license per datos.gob.es legal notice
  (`http://datos.gob.es/datos/?q=aviso-legal` — needs confirmation, likely CC-BY-style with attribution).

## 6b. Decisions taken & setup done (2026-06-05)

- **Tool:** `geoparquet-io` (CLI **`gpio`** 1.1.1, `uv tool install geoparquet-io`). `gpio convert in.shp out.parquet`
  = convert + bbox column + Hilbert sort + ZSTD + GeoParquet-1.1 validation, in one shot. Also `gpio extract`
  (files **and services** incl. WFS), `gpio add` (H3/S2/quadkey indices), `gpio partition`, `gpio pmtiles`,
  `gpio publish` (emits STAC), `gpio skills` (LLM guidance).
- **CRS policy: keep native EPSG:25830** (honest to source). Verified `gpio convert` preserves it
  (`geom` typed `GEOMETRY('EPSG:25830')`). Reproject only if we hit an incompatibility.
- **Representation: remote-GeoParquet for the first cut**, Iceberg as the goal. Open risk = whether Iceberg's
  geometry type respects a non-4326 CRS; remote-GeoParquet sidesteps it cleanly.
- **Storage: Google Cloud Storage**, bucket **`gs://carto-portolan-madrid`**, project `cartobq`,
  region **`europe-southwest1` (Madrid — data resides in Spain)**. One bucket, prefix per catalog:
  `…/madrid-city/` and (later) `…/comunidad-madrid/`.
- **License:** Madrid only references the generic `datos.gob.es` legal notice (reuse with attribution); no
  specific SPDX. Prototype → record the URL, leave SPDX blank.
- **Proven end-to-end:** `Bancos.zip` → `gpio convert` (EPSG:25830) →
  `gs://carto-portolan-madrid/madrid-city/data/mobiliario_urbano_bancos/…parquet` → DuckDB query over `gs://`
  (authenticated) = 76,822 rows, attribute queries OK. The full chain works.
- **Open: anonymous read.** Portolan needs anonymous public read. Granting `allUsers:objectViewer` was not done
  (security-weakening; needs explicit go-ahead, and CARTO org policy may block it). Decision pending — see §7.

## 7b. Comunidad de Madrid (IDEM) — second catalog (separate repo, explored 2026-06-05)

Catalog: `https://idem.comunidad.madrid/catalogocartografia/srv/spa/catalog.search#/home` — **GeoNetwork**,
**1,380 CSW records** (`…/srv/spa/csw`; the JSON API `…/srv/api/search` is 403, use CSW). Two realities:

- The **catalog heavily harvests the City of Madrid** (records point at `geoportal.madrid.es`,
  `servpub.madrid.es`, `sigma.madrid.es`) → must **dedupe vs the city catalog**.
- The **distinct regional asset** is the Comunidad's own GeoServer **`geoidem`**
  (`https://idem.comunidad.madrid/geoidem/ows?service=WFS`): **248 feature types** across 21 workspaces —
  ZonasRiesgo (48), UsoDelSuelo (39), Zonas (38), ServiciosPublicos (36), BTA (20, base topográfica),
  RedesTransporte (10), UnidadesAdministrativas (9), LugaresProtegidos (9), InstalacionesMedioAmbiente (8),
  Hidrografia (5), Elevaciones (4), CubiertaTerrestre, Edificios, Geología, Hábitats, etc.
- **Conversion path for the region = WFS → `gpio extract`** (vs the City's SHP-ZIP → `gpio convert`).
- `sigma.madrid.es` = the City's **ArcGIS Server** (`/hosted/services/<…>/MapServer/WMSServer`).

## 6c. Proof build + CRS test (2026-06-05) — multi-format, native EPSG:25830

Built a 3-dataset proof set across geometry types, each in **three representations**, mirroring the
Helsinki model but keeping **native EPSG:25830**, published to `gs://carto-portolan-madrid/madrid-city/`:

| Dataset | Geom | Rows | v2 Iceberg | v3 Iceberg | Remote GeoParquet |
|---|---|---|---|---|---|
| `mobiliario_urbano_bancos` | Point | 76,822 | ✓ | ✓ | ✓ |
| `alumbrado_canalizaciones` | Line | 68,081 | ✓ | ✓ | ✓ |
| `agencia_categorias_fiscales` | Polygon | 233 | ✓ | ✓ | ✓ |

Plus a `catalog.datasets` **stac-geoparquet** index and a static **Iceberg-REST** surface (prefix `sdi`).
Builder: `/tmp/madrid_build.py` (run with the iceberg-geo-testbed venv). Tool: `gpio convert` (native CRS).

**Live endpoint:** `https://storage.googleapis.com/carto-portolan-madrid/madrid-city`

```sql
INSTALL iceberg;LOAD iceberg;INSTALL httpfs;LOAD httpfs;INSTALL spatial;LOAD spatial;
ATTACH 'mad' (TYPE iceberg, ENDPOINT 'https://storage.googleapis.com/carto-portolan-madrid/madrid-city',
              AUTHORIZATION_TYPE 'none');
SHOW ALL TABLES;                                    -- v2.* , v3.* , catalog.datasets
SELECT id, json_extract_string(properties,'$.crs') FROM mad.catalog.datasets;
-- native-CRS distance in METRES, NO ST_Transform:
SELECT round(min(ST_Distance(geom, ST_Point(440742,4474264)))) AS nearest_bench_m
FROM mad.v3.mobiliario_urbano_bancos;
```

**CRS verdict (DuckDB):** keeping native EPSG:25830 **works across all three representations**.
v3 native geometry reads as `GEOMETRY('EPSG:25830')` — CRS preserved; v2 + v3 distance queries run in
metres with no transform (29 m to a test point). Anonymous `ATTACH`, `iceberg_scan`, and `read_parquet`
all work. **Caveat (analytic):** BigQuery `GEOGRAPHY` is WGS84-only → a native-projected geom won't load
there; Snowflake `GEOMETRY` supports SRID → expected to work. The "reprojected v3 variant" fallback is thus
only needed to serve WGS84-only engines (BigQuery / CARTO-on-BQ), not DuckDB/Snowflake.

## 6d. Raster / imagery → COG (2026-06-05)

Madrid's catalog is **raster-heavy** (orthophotos 209, vuelos 237, satélite 65, vegetación, etc.), served
by the **`servpub.madrid.es/georaster`** GeoServer. **WCS is DISABLED** there → no source-coverage
extraction; programmatic access is **WMS GetMap** only. Two COG paths, both proven:

**A) Direct-download source GeoTIFFs → COG (cleanest):**
- **Urban Heat Island (ISLA_CALOR / ICU)** — `OT<year>_ICU.zip` (the index) + `Variables_ICU_<year>.zip`
  (≈8 surfaces: land-surface-temp `TST`, brightness, distances, mean altitude…), **years 2020–2024**.
  Confirmed GeoTIFFs (WGS84 / UTM 30N ≈ EPSG:32630, 15 m). Converted `ICU2024` → valid COG (DEFLATE, 512²
  tiles, overviews) in one `gdal_translate -of COG`.
- **MDT elevation** — `ELEVACIONES/MDT_1m.zip` (2025, 1 m) and `MDT2019_COG.z` (**already a COG**). Direct
  download (intermittently in maintenance).

**B) WMS GetMap → GeoTIFF → COG (for the imagery bulk — orthophotos/satellite/vegetation/noise/historic):**
- GeoServer layers, e.g. `georaster/ORTOFOTOS_COMPLETAS/ORTO_2016_10_20/ows`. Workspaces seen:
  ORTOFOTOS_COMPLETAS (1954→2024), ORTOFOTOS_PARCIALES, IMAGENES_SATELITE (2002→2020), VEGETACION,
  ELEVACIONES, CONTROL_ACUSTICO (noise maps Lden/Lnight), CARTOTECA (historic).
- `GetMap&format=image/geotiff&crs=EPSG:25830` returns **correctly georeferenced, native-resolution**
  tiles (ORTO_2016 = **0.25 m**, 3-band RGB). Path = tile WMS GetMap at native res → mosaic →
  `gdal_translate -of COG`. **Caveat:** full-city 25 cm orthophotos are enormous (~600 km² → ~10⁹ px/band);
  realistic options = request source mosaics from Madrid, or build COGs at a chosen resolution / per-AOI.

So: **yes, lots of COG-able raster.** Heat-island + MDT are clean today; orthophoto/satellite COGs are
feasible via WMS but want either Madrid's source files or a deliberate resolution/scope choice.

## 6e. Autonomous full import (2026-06-05)

Imported **everything reachable** and registered **metadata for all 967 datasets** (per the "list it even
without data" directive). Pipeline (committed under `tools/madrid/`): `fetch_convert.py` (resumable
download→`gpio convert`/COG/CSV, crash-safe `build_ledger.json`) → `build_madrid_catalog.py` (multi-format
Iceberg + stac index) → `publish.sh` (GCS).

Results:
- **1,006 STAC index rows** in `catalog.datasets` (967 datasets; multi-layer archives expand to more rows).
- **80 materialized layers across 41 datasets**: 74 vector → **Iceberg v2 + v3 + remote GeoParquet** (native
  EPSG:25830); 4 raster → **COG**; 2 non-spatial → **Iceberg `tab`** (`portolan:geospatial:false`).
- **151 Iceberg tables** + 75 remote GeoParquet + 4 COGs (~1.3 GB) on `gs://carto-portolan-madrid/madrid-city/`.
- **926 metadata-only rows** — every non-downloadable / maintenance dataset is listed with
  `materialized:false` and a `data_status` (`maintenance` ×9, `metadata_only`, `no_layers`, `extract_failed`,
  `build_error`) plus a `source` link, so it shows in the catalog and can be filled in later.

Metadata completeness (Portolan): each Iceberg table carries full properties (`title`, `theme`, OSI
`semantics`, `license`, `crs`, `geo`) **and per-column `doc`** (self-describing schema); materialized
index rows carry the **STAC Iceberg** extension (`iceberg:catalog_uri`/`table_id`/`current_snapshot_id`);
the root **`catalog.json`** carries the **git-backed-catalog** extension (`git:repository`, vcs/issues/
monitor links) alongside `versions.json`. Mutable files (surface, metadata, index) are uploaded
`Cache-Control: no-cache` so in-place overwrites propagate immediately. *(Not generated: the alternate
per-collection `collection.json` + OGC API-Records file surface — this catalog uses the
`iceberg-rest-static` model: stac-geoparquet index + Iceberg-REST surface.)*

Re-run anytime (resumable): re-run `fetch_convert.py` (skips done, retries maintenance once the window
clears) → `build_madrid_catalog.py` → `publish.sh`. Querying the whole catalog (materialized + listed):

```sql
ATTACH 'mad' (TYPE iceberg, ENDPOINT 'https://storage.googleapis.com/carto-portolan-madrid/madrid-city', AUTHORIZATION_TYPE 'none');
SELECT json_extract_string(properties,'$.data_status') st, count(*) FROM mad.catalog.datasets GROUP BY 1;
SELECT id FROM mad.catalog.datasets WHERE json_extract_string(properties,'$.materialized')='true';
```

## 7. Open decisions (for you)

1. **Sovereign bucket** — you're still choosing the provider/region. The repo template assumes UpCloud
   (`s3_endpoint` + `mc` alias); swap in your endpoint once chosen.
2. **Scope of v1** — start with the 64 downloadable datasets (mostly vector), or also pursue source rasters
   from Madrid for the imagery archive?
3. **CRS policy** — reproject everything to WGS84/CRS84, or keep native EPSG:25830?
4. **Iceberg vs. plain remote-GeoParquet** for vectors in the first cut.
5. **Refresh** — many layers update daily/weekly; do we want a scheduled re-harvest, or one-time import?
6. **License confirmation** — verify the exact reuse license/attribution Madrid requires.
