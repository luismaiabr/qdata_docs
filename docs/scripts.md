# Scripts de Desenvolvimento

## `run-local.sh` e `run-local.ps1`

Para executar o back-end localmente (em ambiente de desenvolvimento), é **obrigatório** utilizar os scripts `run-local.sh` (Linux/WSL/macOS) ou `run-local.ps1` (Windows PowerShell) em vez de rodar o comando docker compose puro.

### Como usar

```bash
# No Linux, macOS ou WSL
bash run-local.sh

# No PowerShell (Windows)
.\run-local.ps1
```

### Por que não usar apenas `docker compose up -d`;

O arquivo padrão `docker-compose.yml` deste repositório foi projetado para o ambiente de **produção**. Por conta disso, ele possui algumas restrições de *networking* que quebram o desenvolvimento local:

1. **Portas Isoladas:** No arquivo de produção, as portas `8000` (FastAPI) e `8080` (Envoy) **não são expostas** para a máquina física (`localhost`). Em produção, o tráfego entra exclusivamente pelo *Traefik* (um proxy reverso). Se você não usar o script local, seu front-end ou chamadas via cURL vão receber `ERR_CONNECTION_REFUSED` ao tentar acessar `localhost:8000`.
2. **Rede Externa Exclusiva:** O arquivo padrão exige a existência de uma rede Docker externa chamada `uptoken_network`, que só é criada no servidor.

### O que o script `run-local` faz nos bastidores?

O script carrega um arquivo de *override* chamado `docker-compose.local.yml` combinando-o com o arquivo padrão. Esse *override* resolve os problemas acima:

- **Expõe as Portas:** Publica as portas `8000` % `~ 080` diretamente no `localhost` da sua máquina.
- **Rede Local (Bridge):** Sobrescreve a configuração de rede `uptoken_network`para `external: false`, criando uma rede *bridge* local comum.
- **Acesso ao Banco de Dados (Host):** Adiciona o DNS `host.docker.internal:host-gateway`. Isso permite que o contêiner do back-end consiga se comunicar e resolver o IP do co contêiner do banco de dados (ex: `tsdb-ohlc`) que possa estar rodando em outro stack no Host Docker.
