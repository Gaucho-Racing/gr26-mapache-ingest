# GR26 Mapache Ingest

> [!NOTE]
> This service originally lived in the main [Mapache repository](https://github.com/Gaucho-Racing/Mapache). It was split into this repository as an archival snapshot while Mapache undergoes a complete v4 rewrite.

GR26 Mapache Ingest translates live and cold-storage telemetry from Gaucho Racing's GR26 car into Mapache's current signal representation.

## How it works

### Live telemetry

1. The service consumes MQTT messages on the GR26 topic hierarchy, with CAN frames addressed as `gr26/{vehicle_id}/{node_id}/{can_id}`.
2. Each payload contains an 8-byte big-endian microsecond timestamp, a 2-byte upload key, and the raw CAN payload.
3. The upload key is validated against the Mapache vehicle service.
4. A GR26-specific decoder converts known CAN messages into Mapache signals. Unknown or malformed frames are retained with decode-status metadata.
5. Raw frames and decoded signals are written to ClickHouse.
6. Decoded signals are published to `query/live/{vehicle_id}/{signal_name}` for Mapache's live-data service.

### Shelter batches

GR26 also handles Epic Shelter cold-storage files. Shelter batch notifications enqueue Foreman jobs, which download Parquet files from S3, decode their CAN rows through the same frame pipeline, and bulk-insert the results into ClickHouse.

## Stored values

Decoded values use the Mapache v3 `Signal` schema in the shared ClickHouse `signal` table. A signal contains an ID, vehicle ID, node-prefixed name, decoded value, raw value, microsecond timestamp, production time, and ingestion time.

Every accepted CAN frame is also stored in `gr26_can` with its vehicle, node, CAN ID, raw bytes, upload key, timestamps, and JSON decode metadata. Ping data is stored in the shared ClickHouse ping schema.

## Running

```sh
go build ./...
```

The service requires MQTT and can optionally connect to ClickHouse, Foreman, S3, and Mapache's vehicle-authentication endpoint. See `config/config.go` for the accepted settings.
