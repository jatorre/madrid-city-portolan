# Madrid city catalog — build pipeline

End-to-end pipeline that turns Madrid's geoportal (IDEAM) into a multi-format, cloud-native
Portolan catalog on GCS (`gs://carto-portolan-madrid/madrid-city/`). All vector data keeps its
**native CRS EPSG:25830**. See `../../MADRID_INVENTORY.md` for the full investigation + results.

## Files
- `manifest.json` — all **967** datasets parsed from the DCAT feed (id, title, description,
  keywords, theme, category, distributions, kind). Source of truth for what exists.
- `fetch_convert.py` — resumable: downloads each downloadable dataset and converts vector→GeoParquet
  (`gpio`, native CRS), raster→COG (`gdal_translate -of COG`), CSV→parquet. Writes `build_ledger.json`
  (per-dataset status, crash-safe, skip-if-done). Maintenance/failed downloads stay non-`done` and
  become metadata-only rows downstream.
- `build_ledger.json` — last run's state (what materialized vs maintenance/metadata-only).
- `build_madrid_catalog.py` — builds, per materialized layer, Iceberg **v2** (WKB+bbox) + **v3**
  (native `geometry(EPSG:25830)`) + remote GeoParquet; rasters→COG assets; CSV→`tab` Iceberg
  (`portolan:geospatial:false`); and **one `catalog.datasets` stac-geoparquet index covering all 967**
  (materialized rows carry assets; the rest carry `data_status` + `source`). Emits a static
  Iceberg-REST surface (prefix `sdi`).
- `publish.sh` — uploads the staged tree + IRC surface to GCS (bucket is public-read).

## Run (resumable — safe to re-run; picks up maintenance datasets once Madrid's window clears)
```bash
# deps: gpio (uv tool install geoparquet-io), GDAL, DuckDB, gcloud (authed), and the
# iceberg-geo-testbed venv for pyiceberg/geoarrow.
python3 fetch_convert.py                                   # -> /tmp/mad_data + build_ledger.json
/path/to/iceberg-geo-testbed/.venv/bin/python build_madrid_catalog.py   # -> /tmp/madrid_catalog
bash publish.sh                                            # -> gs://carto-portolan-madrid/madrid-city
```
Paths are currently under `/tmp` (manifest at `/tmp/mad_manifest.json`, staging at
`/tmp/madrid_catalog`); copy `manifest.json` there or adjust the constants at the top of each script.

## Query the published catalog (anonymous)
```sql
INSTALL iceberg;LOAD iceberg;INSTALL httpfs;LOAD httpfs;INSTALL spatial;LOAD spatial;
ATTACH 'mad' (TYPE iceberg,
  ENDPOINT 'https://storage.googleapis.com/carto-portolan-madrid/madrid-city',
  AUTHORIZATION_TYPE 'none');
SELECT json_extract_string(properties,'$.data_status') st, count(*) FROM mad.catalog.datasets GROUP BY 1;
```
