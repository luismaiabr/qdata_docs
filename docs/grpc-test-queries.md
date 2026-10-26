# gRPC Test Queries

This guide shows how to make repeatable test queries against the OHLC gRPC service and compare the results with local source-of-truth data.

## Endpoints

Use the native gRPC endpoint for command-line or Python clients:

- Production: `uptoken.cloud:50052`
- Local direct app access: `localhost:50051`

Do not use `8443` for native gRPC clients. That port is reserved for gRPC-Web/browser traffic.

## Expected credentials format

Store test credentials in `.env` using these keys:

```env
USERNAME_TEST=your_username_here
PASSWORD_TEST=your_password_here
```

The auth flow is:

1. Call `POST https://uptoken.cloud/api/v1/auth/token` with the username and password from `.env`.
2. Use the returned JWT as gRPC metadata:

```text
authorization: Bearer <access_token>
```

## OHLC query shape

The OHLC service method is `qData.OHLCService/StreamOHLC`.

Request fields:

- `symbol`: for example `WINM26`
- `timeframe`: for example `M1`
- `start_time`: Unix seconds
- `end_time`: Unix seconds
- `limit`: optional, use `1000` for test queries

## Example with grpcurl

If you want to inspect the raw stream, first get a token:

```bash
TOKEN=$(curl -s https://uptoken.cloud/api/v1/auth/token \
  -H 'Content-Type: application/json' \
  -d '{"username":"'"$USERNAME_TEST"'","password":"'"$PASSWORD_TEST"'"}' \
  | jq -r '.access_token')
```

Then query the OHLC stream over native gRPC:

```bash
grpcurl -insecure \
  -H "authorization: Bearer $TOKEN" \
  -import-path protos \
  -proto protos/ohlc.proto \
  -d '{
    "symbol": "WINM26",
    "timeframe": "M1",
    "start_time": 1778008800,
    "end_time": 1778786400,
    "limit": 1000
  }' \
  localhost:50051 \
  qData.OHLCService/StreamOHLC
```

For production, replace `localhost:50051` with `uptoken.cloud:50052`.

## Example with the repo client

The repository already includes a Python client pattern in `scripts/test_grpc_client.py`.

For OHLC test queries, the important pieces are:

- create a secure channel for `uptoken.cloud:50052` or an insecure channel for `localhost:50051`
- attach the JWT metadata header `authorization: Bearer <token>`
- request the desired symbol, timeframe, and time range

## Comparing against reference data

When validating against `reference.txt` from the repository, compare these fields:

- `symbol`
- `timeframe`
- `time`
- `open`
- `high`
- `low`
- `close`
- `tick_volume`
- `spread`
- `real_volume`

The stream response may not populate `time_sp` or `time_utc`, so use `time` as the primary join key when matching against the reference file.
