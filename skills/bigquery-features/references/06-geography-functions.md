# Geography Functions

**Impact:** Medium

## What This Covers

BigQuery's GEOGRAPHY type and spatial functions for creating, transforming, and analyzing geospatial data using the S2 geometry library.

## Why It Matters

BigQuery's native geography support enables spatial analysis (distance calculations, containment checks, spatial joins) directly in SQL without exporting to a GIS tool. Geography-based clustering can dramatically reduce bytes scanned for location-scoped queries.

## Example

```sql
-- Create geography points from lat/lng
SELECT
  store_id,
  ST_GEOGPOINT(longitude, latitude) AS location
FROM `project.dataset.stores`;

-- Distance between two points (meters)
SELECT
  a.store_id,
  b.customer_id,
  ST_DISTANCE(a.location, b.location) AS distance_meters
FROM `project.dataset.stores` AS a
CROSS JOIN `project.dataset.customers` AS b
WHERE ST_DWITHIN(a.location, b.location, 5000);  -- within 5km, uses spatial indexing

-- Containment: points within a polygon
SELECT customer_id
FROM `project.dataset.customers`
WHERE ST_WITHIN(
  location,
  ST_GEOGFROMGEOJSON('{"type":"Polygon","coordinates":[[[-122.5,37.7],[-122.3,37.7],[-122.3,37.8],[-122.5,37.8],[-122.5,37.7]]]}')
);

-- Spatial JOIN with ST_INTERSECTS
SELECT
  t.trip_id,
  z.zone_name
FROM `project.dataset.trips` AS t
JOIN `project.dataset.zones` AS z
  ON ST_INTERSECTS(t.route_geo, z.boundary);

-- Load WKT data
SELECT ST_GEOGFROMTEXT('POINT(-122.4194 37.7749)') AS sf_location;

-- Geography-clustered table for fast spatial queries
CREATE TABLE `project.dataset.events_geo`
CLUSTER BY location
AS SELECT *, ST_GEOGPOINT(lng, lat) AS location
FROM `project.dataset.raw_events`;
```

## Edge Cases / Pitfalls

- **Coordinate order:** `ST_GEOGPOINT(longitude, latitude)` -- longitude first, latitude second. This is the most common mistake.
- **ST_DISTANCE units:** Always returns meters on the WGS84 ellipsoid. Divide by 1000 for kilometers.
- **Spatial JOIN performance:** Use `ST_DWITHIN` for distance-bounded joins instead of filtering `ST_DISTANCE` in WHERE -- it uses spatial indexing.
- **Geography clustering:** Cluster by GEOGRAPHY columns for tables frequently queried by location. This reduces bytes scanned significantly for spatial predicates.
- **GeoJSON parsing:** `ST_GEOGFROMGEOJSON` expects valid GeoJSON with coordinates in [longitude, latitude] order. Invalid geometry returns an error; use `SAFE.ST_GEOGFROMGEOJSON` to return NULL instead.
- **Precision:** BigQuery uses the S2 geometry library with snapping to the S2 cell grid. Very small geometries (<1cm) may lose precision.
