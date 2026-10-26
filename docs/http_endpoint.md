# HTTP Data Distribution Endpoint

The `qData_service` now exposes an HTTP endpoint for data distribution alongside the existing gRPC endpoints. This allows clients to fetch historical market data via standard RESTful JSON requests.

## Address

The endpoint is accessible via HTTPS at:
`https://uptoken.cloud/api/v1/distribution/ohlc`

## `GET /api/v1/distribution/ohlc`

Fetches historical OHLC (Open, High, Low, Close) bar data.

**Query Parameters:**
- `symbol` (string, optional): The asset symbol (e.g., `WINJ26`).
- `timeframe` (string, optional): The timeframe of the bars (e.g., `M1`, `M5`, `H1`).
- `start_time` (int, optional): Start Unix timestamp (inclusive).
- `end_time` (int, optional): End Unix timestamp (inclusive).
- `limit` (int, default `1000`): Maximum number of records to return.
- `offset` (int, default `0`): Pagination offset.
- `order_desc` (boolean, default `true`): Order by timestamp descending.

**Response Schema:**
Returns a JSON array of `OHLCBar` objects. Each object contains the following fields:

| Field | Type | Description |
|---|---|---|
| `id` | integer | (Optional) Database record ID |
| `symbol` | string | (Optional) Asset symbol |
| `timeframe` | string | (Optional) Bar timeframe (e.g., `M1`, `M5`, `H1`) |
| `source_id` | string | (Optional) Data source identifier |
| `ingested_at` | string | (Optional) Ingestion timestamp |
| `time` | integer | Unix timestamp in seconds |
| `time_utc` | string | Timestamp in ISO 8601 format (UTC) |
| `time_sp` | string | Timestamp in ISO 8601 format (São Paulo) |
| `open` | number | Open price |
| `high` | number | High price |
| `low` | number | Low price |
| `close` | number | Close price |
| `tick_volume` | integer | Tick volume |
| `spread` | integer | Spread |
| `real_volume` | integer | Real volume |

**Example Response:**
```json
[
  {
    "time": 1765886520,
    "time_utc": "2025-12-16T12:02:00+00:00",
    "time_sp": "2025-12-16T09:02:00-03:00",
    "open": 166675.0,
    "high": 166675.0,
    "low": 166675.0,
    "close": 166675.0,
    "tick_volume": 1,
    "spread": 5,
    "real_volume": 1
  }
]
```

## Integration

The endpoint is registered in the main FastAPI application (`src/qData/main.py`) under the `/api/v1/distribution` prefix. It uses the same underlying database accessors (`fetch_ohlc`) as the gRPC service, ensuring consistent data access across protocols.

