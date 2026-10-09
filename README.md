# Geospatial File Measurement API

A small FastAPI service. You upload a zipped Shapefile or a KML file, it reads
every feature, and it gives back the area of each polygon and the length of each
line, in metres, calculated in a projected coordinate system rather than in
degrees.

```
POST /api/files/                      upload and process a file
GET  /api/files/{id}/                 file info (status, CRS, feature count)
GET  /api/files/{id}/measurements/    per-feature geometry, properties and measurements
```

## Contents

1. [Setup](#setup)
2. [API](#api)
3. [Architecture](#architecture)
4. [Design decisions](#design-decisions)
5. [Limitations](#limitations)
6. [Learnings](#learnings)
7. [Future scope](#future-scope)

## Setup

Python 3.11 or newer. All dependencies ship binary wheels (including GEOS and
PROJ), so there is nothing to install at the OS level.

```bash
python -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate
pip install -r requirements.txt

uvicorn app.main:create_app --factory --reload
```

The API is now on http://127.0.0.1:8000 and the interactive docs are at
http://127.0.0.1:8000/docs.

Data is stored in a SQLite file, `data/geo_measure.db` by default, created on
first start.

### Configuration

| Variable | Default | Meaning |
|---|---|---|
| `GEO_DB_PATH` | `data/geo_measure.db` | SQLite file location |
| `GEO_MAX_UPLOAD_MB` | `50` | Largest accepted upload |
| `GEO_MAX_UNZIPPED_MB` | `250` | Largest total size of the contents of an uploaded zip |

### Docker

```bash
docker build -t geo-measure-api .
docker run -p 8000:8000 -v geo-data:/srv/data geo-measure-api
```

### Tests

```bash
pip install -r requirements-dev.txt
pytest
```

The measurement tests check results against `pyproj.Geod` (geodesic
calculations on the WGS84 ellipsoid) as an independent reference, so they cover
the CRS handling and not only the plumbing.

### Sample data

`sample_data/survey.kml` has two polygons (one with a hole), a line, a point and
a mixed-geometry feature. For shapefiles:

```bash
python scripts/make_sample_data.py
```

writes `sample_data/plots_wgs84.zip` (EPSG:4326) and `sample_data/plots_utm43n.zip`
(EPSG:32643).

## API

### Upload: `POST /api/files/`

Multipart form data.

| Field | Required | Notes |
|---|---|---|
| `file` | yes | `.zip` containing one Shapefile, or a `.kml` |
| `crs` | no | Shapefile only. Used when the zip has no `.prj`, and overrides the `.prj` if both exist. Anything pyproj accepts, e.g. `EPSG:32643`. Ignored for KML, which is always WGS84. |

```bash
curl -F "file=@sample_data/survey.kml" http://127.0.0.1:8000/api/files/
```

```json
{
  "id": "5f0c6e9a2b7d4c1e8a3f90d1b2c4e6f7",
  "filename": "survey.kml",
  "file_type": "kml",
  "status": "COMPLETED",
  "feature_count": 5,
  "crs": "EPSG:4326",
  "error": null,
  "created_at": "2026-10-10T09:12:44.512093Z"
}
```

Responses: `201` created, `413` over the size limit, `415` not a `.zip`/`.kml`,
`422` the file could not be processed. A `422` body looks like
`{"detail": "...", "file_id": "..."}`; the file is also kept with status `FAILED`
and the reason in `error`, so it can be looked up later. (FastAPI's own request
validation errors also use 422, with its usual list-shaped `detail`.)

### File info: `GET /api/files/{id}/`

Same shape as the upload response. `404` for an unknown id.

### Measurements: `GET /api/files/{id}/measurements/`

| Query parameter | Default | Meaning |
|---|---|---|
| `limit` | 100 | Features per page (1 to 1000) |
| `offset` | 0 | Features to skip |
| `include_geometry` | `true` | Set `false` to leave geometries out of big responses (the `geometry` field is then `null`) |

```bash
curl "http://127.0.0.1:8000/api/files/5f0c6e9a2b7d4c1e8a3f90d1b2c4e6f7/measurements/?limit=2"
```

Abridged, values rounded:

```json
{
  "file_id": "5f0c6e9a2b7d4c1e8a3f90d1b2c4e6f7",
  "crs": "EPSG:4326",
  "total_features": 5,
  "limit": 2,
  "offset": 0,
  "summary": {
    "total_area_sq_m": 149045.0,
    "total_length_m": 1192.2,
    "by_geometry_type": {"Polygon": 2, "LineString": 1, "Point": 1, "GeometryCollection": 1},
    "by_status": {"OK": 3, "NOT_APPLICABLE": 1, "UNSUPPORTED": 1}
  },
  "features": [
    {
      "index": 0,
      "geometry_type": "Polygon",
      "crs": "EPSG:4326",
      "geometry": {"type": "Polygon", "coordinates": [[[73.85, 18.52, 0.0], "..."]]},
      "properties": {"name": "Plot A", "owner": "Patil", "crop": "sugarcane"},
      "measurement": {
        "status": "OK",
        "area_sq_m": 46754.1,
        "perimeter_m": 865.2,
        "length_m": null,
        "area_crs": "LAEA(lat_0=18.5, lon_0=73.9)",
        "length_crs": "AEQD(lat_0=18.5, lon_0=73.9)",
        "warnings": [],
        "message": null
      }
    }
  ]
}
```

`geometry` is GeoJSON in the file's own CRS and `crs` says which CRS that is.
`index` is the feature's position in the file (document order for KML, record
order for a shapefile).

Per-feature `measurement.status`:

| Status | Meaning |
|---|---|
| `OK` | Measured. Polygons get `area_sq_m` and `perimeter_m`, lines get `length_m`. |
| `NOT_APPLICABLE` | Points. Nothing to measure. |
| `UNSUPPORTED` | No geometry, empty geometry, or a type we do not measure (e.g. `GeometryCollection`). `message` says which. |
| `ERROR` | The geometry was malformed or could not be transformed. `message` says why. |

`summary.total_length_m` only adds up line features; polygon perimeters are not
included in it. The summary always covers the whole file, regardless of paging.

If a file is not `COMPLETED`, this endpoint returns `409`.

## Architecture

```
app/
  main.py              app factory: wires settings, repository and router
  config.py            settings read from environment variables
  api/files.py         the three endpoints; HTTP concerns only (limits, status codes)
  schemas.py           pydantic response models
  services/ingest.py   orchestrates one upload: parse -> resolve CRS -> measure -> store
  parsers/
    kml.py             KML  -> list of features (GeoJSON-style dicts)
    shapefile_zip.py   zip  -> list of features, CRS taken from .prj
    base.py            RawFeature / ParsedFile
  geo/
    crs.py             CRS resolution and local projection factory
    measure.py         Measurer: one feature -> Measurement
  db.py                SQLite repository
tests/
```

### File-processing flow

1. The endpoint checks the extension, reads the upload in chunks and stops at the
   size limit.
2. `ingest.process_upload` inserts a file row with status `PROCESSING`.
3. The matching parser turns the bytes into features. Parsers know nothing about
   shapely or measuring; they output plain GeoJSON-style dicts plus properties,
   and the CRS definition if the format has one (`.prj` text, or EPSG:4326 for KML).
   A zip is read in memory, never extracted to disk.
4. `resolve_crs` builds a `pyproj.CRS` from the `.prj` / the `crs` override.
5. Every feature goes through `Measurer.measure`. One bad feature produces a
   feature-level `ERROR` or `UNSUPPORTED` and the rest carry on.
6. Features and the file summary are written in one transaction and the file is
   marked `COMPLETED`. Any file-level failure (not a zip, broken XML, no CRS)
   marks it `FAILED` with the reason and the API answers `422`.

### Measurement flow

For one feature, `Measurer.measure`:

1. builds a shapely geometry and drops Z values (altitude is not used)
2. returns early for points (`NOT_APPLICABLE`) and for types it does not measure
   (`UNSUPPORTED`)
3. flags invalid geometry (self-intersections and so on) in `warnings`
4. converts coordinates to WGS84 lon/lat if the source CRS is something else
5. takes the centroid and rounds it to 0.1 degree
6. projects into a local projection centred there and measures in metres

### CRS handling

The rule is that nothing is ever measured in degrees, and nothing is measured in
the source CRS either, even when it is already projected. Instead each feature is
re-projected into a projection chosen for the measurement being taken:

| Measurement | Projection | Why |
|---|---|---|
| Polygon area | Lambert azimuthal equal-area (`laea`) centred on the feature | Preserves area by construction, so the result is exact regardless of where the feature sits |
| Line length, polygon perimeter | Azimuthal equidistant (`aeqd`) centred on the feature | Distances measured from the centre are true, and distortion elsewhere grows with the square of the distance from it, which is tiny for survey-sized features |

Both are built on the WGS84 datum with a `+proj=...` string, e.g.
`+proj=laea +lat_0=18.5 +lon_0=73.9 +datum=WGS84 +units=m`. The projection used is
reported back in `area_crs` and `length_crs` so a result can be reproduced.

Source CRS comes from the `.prj` (or the `crs` form field). A label like
`EPSG:32643` is reported when pyproj can identify one, otherwise the CRS name.
Axis order is always treated as x = lon/easting, y = lat/northing.

## Design decisions

**FastAPI rather than Django + DRF.** The service is three endpoints and a file
upload. There is no admin, auth, or relational model that would benefit from
Django's ORM. FastAPI gives request validation and an OpenAPI page for little code.

**pyshp, shapely, pyproj and a small KML reader, instead of GeoPandas / Fiona / GDAL.**
GDAL would read everything (KMZ, GeoPackage, ...) but is the painful part of most
geospatial installs. These libraries are all `pip install` with wheels, which
keeps setup to two commands. The cost is a narrower set of formats: the KML
reader handles Placemarks with Point, LineString, LinearRing, Polygon (with
holes) and MultiGeometry, and nothing else.

**Local equal-area / equidistant projections instead of one UTM zone.** UTM is
the obvious choice. Problems with it here: a file can span zones,
the polar regions need a different system, grid distances are off by up to about
0.1% depending on position in the zone, and UTM is not equal-area so area needs a
correction anyway. A projection centred on each feature avoids the zone logic and
gives an area that is exact. The `Transformer` objects are cached on the
rounded centre because creating them is the slowest step.

**Why not just use geodesic formulas (`pyproj.Geod`)?** They would be the most
accurate answer for lengths and I use them in the tests as a reference. The brief
asks for a transformation to a projected CRS, and a projected measurement is
easier to explain and to reproduce with any GIS tool, so I kept that and use
`Geod` to check it.

**Reproject even when the data is already projected.** Web Mercator
(EPSG:3857) is projected, but measuring in it inflates areas by `1 / cos²(lat)`
(2x at 45° latitude, 4x at 60°). Always going through WGS84 and a local projection
means the code does not need to know which projections are safe to measure in.
There is a test for exactly this case.

**Invalid geometry is reported, not repaired.** A bow-tie polygon still gets a
number, with a warning saying why it is invalid. `make_valid` would change the
geometry and therefore the numbers without the user asking.

**Measurements are computed once, at upload.** The data is immutable after
upload, so reads are plain SQL lookups with pagination and the summary is
precomputed.

**Synchronous processing.** Parsing and measuring happen inside the request.
That is fine at the 50 MB upload cap and keeps the system small. The `status`
field (`PROCESSING` / `COMPLETED` / `FAILED`) is there so moving this onto a
background worker later does not change the API.

**SQLite behind a small repository class.** No setup, and good enough for this
scale. Everything that touches SQL lives in `db.py`, so swapping in Postgres/PostGIS
means rewriting one file. Each call opens its own short-lived connection, so there
is no connection sharing across FastAPI's worker threads.

**Parsers return plain dicts, not shapely objects.** It keeps the file-format code
independent of the measuring code and lets the KML parser be unit-tested without
any geo libraries installed.

**Failures are scoped.** A file-level problem fails the upload (422). A
feature-level problem (bad coordinates in one Placemark, an unsupported
geometry type) only affects that feature's `measurement`.

**Upload handling is defensive.** Size limit while streaming, total uncompressed
zip size limit, zip members read in memory instead of extracted (no zip-slip),
KML parsed with entity resolution and network access turned off.

## Limitations

- Geometry that crosses the antimeridian (longitude ±180) in a single feature is
  not handled: the coordinates are treated as a plain planar path.
- A single feature that spans a very large area (hundreds of kilometres) is
  measured with more distortion than the survey-sized features this is aimed at.
- Only the formats listed above. No KMZ, GeoJSON, GPX or GeoPackage.
- KML `ExtendedData` values come back as strings, since KML does not reliably
  carry types.
- A shapefile zip must hold exactly one shapefile.
- The whole upload is held in memory while it is processed.
- No authentication, and no endpoint to list or delete files.

## Learnings

- Shapefile polygon rings carry meaning in their winding order: clockwise is an
  outer ring, counter-clockwise is a hole. pyshp relies on that when it builds
  GeoJSON, so test fixtures have to write outer rings clockwise or they are read
  back as holes.
- KML is `lon,lat,alt`, not `lat,lon`, and real files have stray spaces after
  the commas even though the spec says no spaces. The parser tolerates that.
- With an equal-area projection the choice of centre does not affect the area at
  all, only the shape distortion. That is what made it safe to round the centre
  and cache transformers.
- UTM is not as accurate as it looks for lengths. Grid distance is scaled by a
  factor that varies across the zone (about 0.9996 on the central meridian, over
  1.0009 at the zone edge on the equator), which is why lengths use a local projection.
- Writing the tests against `pyproj.Geod` rather than hard-coded numbers made the
  expected values independent of my own projection code.

## Future scope

- Move processing to a background worker (Celery / RQ) and return `202` with a
  status URL for large files.
- PostgreSQL + PostGIS for storage, which would allow spatial queries
  (features within a bounding box, intersecting a given polygon).
- More formats: KMZ, GeoJSON, GPX, GeoPackage; possibly through GDAL behind the
  same parser interface.
- Optional `units=` parameter (hectares, acres, kilometres, feet).
- Accuracy flag: compare the projected result with a geodesic one and warn when
  they differ by more than a threshold.
- Antimeridian handling by splitting geometry at ±180.
- Streaming uploads to disk instead of memory, and a configurable feature cap.
- Auth, per-user quotas, list and delete endpoints, and Alembic migrations once
  the schema starts to change.
