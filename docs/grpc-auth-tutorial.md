# Autenticação gRPC — Guia de Implementação

Este guia explica como funciona a autenticacao JWT no qData no estado atual do projeto, com usuarios armazenados no Supabase. Para ter acesso envie um e-mail para **[`luismaiasombra@gmail.com`](mailto:luismaiasombra@gmail.com)**


## Como Funciona

O fluxo de autenticação usa **JWT (JSON Web Token)** compartilhado entre HTTP e gRPC:

```
1. Cliente obtém um token JWT via HTTPS (REST):
   POST https://uptoken.cloud/api/v1/auth/token  →  { "access_token": "eyJ..." }

2. Cliente envia o token como metadata em cada chamada gRPC:
   metadata: { "authorization": "Bearer eyJ..." }

3. Servidor gRPC valida o token com REST_JWT_SECRET via interceptor
```

> **Importante (estado atual)**: o interceptor gRPC valida o mesmo token JWT emitido pelo endpoint REST, usando `REST_JWT_SECRET`.
> `GRPC_JWT_SECRET` permanece no `.env`, mas nao e usado pelo interceptor atual.

### Endpoints por tipo de cliente

| Tipo de cliente | Endpoint | Observacao |
|---|---|---|
| REST auth/login | `https://uptoken.cloud/api/v1/auth/token` | Emite o JWT |
| gRPC nativo (TLS) | `uptoken.cloud:50052` | Node/Python/Go em producao |
| gRPC-Web (browser) | `https://uptoken.cloud:8443` | React/Angular/Vue no navegador |
| Local debug gRPC | `localhost:50051` | Inseguro, somente desenvolvimento |

---

## Configuração de Credenciais

### 1. Defina as variáveis no `.env`

```env
# Supabase (armazenamento de usuários)
SUPABASE_URL=https://bpxfnvnuxpjxnxphzlqn.supabase.co
SUPABASE_REST_KEY=sua-service-role-key

# Segredos JWT configurados no ambiente
REST_JWT_SECRET=chave-secreta-para-rest
GRPC_JWT_SECRET=chave-secreta-para-grpc

# Tempo de expiração do token em minutos (padrão: 60)
ACCESS_TOKEN_EXPIRE_MINUTES=60
```

> Os usuarios sao armazenados na tabela `users` do Supabase com colunas `username`, `hashed_password`, `service_type` e `is_active`.
> No fluxo atual, o login em `/api/v1/auth/token` e o interceptor gRPC consultam `service_type = 'rest'`.

### 2. Gerando o hash bcrypt da sua senha

Use este comando para gerar o hash da senha que você obteve do gerenciador de senhas:

```bash
poetry run python3 -c "
from passlib.context import CryptContext
pwd = CryptContext(schemes=['bcrypt'], deprecated='auto')
senha = input('Digite a senha: ')
print(f'Hash: {pwd.hash(senha)}')
"
```

> Copie o hash gerado (comeca com `$2b$`) e salve na coluna `hashed_password` da tabela `users` no Supabase.

---

## Implementação no Servidor

### 1. Interceptor de Autenticação gRPC

Crie o arquivo `src/qData/api/grpc/interceptor.py`:

```python
"""
gRPC server interceptor that validates JWT tokens from call metadata.

Expects the client to send metadata:
    ("authorization", "Bearer <token>")

Reuses the REST_JWT_SECRET and algorithm from the HTTP auth module.
"""
import logging
from typing import Optional

import grpc
from jose import JWTError, jwt

from qData.core.settings import REST_JWT_SECRET, JWT_ALGORITHM
from qData.auth.user_store import get_user

logger = logging.getLogger(__name__)


def _extract_token(metadata) -> Optional[str]:
    """Extract Bearer token from gRPC metadata."""
    for key, value in metadata:
        if key == "authorization":
            parts = value.split()
            if len(parts) == 2 and parts[0].lower() == "bearer":
                return parts[1]
    return None


def _validate_token(token: str) -> dict:
    """Validate JWT token and return the payload. Raises on failure."""
    payload = jwt.decode(token, REST_JWT_SECRET, algorithms=[JWT_ALGORITHM])
    sub = payload.get("sub")
    if not sub:
        raise ValueError("Token missing 'sub' claim")
    return payload


class AuthInterceptor(grpc.aio.ServerInterceptor):
    """
    Async gRPC server interceptor that enforces JWT authentication.

    All RPC methods require a valid Bearer token in the 'authorization'
    metadata key. Unauthenticated calls are rejected with UNAUTHENTICATED.
    """

    async def intercept_service(self, continuation, handler_call_details):
        # Extract metadata
        metadata = handler_call_details.invocation_metadata
        token = _extract_token(metadata)

        if not token:
            logger.warning("gRPC call without authorization token: %s", handler_call_details.method)
            return self._unauthenticated("Token de autenticação ausente. Envie metadata: authorization = 'Bearer <token>'")

        try:
            _validate_token(token)
        except JWTError as e:
            logger.warning("gRPC invalid JWT: %s", e)
            return self._unauthenticated(f"Token inválido: {e}")
        except Exception as e:
            logger.error("gRPC auth error: %s", e)
            return self._unauthenticated("Erro na validação do token")

        # Token is valid — proceed to the actual handler
        return await continuation(handler_call_details)

    @staticmethod
    def _unauthenticated(detail: str):
        """Return an RPC handler that immediately aborts with UNAUTHENTICATED."""

        async def _abort(request, context):
            await context.abort(grpc.StatusCode.UNAUTHENTICATED, detail)

        # Works for all RPC types (unary, server-stream, etc.)
        return grpc.unary_unary_rpc_method_handler(_abort)
```

