# Uso da API via Protocolo HTTPS (REST)

O `qData_service` oferece uma interface REST completa utilizando o protocolo HTTPS para o consumo de dados de mercado (OHLC) e controle de estado, como uma alternativa moderna e simplificada ao uso de gRPC.

## Domínios e Rotas Disponíveis

As requisições REST podem ser enviadas para os seguintes domínios:

1. **`https://uptoken.cloud`**: O domínio principal da aplicação. Hospeda tanto o tráfego REST tradicional quanto a raiz da aplicação.
2. **`https://http.uptoken.cloud`**: Um subdomínio de conveniência configurado no proxy principal de forma dedicada para tratar de requisições puramente HTTP/REST. 

*(Dica: Ambas as URLs apontam para a mesma instância do backend FastAPI rodando internamente).*

## Autenticação

Todas as rotas de consumo de dados são protegidas. Você deve obter um token JWT fazendo login:

**Endpoint de Login:**
`POST /api/v1/auth/token`

**Exemplo (cURL):**
```bash
curl -X POST https://http.uptoken.cloud/api/v1/auth/token \
     -H "Content-Type: application/json" \
     -d '{"username": "seu_usuario", "password": "sua_senha"}'
```

A resposta conterá o `access_token`, que deve ser utilizado nos cabeçalhos (`Headers`) das próximas requisições:
`Authorization: Bearer <seu_token>`

## Rota de Distribuição: Recuperando Dados (OHLC)

O principal endpoint para recuperação de barras financeiras e histórico de mercado é:

`GET /api/v1/distribution/ohlc`

### Parâmetros (Query Parameters)
- `symbol` (string): Símbolo do ativo. Ex: `WINJ26`.
- `timeframe` (string): Tempo gráfico. Ex: `M1`, `M5`, `H1`.
- `start_time` (int): Timestamp Unix de início (inclusivo).
- `end_time` (int): Timestamp Unix de término (inclusivo).
- `limit` (int): Limite de registros retornados (padrão 1000).
- `offset` (int): Deslocamento para paginação (padrão 0).
- `order_desc` (bool): Ordem decrescente (padrão true).

### Exemplo (cURL)
```bash
curl -G "https://http.uptoken.cloud/api/v1/distribution/ohlc" \
     --data-urlencode "symbol=WINJ26" \
     --data-urlencode "timeframe=M1" \
     --data-urlencode "limit=10" \
     -H "Authorization: Bearer <seu_token>"
```

### Formato de Resposta (JSON)
Diferente dos streams gRPC, a API REST devolve os dados integralmente formatados em um array JSON pronto para o uso:

```json
[
  {
    "time": 1708261200,
    "timeUtc": "2024-02-18T13:00:00Z",
    "timeSp": "2024-02-18T10:00:00-03:00",
    "open": 129000.0,
    "high": 129500.0,
    "low": 128800.0,
    "close": 129100.0,
    "tickVolume": 1500,
    "spread": 5,
    "realVolume": 15000
  }
]
```

## Explorando a API (Swagger UI)

Como o backend é construído com FastAPI, uma documentação interativa e completa (OpenAPI/Swagger) está automaticamente disponível.

Você pode explorar todas as rotas de HTTPS disponíveis acessando:
🔗 **https://uptoken.cloud/docs**

Através da interface Swagger, é possível autorizar-se, preencher os parâmetros em um formulário e testar diretamente as chamadas REST direto pelo navegador.

