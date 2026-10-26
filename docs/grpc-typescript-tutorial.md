# qData gRPC — Consumindo a Interface via TypeScript

Tutorial focado no consumo dos endpoints de streaming **OHLC** e **Tick** do qData via gRPC em TypeScript.

---

## Índice

1. [Estruturas de Dados](#estruturas-de-dados)
2. [Conectando ao Servidor](#conectando-ao-servidor)
3. [Streaming de Barras OHLC](#streaming-de-barras-ohlc)
4. [Streaming de Ticks](#streaming-de-ticks)
5. [Modo Histórico vs Modo ao Vivo](#modo-histórico-vs-modo-ao-vivo)
6. [Exemplo Completo](#exemplo-completo)
7. [Tratamento de Erros](#tratamento-de-erros)
8. [Dicas e Boas Práticas](#dicas-e-boas-práticas)

---

## Estruturas de Dados

### `TickRequest` — O que você envia

```typescript
interface TickRequest {
  symbol: string;    // Símbolo do instrumento, ex: "EURUSD"
  start_ts: number;  // Timestamp inicial em milissegundos (epoch). 0 = desde o início.
  end_ts: number;    // Timestamp final em milissegundos (epoch). 0 = modo ao vivo (sem fim).
  limit: number;     // Máximo de ticks por lote (lado do servidor, padrão 1000). Geralmente deixe como 0.
}
```

### `Tick` — O que você recebe (por mensagem)

```typescript
interface Tick {
  symbol: string;  // Símbolo do instrumento, ex: "EURUSD"
  ts: number;      // Timestamp em milissegundos desde epoch (ex: 1771948937000)
  price: number;   // Preço da última negociação (double)
  size: number;    // Volume da negociação (inteiro)
}
```

### `OHLCRequest` — O que você envia

```typescript
interface OHLCRequest {
  symbol: string;     // Símbolo do instrumento, ex: "EURUSD"
  timeframe: string;  // Timeframe do candle, ex: "1m", "5m", "1h", "1d"
  start_time: number; // Timestamp inicial em segundos (Unix epoch). 0 = desde o início.
  end_time: number;   // Timestamp final em segundos (Unix epoch). 0 = modo ao vivo (sem fim).
  limit: number;      // Máximo de barras por lote. Geralmente deixe como 0.
}
```

### `OHLCBar` — O que você recebe (por mensagem)

```typescript
interface OHLCBar {
  // Metadados
  id?: number;           // ID da linha no banco de dados (opcional)
  symbol?: string;       // Símbolo do instrumento
  timeframe?: string;    // Timeframe do candle
  source_id?: string;    // Identificador da fonte de dados (opcional)
  ingested_at?: number;  // Quando a barra foi armazenada (Unix timestamp, opcional)

  // Dados do candle
  time: number;          // Horário de abertura do candle em segundos (Unix epoch)
  time_utc?: string;     // Pode vir vazio em algumas respostas; use fallback para `time`
  time_sp?: string;      // Pode vir vazio em algumas respostas

  // Preços OHLC
  open: number;          // Preço de abertura (double)
  high: number;          // Preço máximo (double)
  low: number;           // Preço mínimo (double)
  close: number;         // Preço de fechamento (double)

  // Volume
  tick_volume: number;   // Número de ticks no candle
  spread: number;        // Spread no fechamento
  real_volume: number;   // Volume real negociado
}
```

> **Nota (estado atual do servidor):** os campos `symbol`, `timeframe`, `time`, `open`, `high`, `low`, `close`, `tick_volume`, `real_volume` e `spread` sao os mais confiaveis no stream.
> Campos como `time_utc`, `time_sp`, `source_id`, `ingested_at` e `id` podem nao vir preenchidos em todas as respostas.

> **Importante:** Os timestamps são diferentes entre os serviços!
> - **Ticks**: **milissegundos** desde epoch
> - **OHLC**: **segundos** desde epoch
> - Conversão: `tickTs / 1000 === ohlcTime`

---

## Conectando ao Servidor

> [!IMPORTANT]
> No estado atual do servidor, chamadas gRPC sem metadata `authorization: Bearer <token>` sao rejeitadas com `UNAUTHENTICATED`.
> Antes de consumir os streams, obtenha token em `POST https://uptoken.cloud/api/v1/auth/token`.

```typescript
import * as grpc from "@grpc/grpc-js";
import * as protoLoader from "@grpc/proto-loader";
import path from "path";

// Carregar definições dos protos
const PROTO_DIR = path.join(__dirname, "../protos");

function loadProto(filename: string) {
  const packageDef = protoLoader.loadSync(path.join(PROTO_DIR, filename), {
    keepCase: true,     // mantém nomes em snake_case (start_ts, end_ts, etc.)
    longs: Number,      // representa int64 como Number do JavaScript
    enums: String,
    defaults: true,
    oneofs: true,
  });
  return grpc.loadPackageDefinition(packageDef);
}

const ticksProto: any = loadProto("ticks.proto");
const ohlcProto: any = loadProto("ohlc.proto");

// Criar stubs dos serviços
// Produção: TLS via domínio
const GRPC_ADDRESS = "uptoken.cloud:50052";

const tickClient = new ticksProto.qData.TickService(
  GRPC_ADDRESS,
  grpc.credentials.createSsl()  // TLS — usa CAs do sistema
);

const ohlcClient = new ohlcProto.qData.OHLCService(
  GRPC_ADDRESS,
  grpc.credentials.createSsl()
);

function authMetadata(token: string): grpc.Metadata {
  const md = new grpc.Metadata();
  md.set("authorization", `Bearer ${token}`);
  return md;
}

// Dev local (sem TLS):
// const GRPC_ADDRESS = "localhost:50051";
// grpc.credentials.createInsecure()
```

---

## Streaming de Barras OHLC

```typescript
function streamOHLC(
  symbol: string,
  timeframe: string,
  startTime: number,  // Unix em segundos
  endTime: number,    // Unix em segundos. Passe 0 para modo ao vivo.
  token: string
): void {
  const request = {
    symbol,
    timeframe,
    start_time: startTime,
    end_time: endTime,
  };

  const stream = ohlcClient.StreamOHLC(request, authMetadata(token));

  stream.on("data", (bar: any) => {
    const labelTempo = bar.time_utc || new Date(bar.time * 1000).toISOString();
    console.log(
      `[OHLC] ${bar.symbol} ${bar.timeframe} ${labelTempo}`,
      `A=${bar.open} M=${bar.high} m=${bar.low} F=${bar.close}`,
      `vol=${bar.tick_volume}`
    );
  });

  stream.on("end", () => {
    console.log("[OHLC] Stream encerrado.");
  });

  stream.on("error", (err: grpc.ServiceError) => {
    console.error("[OHLC] Erro:", err.message);
  });
}

// Exemplo: Obter a última hora de candles de 1 minuto para EURUSD
const now = Math.floor(Date.now() / 1000);
streamOHLC("EURUSD", "1m", now - 3600, now, "SEU_JWT_AQUI");
```

---

## Streaming de Ticks

```typescript
function streamTicks(
  symbol: string,
  startTs: number,  // Milissegundos desde epoch
  endTs: number,    // Milissegundos desde epoch. Passe 0 para modo ao vivo.
  token: string
): void {
  const request = {
    symbol,
    start_ts: startTs,
    end_ts: endTs,
  };

  const stream = tickClient.StreamTicks(request, authMetadata(token));

  stream.on("data", (tick: any) => {
    const date = new Date(tick.ts);  // tick.ts é em milissegundos
    console.log(
      `[TICK] ${tick.symbol} ${date.toISOString()}`,
      `preço=${tick.price} tamanho=${tick.size}`
    );
  });

  stream.on("end", () => {
    console.log("[TICK] Stream encerrado.");
  });

  stream.on("error", (err: grpc.ServiceError) => {
    console.error("[TICK] Erro:", err.message);
  });
}

// Exemplo: Obter os últimos 5 minutos de ticks
const nowMs = Date.now();
streamTicks("EURUSD", nowMs - 5 * 60 * 1000, nowMs, "SEU_JWT_AQUI");
```

---

## Modo Histórico vs Modo ao Vivo

O comportamento do servidor muda conforme você define ou não um timestamp de fim:

| Parâmetro | Modo Histórico | Modo ao Vivo |
|---|---|---|
| `end_time` / `end_ts` | Definido com um timestamp específico | Definido como `0` (ou omitido) |
| Comportamento do servidor | Envia todos os dados correspondentes e depois fecha o stream | Envia os dados existentes e continua consultando por novos dados indefinidamente |
| O stream termina? | Sim, automaticamente quando os dados se esgotam | Não — permanece aberto até você cancelar |
| Caso de uso | Backfill, análise, backtesting | Dashboard em tempo real, trading ao vivo |

### Exemplo de Modo ao Vivo

```typescript
// Modo ao vivo: sem timestamp final → stream permanece aberto indefinidamente
function streamTicksAoVivo(symbol: string): grpc.ClientReadableStream<any> {
  const stream = tickClient.StreamTicks({
    symbol,
    start_ts: Date.now(),  // a partir de agora
    end_ts: 0,             // 0 = modo ao vivo
  });

  stream.on("data", (tick: any) => {
    console.log(`[AO VIVO] ${tick.symbol} preço=${tick.price} @ ${new Date(tick.ts).toISOString()}`);
  });

  stream.on("error", (err: grpc.ServiceError) => {
    if (err.code === grpc.status.CANCELLED) {
      console.log("[AO VIVO] Stream cancelado pelo cliente.");
    } else {
      console.error("[AO VIVO] Erro:", err.message);
    }
  });

  return stream; // mantenha a referência para cancelar depois
}

// Iniciar o stream ao vivo
const liveStream = streamTicksAoVivo("EURUSD");

// Cancelar após 60 segundos
setTimeout(() => {
  liveStream.cancel();
}, 60_000);
```

---

## Exemplo Completo

```typescript
import * as grpc from "@grpc/grpc-js";
import * as protoLoader from "@grpc/proto-loader";
import path from "path";

// ──────────────────────────────────────────────
// 1. Carregar protos e criar clientes
// ──────────────────────────────────────────────
const PROTO_DIR = path.join(__dirname, "../protos");
const GRPC_ADDRESS = process.env.GRPC_ADDRESS || "uptoken.cloud:50052";

function loadProto(filename: string) {
  const packageDef = protoLoader.loadSync(path.join(PROTO_DIR, filename), {
    keepCase: true,
    longs: Number,
    enums: String,
    defaults: true,
    oneofs: true,
  });
  return grpc.loadPackageDefinition(packageDef);
}

const ticksProto: any = loadProto("ticks.proto");
const ohlcProto: any = loadProto("ohlc.proto");

// Credenciais TLS (produção) ou insecure (local)
const creds = GRPC_ADDRESS.includes("uptoken.cloud")
  ? grpc.credentials.createSsl()
  : grpc.credentials.createInsecure();

const tickClient = new ticksProto.qData.TickService(GRPC_ADDRESS, creds);

const ohlcClient = new ohlcProto.qData.OHLCService(GRPC_ADDRESS, creds);

// ──────────────────────────────────────────────
// 2. Auxiliar: coletar stream em um array
// ──────────────────────────────────────────────
function coletarStream<T>(stream: grpc.ClientReadableStream<T>): Promise<T[]> {
  return new Promise((resolve, reject) => {
    const resultados: T[] = [];
    stream.on("data", (item: T) => resultados.push(item));
    stream.on("end", () => resolve(resultados));
    stream.on("error", (err: grpc.ServiceError) => reject(err));
  });
}

// ──────────────────────────────────────────────
// 3. Buscar barras OHLC históricas
// ──────────────────────────────────────────────
async function buscarOHLCHistorico(
  symbol: string,
  timeframe: string,
  startTime: number,
  endTime: number
) {
  const stream = ohlcClient.StreamOHLC({
    symbol,
    timeframe,
    start_time: startTime,
    end_time: endTime,
  });

  const barras = await coletarStream(stream);

  console.log(`\nRecebidas ${barras.length} barras OHLC para ${symbol} (${timeframe}):`);
  console.log("─".repeat(80));

  for (const bar of barras) {
    const labelTempo = bar.time_utc || new Date(bar.time * 1000).toISOString();
    console.log(
      `  ${labelTempo}  A=${bar.open.toFixed(4)}  M=${bar.high.toFixed(4)}` +
      `  m=${bar.low.toFixed(4)}  F=${bar.close.toFixed(4)}  vol=${bar.tick_volume}`
    );
  }

  return barras;
}

// ──────────────────────────────────────────────
// 4. Buscar ticks históricos
// ──────────────────────────────────────────────
async function buscarTicksHistoricos(
  symbol: string,
  startTs: number,
  endTs: number
) {
  const stream = tickClient.StreamTicks({
    symbol,
    start_ts: startTs,
    end_ts: endTs,
  });

  const ticks = await coletarStream(stream);

  console.log(`\nRecebidos ${ticks.length} ticks para ${symbol}:`);
  console.log("─".repeat(80));

  for (const tick of ticks) {
    const data = new Date(tick.ts).toISOString();
    console.log(
      `  ${data}  preço=${tick.price.toFixed(5)}  tamanho=${tick.size}`
    );
  }

  return ticks;
}

// ──────────────────────────────────────────────
// 5. Executar
// ──────────────────────────────────────────────
async function main() {
  const agoraSeg = Math.floor(Date.now() / 1000);
  const agoraMs = Date.now();

  try {
    // OHLC histórico: últimas 24 horas de candles de 1 minuto
    await buscarOHLCHistorico("EURUSD", "1m", agoraSeg - 86400, agoraSeg);

    // Ticks históricos: última hora
    await buscarTicksHistoricos("EURUSD", agoraMs - 3600_000, agoraMs);
  } catch (err) {
    console.error("Falha:", err);
  }
}

main();
```

---

## Tratamento de Erros

Erros gRPC comuns que você pode encontrar:

| Código de Status | Significado | Causa Provável |
|---|---|---|
| `UNAVAILABLE` | Servidor inacessível | Servidor está fora do ar ou endereço errado |
| `DEADLINE_EXCEEDED` | Requisição expirou | O servidor demorou demais; ou o stream ao vivo excedeu o prazo do cliente |
| `CANCELLED` | Stream cancelado | Você chamou `stream.cancel()` — isso é normal para streams ao vivo |
| `UNKNOWN` | Exceção no servidor | Geralmente um erro de banco de dados; verifique os logs do servidor |
| `UNIMPLEMENTED` | Método não encontrado | Incompatibilidade de proto — regenere seus tipos |

### Tratando erros de forma elegante

```typescript
stream.on("error", (err: grpc.ServiceError) => {
  switch (err.code) {
    case grpc.status.UNAVAILABLE:
      console.error("Servidor inacessível. Tentando novamente em 5s...");
      setTimeout(() => reconectar(), 5000);
      break;
    case grpc.status.CANCELLED:
      // Normal — o cliente cancelou o stream
      break;
    case grpc.status.DEADLINE_EXCEEDED:
      console.warn("Prazo excedido — considere aumentar o timeout.");
      break;
    default:
      console.error(`Erro gRPC [${err.code}]: ${err.message}`);
  }
});
```

### Definindo um prazo (timeout)

```typescript
const prazo = new Date(Date.now() + 30_000); // 30 segundos

const stream = ohlcClient.StreamOHLC(request, { deadline: prazo });
```

> **Atenção**: Não defina um prazo em streams no modo ao vivo — eles são feitos para rodar indefinidamente.

---

## Dicas e Boas Práticas

1. **Os timestamps são diferentes entre os serviços**
   - Ticks: **milissegundos** desde epoch
   - OHLC: **segundos** desde epoch
   - Conversão: `tickTs / 1000 === ohlcTime`

2. **Use `longs: Number`** nas opções do proto-loader. Sem isso, campos `int64` chegam como objetos `Long` em vez de números simples do JavaScript.

3. **Streams ao vivo precisam de cancelamento**. Sempre mantenha uma referência ao stream e chame `.cancel()` quando terminar para evitar vazamento de recursos.

4. **Lógica de reconexão**. Os streams gRPC não reconectam automaticamente. Se o servidor reiniciar, você receberá um erro `UNAVAILABLE` — implemente retry com backoff exponencial.

5. **Contrapressão (backpressure)**. Se o seu processamento for lento, os dados podem se acumular na memória. Use `stream.pause()` e `stream.resume()` se necessário:
   ```typescript
   stream.on("data", async (tick) => {
     stream.pause();
     await processarTickLentamente(tick);
     stream.resume();
   });
   ```

6. **`keepCase: true`** no proto-loader preserva os nomes em snake_case (`start_ts`, `time_utc`). Sem isso, eles são convertidos para camelCase (`startTs`, `timeUtc`).

7. **Variável de ambiente para o endereço**. O exemplo acima lê `GRPC_ADDRESS` do ambiente, com padrão `uptoken.cloud:50052`:
   ```bash
   # Produção (TLS)
   GRPC_ADDRESS=uptoken.cloud:50052 npx ts-node src/index.ts

   # Dev local (sem TLS)
   GRPC_ADDRESS=localhost:50051 npx ts-node src/index.ts
   ```