### 2. Registrar o Interceptor no Servidor

Atualize o `start_grpc()` em `src/qData/main.py`:

```python
async def start_grpc():
    """Start gRPC server."""
    global grpc_server
    try:
        import sys, os
        base_path = os.path.dirname(os.path.abspath(__file__))
        gen_path = os.path.join(base_path, "gen")
        if gen_path not in sys.path:
            sys.path.append(gen_path)

        from qData.gen import ohlc_pb2_grpc, ticks_pb2_grpc
        from qData.api.grpc.services import OHLCService, TickService
        from qData.api.grpc.interceptor import AuthInterceptor  # ← NOVO

        grpc_server = grpc.aio.server(
            interceptors=[AuthInterceptor()]  # ← NOVO
        )

        ohlc_pb2_grpc.add_OHLCServiceServicer_to_server(OHLCService(), grpc_server)
        ticks_pb2_grpc.add_TickServiceServicer_to_server(TickService(), grpc_server)

        listen_addr = '[::]:50051'
        grpc_server.add_insecure_port(listen_addr)
        logging.info(f"Starting gRPC server on {listen_addr}")
        await grpc_server.start()
    except Exception as e:
        logging.error(f"Failed to start gRPC server: {e}", exc_info=True)
```

---

## Consumindo com Autenticação (TypeScript)

### Passo 1: Obter o token JWT via HTTP

```typescript
async function obterToken(username: string, password: string): Promise<string> {
  const resp = await fetch("https://uptoken.cloud/api/v1/auth/token", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ username, password }),
  });

  if (!resp.ok) {
    throw new Error(`Falha na autenticação: ${resp.status}`);
  }

  const data = await resp.json();
  return data.access_token;
}
```

### Passo 2: Enviar o token como metadata no gRPC

```typescript
import * as grpc from "@grpc/grpc-js";

// Criar metadata com o token
function criarMetadataAuth(token: string): grpc.Metadata {
  const metadata = new grpc.Metadata();
  metadata.set("authorization", `Bearer ${token}`);
  return metadata;
}

// Usar nas chamadas gRPC
async function streamComAuth(token: string) {
  const metadata = criarMetadataAuth(token);

  // OHLC — passa metadata como segundo argumento
  const ohlcStream = ohlcClient.StreamOHLC(
    { symbol: "EURUSD", timeframe: "1m", start_time: 0, end_time: Math.floor(Date.now() / 1000) },
    metadata  // ← autenticação aqui
  );

  ohlcStream.on("data", (bar: any) => {
    console.log(`[OHLC] ${bar.symbol} O=${bar.open} C=${bar.close}`);
  });

  // Ticks — mesma coisa
  const tickStream = tickClient.StreamTicks(
    { symbol: "EURUSD", start_ts: Date.now() - 3600000, end_ts: Date.now() },
    metadata  // ← autenticação aqui
  );

  tickStream.on("data", (tick: any) => {
    console.log(`[TICK] ${tick.symbol} price=${tick.price}`);
  });
}
```

### Exemplo Completo com Autenticação

