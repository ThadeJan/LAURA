# LAURA

PostGIS-backed data foundation for documenting tree service work on site.

## Start the database

Requirements: Docker with Compose.

```bash
docker compose up -d
docker compose exec db psql -U laura -d laura
```

The database is available on `localhost:5432` with:

| Setting | Value |
| --- | --- |
| Database | `laura` |
| User | `laura` |
| Password | `laura_dev` |
| Schema | `app` |

The development password is intentionally simple and must be replaced before deployment. Initialization scripts run only when the PostgreSQL volume is first created. To recreate the development database from scratch:

```bash
docker compose down -v
docker compose up -d
```

## Data model

- `customer` and `site` hold customer, contact, address, and map-location data.
- `land_parcel` and `land_use_period` store farm boundaries, computed area, soil metadata, and non-overlapping land-use history.
- `produce_catalog` and `production_batch` store crops, varieties, seasons, planting/harvest dates, and planned versus actual yields.
- `employee` stores crews, labor cost, billing rate, and certifications.
- `asset` stores machinery and hourly operating cost.
- `price_item` stores sell and cost prices with non-overlapping effective time ranges.
- `work_order` is the simple field-work center: customer, site, status, schedule, and notes.
- `time_entry`, `asset_usage`, and `expense` capture job costs and actual work.
- `spatial_observation` stores trees, hazards, boundaries, photos, and other geometry with the time it was observed. Geometry uses WGS 84 (`EPSG:4326`).
- `subcontractor` and `subcontract_booking` book an external service against one spatial observation, including scheduled time, quantity, agreed rate, actual cost, invoice, and status.
- `work_order_financials` provides revenue, cost, and gross margin for reporting.

The schema uses `tstzrange` for all time-bound records. This preserves both the event time and uncertainty about when a work period began or ended, and it makes overlap queries/indexes available to a future field GUI.

## Field GUI query starting points

The frontend can stay small by centering on three queries:

```sql
-- Today's work queue
SELECT work_order_id, work_order_number, title, status, scheduled_during
FROM app.work_order
WHERE scheduled_during && tstzrange(now(), now() + interval '1 day', '[)')
ORDER BY lower(scheduled_during);

-- Map observations for one job
SELECT observation_id, observation_type, attributes, photo_uri,
	   ST_AsGeoJSON(geometry) AS geometry
FROM app.spatial_observation
WHERE work_order_id = $1
ORDER BY created_at;

-- Cost and margin summary for one job
SELECT * FROM app.work_order_financials WHERE work_order_id = $1;
```

## Files

- `docker-compose.yml` runs PostgreSQL 16 with PostGIS 3.4.
- `db/init/001_schema.sql` creates extensions, tables, constraints, indexes, triggers, and reporting view.
- `db/init/002_seed.sql` inserts safe example data for local development.
