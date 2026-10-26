# qData

Real-time and historical market data platform with a **REST API** (FastAPI) and **gRPC streaming** interface for OHLC bars and ticks. To access this API, send an e-mail to **[`luismaiasombra@gmail.com`](mailto:luismaiasombra@gmail.com)**

---

## Connecting to qData

### Endpoints

| Service | URL | Protocol |
|---|---|---|
| **REST API** | `https://uptoken.cloud/` | HTTPS (TLS 1.2+) |
| **REST API (dashboard)** | `https://http.uptoken.cloud/` | HTTPS (CORS p/ `stats.uptoken.cloud`) |
| **gRPC (nativo)** | `uptoken.cloud:50052` | gRPC over TLS |
| **gRPC-Web** | `https://grpc.uptoken.cloud/` | gRPC-Web (browser) |
| **gRPC-Web (alt)** | `https://uptoken.cloud:8443/` | gRPC-Web (browser) |
| **API Docs** | `https://uptoken.cloud/docs` | Swagger UI |

> [!NOTE]
> HTTP requests to port 80 are automatically redirected to HTTPS.

### Portas e acesso (migração para o Traefik central — set/2026)

**Nenhuma porta do stack é mais publicada no host.** As antigas portas
`8000` (REST), `50051` (gRPC) e `8080` (Envoy) foram **fechadas no servidor** —
elas contornavam o firewall e expunham a API sem TLS. Todo o acesso externo
passa pelo **Traefik central** (dono de 80/443/8443/50052, TLS wildcard
`*.uptoken.cloud` via DNS-01).

| Porta interna (rede Docker) | Serviço | Acesso externo |
|---|---|---|
| `app:8000` | FastAPI (REST) | `https://uptoken.cloud`, `https://http.uptoken.cloud` |
| `app:50051` | gRPC (h2c) | `uptoken.cloud:50052` (TLS terminado no Traefik) |
| `envoy:8080` | Envoy (bridge gRPC-Web) | `https://grpc.uptoken.cloud` e `https://uptoken.cloud:8443` |

Detalhes técnicos:

- Os containers `qData-app` e `qData-envoy` participam da rede externa
  `uptoken_network` com os **aliases `app` e `envoy`** (declarado no
  `docker-compose.yml` — não usar `docker network connect` manual: o attach se
  perde a cada recreate do compose).
- O Envoy alcança o gRPC do app via **`app:50051`** (DNS do Docker), não via
  `host.docker.internal` — que dependia da porta publicada no host.
- `http.uptoken.cloud` serve a **mesma API REST** do `uptoken.cloud`; existe
  porque o dashboard (FinHubLTI) troca o host no modo "HTTPS (REST)".
- CORS: configure a variável **`CORS_LIST`** (array JSON) — ver `.env.example`.

---

### REST API — Quick Start

#### 1. Authenticate

Users are authenticated against the `users` table in Supabase. You'll need a username and password that have been hashed and stored there.

```bash
curl -X POST https://uptoken.cloud/api/v1/auth/token \
  -H "Content-Type: application/json" \
  -d '{"username": "demo_user", "password": "your_password"}'
```

Response:
```json
{
  "access_token": "eyJ...",
  "token_type": "bearer"
}
```

#### 2. Make authenticated requests

```bash
curl https://uptoken.cloud/api/v1/auth/me \
  -H "Authorization: Bearer <access_token>"
```

#### 3. Fetch Historical Data (OHLC)

You can fetch historical OHLC bar data via standard GET requests:

```bash
curl "https://uptoken.cloud/api/v1/distribution/ohlc?symbol=WINJ26&timeframe=M1&limit=5" \
  -H "Authorization: Bearer <access_token>"
```

