# Banco efêmero de testes

O workflow cria um serviço `timescale/timescaledb:latest-pg16` com banco `test_db` e credenciais fictícias. Aguarda `pg_isready`, publica a porta 5432 no runner e executa `alembic upgrade head` antes do pytest. O serviço é removido ao encerrar o job.

`DATABASE_URL` controla as migrações; `TEST_DATABASE_URL` controla explicitamente o teste de ingestão. Ambas apontam para o banco descartável no CI. O teste não usa o `.env` da aplicação nem o engine global de SQLite. Executar a suíte sem a variável de teste não grava dados em produção.

O endpoint de ingestão recebe uma barra, responde 201 e o teste lê o registro persistido em uma sessão independente. Autenticação unitária usa usuários em memória, sem Supabase. Nenhuma credencial de produção é necessária.

Esse isolamento dispensa uma VPS de staging para os testes: o PostgreSQL roda no runner do GitHub. O banco é reconstruído pelas migrações versionadas, sem depender dos scripts de inicialização de desenvolvimento.

Para execução local, consultar o [tutorial da pipeline](github-actions-tutorial.md).

