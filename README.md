# mercatus-platform

Infraestrutura local compartilhada do Mercatus. O perfil Compose `core` fornece
um PostgreSQL 17, com um database por serviço. O perfil `events` fornece um
broker Apache Kafka em modo KRaft (nó único, sem Zookeeper) e cria os tópicos da
saga. O perfil `security` fornece o Keycloak, provedor de identidade que emite os
tokens JWT validados pelos serviços. As aplicações Java são iniciadas separadamente em seus próprios
repositórios.

## Pré-requisitos e execução

Docker com Compose v2 e acesso ao daemon. Execute a partir deste repositório:

```bash
cp compose/.env.example compose/.env
docker compose --env-file compose/.env -f compose/docker-compose.yml --profile core config --quiet
docker compose --env-file compose/.env -f compose/docker-compose.yml --profile core up -d
docker compose --env-file compose/.env -f compose/docker-compose.yml --profile core ps
```

Para subir também o Kafka da saga, acrescente o perfil `events`:

```bash
docker compose --env-file compose/.env -f compose/docker-compose.yml --profile core --profile events config --quiet
docker compose --env-file compose/.env -f compose/docker-compose.yml --profile core --profile events up -d
docker compose --env-file compose/.env -f compose/docker-compose.yml --profile core --profile events ps
```

O serviço `kafka-topics` roda uma vez, cria `inventory.commands.v1` e
`inventory.events.v1` e encerra com código 0; vê-lo como `Exited (0)` em `ps` é o
estado esperado.

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
| `KAFKA_PORT` | `9092` | Porta do broker Kafka publicada no host (perfil `events`) |
| `KEYCLOAK_PORT` | `8180` | Porta do Keycloak publicada no host (perfil `security`); faz parte do issuer dos tokens |
| `KEYCLOAK_ADMIN` | `admin` | Usuário do console administrativo (realm `master`) |
| `KEYCLOAK_ADMIN_PASSWORD` | `admin` | Senha desse usuário |

O script `compose/initdb/01-create-databases.sql` cria `catalog`, `customer`,
`inventory`, `order` e `keycloak` e concede privilégios ao usuário `mercatus`. Portanto, trocar
`POSTGRES_USER` exige ajustar também esse script; `POSTGRES_DB` deve ser diferente
dos databases criados por ele. Alterar essas variáveis não reconfigura um
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

## Perfil `events` (Kafka)

O perfil `events` sobe um broker Apache Kafka 3.9 em modo KRaft com um único nó
(papéis `broker` e `controller` no mesmo processo, sem Zookeeper), adequado ao
ambiente local de 16 GB. O
heap da JVM é limitado a 512 MB (`KAFKA_HEAP_OPTS`), o que mantém o consumo bem
abaixo de 1 GB em uso leve.

O broker expõe dois listeners de dados: `HOST`, anunciado como
`localhost:${KAFKA_PORT}` para clientes no host (as aplicações Java iniciadas
localmente), e `INTERNAL`, anunciado como `kafka:29092` para outros containers na
rede do Compose. O listener `CONTROLLER` é interno ao KRaft. Clientes rodando no
host usam `localhost:9092`.

O serviço `kafka-topics` depende do broker saudável, cria os tópicos
`inventory.commands.v1` e `inventory.events.v1` (uma partição, fator de
replicação 1, chave `orderId`) com `--if-not-exists` e encerra. A criação
automática de tópicos está desligada (`KAFKA_AUTO_CREATE_TOPICS_ENABLE=false`)
para que apenas os tópicos declarados existam.

Para listar os tópicos com o broker no ar:

