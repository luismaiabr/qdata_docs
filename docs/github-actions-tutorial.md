# Pipeline de testes com GitHub Actions

O workflow `testes.yml` valida pushes em `main`/`githubactions`, pull requests para `main` e execuções manuais. O job `testes` instala Python 3.13 e dependências, inicia TimescaleDB/PostgreSQL descartável, aplica `alembic upgrade head` e executa `pytest tests/ -v`. O log fica disponível como artefato por sete dias.

## Isolamento

O job usa valores fictícios definidos no workflow. Não exige GitHub Environment, secret `ENV`, `.env` de produção ou acesso ao Supabase. Os testes de autenticação usam usuários em memória; isso não substitui testes do fluxo real com Supabase.

`DATABASE_URL` controla as migrações. O teste de ingestão exige separadamente `TEST_DATABASE_URL` e cria sua própria sessão, sem herdar o engine SQLite dos testes unitários nem a URL de produção do `.env`. Valida a resposta HTTP e o registro persistido.

## Execução local

Com Docker ativo e [act](https://github.com/nektos/act) instalado:

```bash
act push --artifact-server-path /tmp/qData-artifacts
# Se necessário, selecionar uma imagem mais completa:
act push -P ubuntu-latest=catthehacker/ubuntu:act-latest --artifact-server-path /tmp/qData-artifacts
```

Não passar secrets ou o `.env` de produção para `act`. No GitHub, abrir **Actions → Testes → Run workflow** para execução manual.

Para rodar diretamente com Python, instalar dependências do projeto e de teste conforme o workflow. Com PostgreSQL descartável disponível:

```bash
export PYTHONPATH=src
export DATABASE_URL='postgresql://test_user:test_password@localhost:5432/test_db'
export TEST_DATABASE_URL="$DATABASE_URL"
alembic upgrade head
python -m pytest tests/ -v
```

Sem `TEST_DATABASE_URL`, a integração de ingestão é ignorada localmente. No CI, a ausência dessa variável falha o teste. Os testes unitários usam SQLite.

## Diagnóstico

- Docker indisponível: iniciar o daemon/Docker Desktop antes de usar `act`.
- Integração sem tabelas: aplicar Alembic no mesmo banco de `TEST_DATABASE_URL`.
- Dependências: o grupo `main` está no Poetry; o grupo `dev` é instalado com `pip` conforme o workflow, que usa Poetry 2.1.3.

Recomenda-se proteger `main` exigindo o check `testes`. O evento `pull_request` valida PRs antes do merge e funciona sem secrets, inclusive para forks: [documentação oficial](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows).
