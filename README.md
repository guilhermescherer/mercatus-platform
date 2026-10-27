# mercatus-platform

Infraestrutura local compartilhada do Mercatus. Atualmente o perfil Compose
`core` fornece um PostgreSQL 17, com um database por serviço. As aplicações Java
são iniciadas separadamente em seus próprios repositórios.

## Pré-requisitos e execução

Docker com Compose v2 e acesso ao daemon. Execute a partir deste repositório:

```bash
cp compose/.env.example compose/.env
docker compose --env-file compose/.env -f compose/docker-compose.yml --profile core config --quiet
docker compose --env-file compose/.env -f compose/docker-compose.yml --profile core up -d
docker compose --env-file compose/.env -f compose/docker-compose.yml --profile core ps
```

O arquivo `compose/.env` é opcional quando os padrões abaixo são suficientes.
Ao usá-lo, passe `--env-file` como nos exemplos. Cada serviço documenta também
o comando de inicialização com os padrões, a partir da sua própria pasta.

## Configuração

| Variável | Padrão | Efeito |
|---|---|---|
| `POSTGRES_USER` | `mercatus` | Usuário criado ao inicializar o volume |
| `POSTGRES_PASSWORD` | `mercatus` | Senha inicial desse usuário |
| `POSTGRES_DB` | `mercatus` | Database inicial neutro |
| `POSTGRES_PORT` | `5432` | Porta publicada no host |

O script `compose/initdb/01-create-databases.sql` cria `catalog`, `customer`,
`inventory` e `order` e concede privilégios ao usuário `mercatus`. Portanto, trocar
`POSTGRES_USER` exige ajustar também esse script; `POSTGRES_DB` deve ser diferente
dos quatro databases criados por ele. Alterar essas variáveis não reconfigura um
volume já inicializado. Se mudar a porta, ajuste as URLs JDBC dos serviços.

O container se chama `mercatus-postgres`, armazena dados no volume nomeado
`postgres-data` do projeto Compose e monta `compose/initdb` somente para leitura.
O healthcheck usa `pg_isready`, com intervalo de 5 segundos, timeout de 3 segundos
e até 10 tentativas. Aguarde o estado saudável antes de iniciar as aplicações.

## Bancos e volumes existentes

Os scripts de `initdb` rodam somente na primeira inicialização de um volume
vazio. Adicionar um database ao SQL não o cria em volumes antigos.

Para listar os databases com a configuração padrão, a partir deste repositório:

```bash
docker compose -f compose/docker-compose.yml --profile core exec postgres psql -U mercatus -d mercatus -c "SELECT datname FROM pg_database ORDER BY datname;"
```

Se `inventory` estiver ausente, crie somente esse database no container existente:

```bash
docker compose -f compose/docker-compose.yml --profile core exec postgres psql -U mercatus -d mercatus -c 'CREATE DATABASE inventory;'
docker compose -f compose/docker-compose.yml --profile core exec postgres psql -U mercatus -d mercatus -c 'GRANT ALL PRIVILEGES ON DATABASE inventory TO mercatus;'
```

O mesmo procedimento se aplica a outro database de serviço ausente. O Compose
provisiona os databases; as tabelas são criadas pelo Flyway ao iniciar cada
serviço. O Inventory atual inicializa seu schema em banco vazio, conforme seu
[README](../mercatus-inventory-service/README.md).

Se `order` estiver ausente, crie somente esse database no volume existente:

```bash
docker compose -f compose/docker-compose.yml --profile core exec postgres psql -U mercatus -d mercatus -c 'CREATE DATABASE "order";'
docker compose -f compose/docker-compose.yml --profile core exec postgres psql -U mercatus -d mercatus -c 'GRANT ALL PRIVILEGES ON DATABASE "order" TO mercatus;'
```

As aspas duplas preservam o nome SQL `order`. Não recrie nem apague o volume
para acrescentar o banco. O [Order](../mercatus-order-service/README.md) aplica
`V1__order.sql` e valida o schema ao iniciar.

## Operação local

```bash
docker compose --env-file compose/.env -f compose/docker-compose.yml --profile core logs -f postgres
docker compose --env-file compose/.env -f compose/docker-compose.yml --profile core stop
docker compose --env-file compose/.env -f compose/docker-compose.yml --profile core down
```

`stop` e `down` preservam os dados do volume. Os comandos que usam `--env-file`
pressupõem que o arquivo foi criado no passo inicial.

## Limites atuais

- Catalog, Customer, Inventory e Order usam por padrão a mesma credencial local
  `mercatus`; o isolamento por role de serviço será implementado conforme
  [ADR-0014](../docs/adr/0014-isolamento-banco-por-credencial.md).
- O Compose contém apenas o perfil `core`; Kafka, outbox e inbox entram na
  fase da saga, conforme [ADR-0015](../docs/adr/0015-mensageria-nasce-na-saga.md).
- Os testes Testcontainers dos serviços criam seus próprios containers e não
  dependem deste Compose estar iniciado.