```typescript
import * as grpc from "@grpc/grpc-js";
import * as protoLoader from "@grpc/proto-loader";
import path from "path";

// Carregar protos
const PROTO_DIR = path.join(__dirname, "../protos");
const GRPC_ADDRESS = "uptoken.cloud:50052";
const API_URL = "https://uptoken.cloud";

function loadProto(filename: string) {
  return grpc.loadPackageDefinition(
    protoLoader.loadSync(path.join(PROTO_DIR, filename), {
      keepCase: true, longs: Number, enums: String, defaults: true, oneofs: true,
    })
  );
}

const ohlcProto: any = loadProto("ohlc.proto");
const ticksProto: any = loadProto("ticks.proto");

const ohlcClient = new ohlcProto.qData.OHLCService(GRPC_ADDRESS, grpc.credentials.createSsl());
const tickClient = new ticksProto.qData.TickService(GRPC_ADDRESS, grpc.credentials.createSsl());

// ──────────────────────────────────────────────
// Autenticação
// ──────────────────────────────────────────────

async function login(username: string, password: string): Promise<string> {
  const resp = await fetch(`${API_URL}/api/v1/auth/token`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ username, password }),
  });
  if (!resp.ok) throw new Error(`Login falhou: ${resp.status}`);
  const { access_token } = await resp.json();
  return access_token;
}

function authMetadata(token: string): grpc.Metadata {
  const md = new grpc.Metadata();
  md.set("authorization", `Bearer ${token}`);
  return md;
}

// ──────────────────────────────────────────────
// Consumo autenticado
// ──────────────────────────────────────────────

function coletarStream<T>(stream: grpc.ClientReadableStream<T>): Promise<T[]> {
  return new Promise((resolve, reject) => {
    const items: T[] = [];
    stream.on("data", (item: T) => items.push(item));
    stream.on("end", () => resolve(items));
    stream.on("error", reject);
  });
}

async function main() {
  // 1. Login — use suas credenciais do gerenciador de senhas
  const USERNAME = process.env.API_USERNAME || "seu-usuario";
  const PASSWORD = process.env.API_PASSWORD || "sua-senha";

  console.log(`Autenticando como ${USERNAME}...`);
  const token = await login(USERNAME, PASSWORD);
  console.log("Token obtido com sucesso.");

  const metadata = authMetadata(token);
  const agora = Math.floor(Date.now() / 1000);

  // 2. Buscar OHLC com autenticação
  const ohlcStream = ohlcClient.StreamOHLC(
    { symbol: "EURUSD", timeframe: "1m", start_time: agora - 86400, end_time: agora },
    metadata
  );
  const barras = await coletarStream(ohlcStream);
  console.log(`Recebidas ${barras.length} barras OHLC`);

  // 3. Buscar Ticks com autenticação
  const tickStream = tickClient.StreamTicks(
    { symbol: "EURUSD", start_ts: Date.now() - 3600_000, end_ts: Date.now() },
    metadata
  );
  const ticks = await coletarStream(tickStream);
  console.log(`Recebidos ${ticks.length} ticks`);
}

main().catch(console.error);
```

Executar com credenciais via variáveis de ambiente:

```bash
API_USERNAME=seu-usuario API_PASSWORD=sua-senha npx ts-node src/index.ts
```

> **Nota (estado atual)**: Para autenticar via REST e gRPC, o usuario precisa estar com `service_type = 'rest'` na tabela Supabase.

---

## Erros de Autenticação

| Código gRPC | Quando acontece | O que fazer |
|---|---|---|
| `UNAUTHENTICATED` | Token ausente ou inválido | Verifique se está enviando metadata `authorization` com `Bearer <token>` |
| `UNAUTHENTICATED` + "Token expirado" | Token JWT expirou | Obtenha um novo token via `POST /api/v1/auth/token` |
| HTTP `400` no login | Usuário ou senha incorretos | Verifique as credenciais |

### Renovação automática de token

```typescript
class AuthClient {
  private token: string = "";
  private expiresAt: number = 0;

  constructor(
    private username: string,
    private password: string,
    private apiUrl: string
  ) {}

  async getMetadata(): Promise<grpc.Metadata> {
    // Renovar se faltam menos de 5 minutos para expirar
    if (Date.now() >= this.expiresAt - 5 * 60 * 1000) {
      await this.refresh();
    }
    const md = new grpc.Metadata();
    md.set("authorization", `Bearer ${this.token}`);
    return md;
  }

  private async refresh(): Promise<void> {
    const resp = await fetch(`${this.apiUrl}/api/v1/auth/token`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ username: this.username, password: this.password }),
    });
    if (!resp.ok) throw new Error("Falha ao renovar token");
    const { access_token } = await resp.json();
    this.token = access_token;
    // Assume expiração de 60 min (configurável via ACCESS_TOKEN_EXPIRE_MINUTES)
    this.expiresAt = Date.now() + 55 * 60 * 1000;
  }
}

// Uso:
const auth = new AuthClient("usuario", "senha", "https://uptoken.cloud");
const metadata = await auth.getMetadata();
const stream = ohlcClient.StreamOHLC(request, metadata);
```

---

## Checklist de Segurança

- [ ] **`REST_JWT_SECRET`** — usar chave forte (minimo 32 caracteres aleatorios), armazenada no gerenciador de senhas
- [ ] **`GRPC_JWT_SECRET`** — manter configurada para compatibilidade/futuro uso quando o interceptor passar a usar segredo dedicado
- [ ] **Credenciais de usuário** — armazenadas no Supabase com senhas bcrypt-hashed
- [x] **TLS em produção** — nginx termina TLS em `uptoken.cloud:443` (REST) e `uptoken.cloud:50052` (gRPC)
- [ ] **Rotação de tokens** — implementar renovação automática no cliente
- [ ] **Rotação de secrets** — trocar periodicamente e reiniciar o servidor
- [ ] **Nunca commitar `.env`** — garantir que está no `.gitignore`