```bash
docker compose -f compose/docker-compose.yml --profile events exec kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

O volume nomeado `kafka-data` preserva os logs do broker entre reinícios. Os
testes Testcontainers dos serviços criam seus próprios brokers e não dependem
deste Compose.

## Perfil `security` (Keycloak)

O perfil `security` sobe o Keycloak 26 em modo de desenvolvimento (HTTP, sem
TLS), com heap limitado a 512 MB. Ele guarda usuários e configurações no
database `keycloak` do Postgres, por isso exige também o perfil `core`:

```bash
docker compose -f compose/docker-compose.yml --profile core --profile security config --quiet
docker compose -f compose/docker-compose.yml --profile core --profile security up -d --wait
```

Em um volume do Postgres criado antes deste perfil, crie o database uma vez,
como descrito em [Bancos e volumes existentes](#bancos-e-volumes-existentes):

```bash
docker compose -f compose/docker-compose.yml --profile core exec postgres psql -U mercatus -d mercatus -c 'CREATE DATABASE keycloak;'
docker compose -f compose/docker-compose.yml --profile core exec postgres psql -U mercatus -d mercatus -c 'GRANT ALL PRIVILEGES ON DATABASE keycloak TO mercatus;'
```

O console administrativo fica em `http://localhost:8180/admin` (usuário e senha
`admin`, no realm `master`). O healthcheck consulta `/health/ready` na porta de
gestão 9000, interna ao container.

### Realm `mercatus`

O realm é versionado em `compose/keycloak/mercatus-realm.json` e importado na
subida (`--import-realm`). O issuer dos tokens é
`http://localhost:8180/realms/mercatus`; os serviços rodando no host usam esse
valor para validar os tokens.

| Item | Valor |
|---|---|
| Papéis de realm | `CUSTOMER`, `OPERATOR`, `ADMIN` (composto: inclui `OPERATOR`), `SERVICE` |
| Usuários de teste | `customer1`, `customer2` (`CUSTOMER`), `operator` (`OPERATOR`), `admin` (`ADMIN`); senha igual ao usuário |
| Cliente `mercatus-cli` | público, para obter tokens de usuário: fluxo padrão com PKCE (callback do Postman) e *direct access grants* |
| Cliente `order-service` | confidencial, só *service accounts* (client credentials), papel `SERVICE`, secret `order-service-dev-secret` |
| Validade do access token | 5 minutos |

Os papéis chegam no token em `realm_access.roles`. Como `ADMIN` é composto, o
token do administrador traz `ADMIN` e `OPERATOR`.

As senhas, o secret e o *direct access grants* (password grant, fora do OAuth
2.1) existem apenas para o laboratório local: não há frontend, e scripts e
coleções HTTP precisam obter tokens diretamente.

Para obter um token de usuário e um token de serviço:

```bash
curl -s http://localhost:8180/realms/mercatus/protocol/openid-connect/token \
  -d grant_type=password -d client_id=mercatus-cli \
  -d username=customer1 -d password=customer1

curl -s http://localhost:8180/realms/mercatus/protocol/openid-connect/token \
  -d grant_type=client_credentials -d client_id=order-service \
  -d client_secret=order-service-dev-secret
```

O campo `access_token` da resposta vai no header `Authorization: Bearer <token>`.

### Alterar o realm

A importação só acontece quando o realm ainda não existe; nas subidas seguintes
o log mostra `Realm 'mercatus' already exists. Import skipped`. Para aplicar uma
mudança feita no arquivo, apague o realm `mercatus` pelo console e reinicie o
Keycloak, ou recrie o database `keycloak`:

```bash
docker compose -f compose/docker-compose.yml --profile core --profile security stop keycloak
docker compose -f compose/docker-compose.yml --profile core exec postgres psql -U mercatus -d mercatus -c 'DROP DATABASE keycloak;' -c 'CREATE DATABASE keycloak;'
docker compose -f compose/docker-compose.yml --profile core --profile security up -d --wait keycloak
```

Recriar o database apaga usuários criados manualmente e gera novos `sub` para os
usuários de teste.

## Limites atuais

- Catalog, Customer, Inventory e Order usam por padrão a mesma credencial local
  `mercatus`; o isolamento por role de serviço (uma credencial por serviço) será
  implementado nas fases de infraestrutura.
- O Compose fornece os perfis `core` (Postgres), `events` (Kafka) e `security`
  (Keycloak). RabbitMQ e demais dependências entram nas fases que os exigem.
- O issuer usa `localhost:8180`, adequado a serviços rodando no host. Serviços
  dentro da rede do Compose não alcançariam esse endereço.
- Os testes Testcontainers dos serviços criam seus próprios containers e não
  dependem deste Compose estar iniciado.
