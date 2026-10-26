# Endpoint HTTP para Distribuição de Dados

O `qData_service` disponibiliza um endpoint HTTP (REST) para distribuição de dados juntamente com os endpoints gRPC existentes. Isso permite que os clientes busquem dados históricos de mercado (OHLC - Abertura, Máxima, Mínima, Fechamento) por meio de requisições JSON RESTful padrão.

## Endereço

O endpoint está acessível via HTTPS em:
`https://uptoken.cloud/api/v1/distribution/ohlc`

## `GET /api/v1/distribution/ohlc`

Retorna dados históricos de barras OHLC.

**Parâmetros de Query (Query Parameters):**
- `symbol` (string, opcional): O símbolo do ativo (ex: `WINJ26`).
- `timeframe` (string, opcional): O tempo gráfico das barras (ex: `M1`, `M5`, `H1`).
- `start_time` (int, opcional): Timestamp Unix de início em segundos (inclusivo).
- `end_time` (int, opcional): Timestamp Unix de término em segundos (inclusivo).
- `limit` (int, padrão `1000`): Número máximo de registros a serem retornados.
- `offset` (int, padrão `0`): Deslocamento para paginação.
- `order_desc` (boolean, padrão `true`): Ordenar por data de forma decrescente.

**Formato da Resposta:**
Retorna um array JSON de objetos `OHLCBar`. Cada objeto contém os seguintes campos:

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | integer | (Opcional) ID do registro no banco de dados |
| `symbol` | string | (Opcional) Símbolo do ativo |
| `timeframe` | string | (Opcional) Tempo gráfico da barra (ex: `M1`, `M5`, `H1`) |
| `source_id` | string | (Opcional) Identificador da fonte de dados |
| `ingested_at` | string | (Opcional) Timestamp de ingestão |
| `time` | integer | Timestamp Unix em segundos |
| `time_utc` | string | Timestamp em formato ISO 8601 (UTC) |
| `time_sp` | string | Timestamp em formato ISO 8601 (São Paulo) |
| `open` | number | Preço de Abertura |
| `high` | number | Preço Máximo |
| `low` | number | Preço Mínimo |
| `close` | number | Preço de Fechamento |
| `tick_volume` | integer | Volume de ticks |
| `spread` | integer | Spread |
| `real_volume` | integer | Volume real |

**Exemplo de Resposta:**
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

## Integração

O endpoint está registrado na aplicação FastAPI principal (`src/qData/main.py`) sob o prefixo `/api/v1/distribution`. Ele utiliza as mesmas funções de acesso ao banco de dados (`fetch_ohlc`) que o serviço gRPC, garantindo consistência no acesso aos dados em ambos os protocolos.

