# qData gRPC-Web Usage Guide

## Overview

qData provides a native gRPC endpoint on port `50051` (internal) and `50052` (external TLS). However, because web browsers cannot perform raw HTTP/2 framing and binary streaming natively, we cannot use typical gRPC clients directly from frontend applications (like React, Vue, or Vanilla JS).

To solve this, we have exposed a dedicated **gRPC-Web proxy**.
The gRPC-Web endpoint is publicly accessible via Nginx at **`https://uptoken.cloud:8443`**.

In DEV_ENV5, requests traverse two proxy layers before reaching the app.

## Architecture

The underlying structure converts browser-compatible HTTP/1.1 or HTTP/2 base64 encoded strings into pure gRPC binary frames.

`Browser (gRPC-Web)` ➔ `nginxmainproxy:8443 (TLS)` ➔ `qData-nginx:8443` ➔ `Envoy:8080 (gRPC-Web translation + CORS)` ➔ `qData-app:50051`

- **Nginx** handles the standard SSL termination using Let's Encrypt certificates.
- **Envoy Proxy**, functioning as a sidecar container, translates the `application/grpc-web-text` framing format into `application/grpc` frames. It also completely manages Cross-Origin Resource Sharing (CORS) rules.

> **Endpoint contract**
> - Browser clients: `https://uptoken.cloud:8443`
> - Native gRPC clients: `uptoken.cloud:50052`
> - Local debug only: `localhost:50051` (insecure)

## Generating Web Clients

To communicate with the gRPC-Web proxy, you need to compile your `.proto` files into JavaScript or TypeScript using the special `grpc-web` plugin.

You can use standard `protoc` alongside the official `protoc-gen-grpc-web` plugin:

```bash
# Generate regular JS messages
protoc -I=protos protos/ohlc.proto protos/ticks.proto \
  --js_out=import_style=commonjs,binary:./generated

# Generate the gRPC-Web service client
protoc -I=protos protos/ohlc.proto protos/ticks.proto \
  --grpc-web_out=import_style=typescript,mode=grpcwebtext:./generated
```

This will produce files typically ending in `_pb.js` (for your message classes) and `_pb.d.ts` / `_pb_service.js` (for the actual client connections).

## Connecting from Web Clients

To use the client in your web application, you need to import the auto-generated code and the `grpc-web` npm package:

```bash
npm install grpc-web google-protobuf
```

### Setup and Authentication

When connecting to the `8443` endpoint, you must instantiate the client with the Nginx URL, and importantly, ensure you are passing the JWT token metadata precisely as you do with native gRPC. 

```typescript
import { QuantwinClient } from './qData_grpc_web_pb';
import { OHLCRequest } from './qData_pb';

// 1. Point the client to the dedicated gRPC-Web TLS port
const client = new QuantwinClient('https://uptoken.cloud:8443', null, null);

// 2. Add your JWT Authorization payload
const metadata = {
  'authorization': 'Bearer YOUR_JWT_TOKEN_HERE'
};

const request = new OHLCRequest();
request.setSymbol('EURUSD');
request.setTimeframe('M1');

// 3. Initiate a stream
const stream = client.streamOHLC(request, metadata);

stream.on('data', (response) => {
  console.log('Received OHLC:', response.toObject());
});

stream.on('error', (err) => {
  console.error('Stream Error:', err.code, err.message);
});

stream.on('end', () => {
  console.log('Stream ended natively by server.');
});
```

### Authentication prerequisite

Obtain your JWT before opening a gRPC stream:

```bash
curl -X POST https://uptoken.cloud/api/v1/auth/token \
  -H "Content-Type: application/json" \
  -d '{"username":"YOUR_USER","password":"YOUR_PASSWORD"}'
```

## CORS & Headers Supported

The Envoy proxy responds to browser `OPTIONS` preflight checks for an allowlist of origins configured in `envoy.yaml` (for example: `localhost`, `127.0.0.1`, `uptoken.cloud`, and `stats.uptoken.cloud`). It safely exposes the specific HTTP trailers utilized by gRPC to track stream health.

**Supported Headers:**
- `authorization` (essential for our JWT implementation)
- `x-grpc-web`
- `grpc-timeout`

**Exposed Headers:**
- `grpc-status`
- `grpc-message`

> **Important**: Do not try connecting a standard native gRPC client (e.g., Python `grpcio` or Go `grpc`) to port `8443`. That port is specifically built to decipher gRPC-Web headers. Native clients must continue using port `50052`.

