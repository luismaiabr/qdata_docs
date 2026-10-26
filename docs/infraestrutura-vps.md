# Proposta de infraestrutura como código para VPS

## Escolha inicial

Para a infraestrutura atual, proponho Docker Compose + Traefik + Ansible. Compose descreve serviços e redes; Traefik centraliza roteamento/TLS; Ansible configura host, usuários, Docker, firewall, diretórios e serviços systemd com playbooks reaplicáveis. O inventário separa cada VPS.

Essa é uma recomendação para o cenário do projeto. Docker descreve [Compose em um único servidor](https://docs.docker.com/compose/how-tos/production/), e Ansible explica [playbooks e convergência para o estado desejado](https://docs.ansible.com/projects/ansible-core/devel/playbook_guide/playbooks_intro.html). Módulos e tarefas devem ser escolhidos para garantir idempotência.

Terraform entra quando houver API/provider do provedor para criar VPS, redes, DNS e volumes. Não substitui o gerenciamento do sistema operacional pelo Ansible: [providers e recursos](https://developer.hashicorp.com/terraform/language/block/provider). Sem API, começar pelo inventário das VPS existentes já traz benefício.

Kubernetes pode ser reavaliado quando houver necessidade de agendamento entre hosts, escalabilidade e capacidade operacional para manter o cluster. Com uma VPS, a prioridade proposta é recuperação e deploy previsível. Uma segunda VPS pode servir primeiro para staging e teste de restauração.

## Entrega gradual

1. Versionar inventário sem segredos, roles Ansible e Compose de produção. Manter `.env`, chaves e dumps fora do Git; usar cofre de segredos.
2. Fazer backup e testar restauração do PostgreSQL. Manter versão anterior da aplicação e configurações para rollback, sem excluir volumes.
3. No Actions: lint, testes, build de imagem, varredura de dependências e publicação no registry com tag do SHA. O workflow atual cobre testes.
4. Criar job de deploy separado, com environment protegido, aprovação e credenciais limitadas. Implantar a imagem pelo SHA, verificar healthcheck e manter rollback.
5. Configurar monitoramento, rotação de logs e alertas; validar recuperação após reboot antes de aumentar complexidade.

## Divergência da cópia de produção

Antes de sincronizar a VPS, registrar SHA, `git status`, diff e arquivos não versionados; guardar backup de configuração e código fora do diretório a substituir. Comparar os ajustes locais de Compose, rede e Traefik com `main`. Não usar `reset --hard` sem preservar diferenças e confirmar o plano.

Preferir diretório de release separado ou imagens pelo SHA. Preservar `.env` e volumes, testar Compose e conectividade, trocar a aplicação e verificar saúde. Migrações de banco exigem plano próprio: voltar a imagem anterior não desfaz automaticamente o schema.