> [!TIP]
> For a full list of query parameters and detailed schemas, check the interactive Swagger UI at [https://uptoken.cloud/docs](https://uptoken.cloud/docs) or the [HTTP Endpoint Documentation](docs/http_endpoint.md).

---

### gRPC — Quick Start

qData exposes two gRPC streaming services:

| Service | Method | Description |
|---|---|---|
| `OHLCService` | `StreamOHLC` | Server-streaming of OHLC candlestick bars |
| `TickService` | `StreamTicks` | Server-streaming of individual ticks |

Os arquivos Proto ficam no diretório `protos/` do projeto de origem.

#### Connecting with TLS (remote)

```typescript
import * as grpc from "@grpc/grpc-js";

const client = new ticksProto.qData.TickService(
  "uptoken.cloud:50052",
  grpc.credentials.createSsl()  // TLS — no cert files needed (uses system CAs)
);
```

#### Connecting without TLS (localhost)

```typescript
const client = new ticksProto.qData.TickService(
  "localhost:50051",
  grpc.credentials.createInsecure()
);
```

#### Example: Stream live ticks

```typescript
const stream = tickClient.StreamTicks({
  symbol: "EURUSD",
  start_ts: Date.now(),  // from now
  end_ts: 0,             // 0 = live mode (stream stays open)
});

stream.on("data", (tick) => {
  console.log(`${tick.symbol} price=${tick.price} @ ${new Date(tick.ts).toISOString()}`);
});
```

> [!TIP]
> Set `end_ts` / `end_time` to `0` for **live streaming** (stream stays open indefinitely).
> Set it to a specific timestamp for **historical mode** (stream closes when data is exhausted).

---

## Documentação

### API e consumo

| Documento | Descrição |
|---|---|
| [Endpoint HTTP para distribuição de dados](docs/http_endpoint.md) | Endpoint REST para dados históricos OHLC |
| [Endpoint HTTP para distribuição de dados (PT)](docs/http_endpoint_pt.md) | Versão em português da documentação do endpoint REST |
| [Uso da API via HTTPS (REST)](docs/https_rest_usage.md) | Autenticação, distribuição OHLC e Swagger |
| [Tutorial gRPC com TypeScript](docs/grpc-typescript-tutorial.md) | Consumo dos streams OHLC e Tick via TypeScript |
| [Uso de gRPC-Web](docs/grpc-web-usage.md) | Integração de clientes web com gRPC-Web e Envoy |
| [Consultas de teste gRPC](docs/grpc-test-queries.md) | Consultas e validação com `grpcurl` |
| [Autenticação gRPC](docs/grpc-auth-tutorial.md) | Fluxo de autenticação JWT para clientes gRPC |
| [Relatório do endpoint de autenticação](docs/auth-endpoint-report.md) | Contrato e status do endpoint de autenticação |

### Operação e infraestrutura

| Documento | Descrição |
|---|---|
| [Referência de portas e infraestrutura](ports.md) | Topologia, portas, rotas e fluxos de tráfego |
| [Infraestrutura de VPS](docs/infraestrutura-vps.md) | Orientações para provisionamento e operação |
| [Manutenção Docker](docs/docker-maintenance.md) | Renovação de certificados e limpeza |
| [Scripts operacionais](docs/scripts.md) | Scripts auxiliares do serviço |
| [Pipeline de testes](docs/github-actions-tutorial.md) | Execução local e CI com GitHub Actions |
| [Banco efêmero de testes](docs/banco-efemero-testes.md) | Configuração do banco usado nos testes |

> Esta pasta reproduz a documentação do `qData_service`. Referências a código,
> scripts e arquivos de configuração apontam para o projeto de origem quando
> esses arquivos não fazem parte deste conjunto documental.

---

## Architecture Overview

The system runs behind the **Traefik central proxy** (TLS wildcard + WAF for the
vault stack). The old `nginxmainproxy` / `qData-nginx` layers were retired.

```mermaid
graph TD
    Internet((Internet)) --> Traefik[Traefik central<br/>80 / 443 / 8443 / 50052<br/>TLS wildcard *.uptoken.cloud]

    subgraph qData_Stack [qData Stack - rede uptoken_network]
        Traefik -- "443 → app:8000 (REST)" --> FastAPI[FastAPI App]
        Traefik -- "50052 → h2c app:50051 (gRPC nativo)" --> gRPC[gRPC Server]
        Traefik -- "443/8443 → envoy:8080 (gRPC-Web)" --> Envoy[Envoy Bridge]
        Envoy -- "app:50051 (h2c)" --> gRPC
    end

    FastAPI --> DB[(PostgreSQL)]
    gRPC --> DB
```

1. **Traefik central**: TLS termination (wildcard Let's Encrypt via DNS-01),
   routing by hostname and entrypoint. No inner nginx, no duplicated TLS hops.
2. **App Layer**: FastAPI (REST, `app:8000`) and gRPC (streaming, `app:50051`,
   HTTP/2 cleartext), reachable **only inside the Docker network**.
3. **Envoy bridge**: translates browser gRPC-Web → native gRPC (h2c).

---

## Logging

O nível de log é controlado **por variável de ambiente**, sem alterar o
código-fonte. A configuração central fica em
`src/qData/core/logging.py` e é aplicada na importação de `qData.main`, valendo
para qualquer entrypoint (`uvicorn qData.main:app` no container, `run_server.py`
no dev, `pytest`).

| Variável | Valores | Padrão | Efeito |
|---|---|---|---|
| `LOG_LEVEL` | `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL` | `INFO` | Nível do logger raiz. Valor inválido cai para `INFO`. |
| `LOG_FORMAT` | `text`, `json` | `text` | `json` emite uma linha JSON por registro (para agregadores de log). |
| `LOG_ACCESS` | `true`, `false` | `true` | `false` silencia o access log do uvicorn (útil com healthcheck a cada 30s). |

Recomendado: `LOG_LEVEL=INFO` em desenvolvimento (mostra os rastros
`[STREAM_BACKEND]` do streaming gRPC e `[HTTP]` da distribuição OHLC) e
`LOG_LEVEL=WARNING` em produção, se quiser reduzir volume.

```bash
# Local
LOG_LEVEL=DEBUG ./run_server.sh

# Docker (via .env, lido pelo compose em env_file)
echo "LOG_LEVEL=INFO" >> .env && docker compose up -d
docker compose logs -f app | grep STREAM_BACKEND
```

> [!NOTE]
> **Por que os logs `INFO` não apareciam antes:** o entrypoint em produção é o
> `uvicorn`, que configura apenas os próprios loggers (`uvicorn.*`) e não
> instala handler no logger raiz. Sem handler na raiz, os registros de
> `qData.*` caíam no `logging.lastResort`, cujo nível é `WARNING` — e eram
> descartados em silêncio. O `setup_logging()` instala esse handler.

### Dados sensíveis

Um filtro de redação (`RedactingFilter`) mascara automaticamente valores de
chaves como `password`, `token`, `secret`, `api_key` e `authorization`, além de
cabeçalhos `Bearer`/`Basic`, substituindo-os por `***REDACTED***` antes de a
mensagem chegar à saída. Ainda assim, **não logue segredos deliberadamente** —
o filtro é uma rede de segurança, não uma autorização.

---

## Self-Hosting

### Prerequisites

- Docker & Docker Compose
- A domain with an A record pointing to your server

### Setup

1. Configure `.env` (see `.env.example`):
   ```env
   DOMAIN=uptoken.cloud
   DATABASE_URL=postgresql://user:pass@host:5432/db
   SUPABASE_URL=https://your-project.supabase.co
   SUPABASE_REST_KEY=your-service-role-key
   REST_JWT_SECRET=your-rest-secret
   GRPC_JWT_SECRET=your-grpc-secret
   CORS_LIST=["https://your-frontend.example.com"]
   LOG_LEVEL=INFO
   ```

   > Detalhes das variáveis de log (`LOG_LEVEL`, `LOG_FORMAT`, `LOG_ACCESS`)
   > na seção [Logging](#logging).

2. Start services:
   ```bash
   docker compose up -d
   ```

   > [!IMPORTANT]
   > Desde a migração para o Traefik central, o compose **não publica portas**
   > e conecta os containers à rede externa `uptoken_network` (aliases `app` e
   > `envoy`). O acesso externo é feito pelo Traefik (ver tabela de Endpoints).
   >
   > **Ambiente de dev/local** sem Traefik: publique portas só temporariamente,
   > criando um override (não commitar):
   > ```yaml
   > # docker-compose.override.yml
   > services:
   >   app:
   >     ports: ["8000:8000", "50051:50051"]
   > ```
   > Configure também `CORS_LIST` no `.env` com as origens do seu frontend.

3. TLS is terminated by the Traefik central proxy (wildcard certificate via
   DNS-01). The legacy certbot/nginx blocks are no longer used.

---

## License

Private — all rights reserved.
