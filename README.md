# Migração PostgreSQL para PostgreSQL com OCI GoldenGate

Este repositório descreve um procedimento para migrar dados de um PostgreSQL de origem para um PostgreSQL de destino usando **OCI GoldenGate**, com carga inicial (*initial load*) e captura contínua de alterações (*CDC*).

O objetivo é reduzir a indisponibilidade: os dados existentes são carregados uma vez, enquanto as alterações posteriores são capturadas e aplicadas até o *cutover*.

> **Importante:** execute em ambiente de homologação antes da produção. Não armazene senhas, chaves privadas, arquivos `pgpass` ou OCIDs sensíveis neste repositório.

## Arquitetura

```text
PostgreSQL origem                         PostgreSQL destino
      |                                         ^
      |  Initial Load (trail I1)                 |
      |  CDC (trail C1)                          |
      v                                         |
        OCI GoldenGate PostgreSQL Deployment
          - Private Endpoint
          - Extracts: EIL / ECDC
          - Replicats: RIL / RCDC
```

O **Private Endpoint** permite que o deployment acesse os bancos pelos seus IPs privados. O acesso público ao Console GoldenGate é opcional e não é necessário para o tráfego de replicação.

## Variáveis usadas neste documento

Substitua os valores abaixo antes da execução.

| Variável | Exemplo | Descrição |
|---|---|---|
| `<DB_ORIGEM>` | `appdb` | Banco PostgreSQL de origem |
| `<DB_DESTINO>` | `appdb_migrado` | Banco PostgreSQL de destino |
| `<HOST_ORIGEM>` | `10.10.10.10` | Host ou FQDN da origem |
| `<HOST_DESTINO>` | `10.20.20.10` | Host ou FQDN do destino |
| `<SCHEMA_APP>` | `app` | Schema que será replicado |
| `<OWNER_APP>` | `app_owner` | Dono das tabelas do schema |
| `<CIDR_GOLDENGATE>` | `10.210.50.0/24` | CIDR/IPs do endpoint GoldenGate |
| `<SENHA_FORTE>` | `***` | Senha do usuário GoldenGate — não versionar no GitHub |

## Usuário GoldenGate

Este procedimento usa o mesmo nome de usuário, `ggadmin`, na origem e no destino. Como são bancos PostgreSQL independentes, não há conflito entre as roles. As permissões, porém, são diferentes: na origem o usuário captura alterações; no destino ele aplica as alterações.

Você pode usar senhas diferentes na origem e no destino, mesmo que o nome do usuário seja igual. Evite reutilizar a senha da aplicação ou usar o usuário dono das tabelas. Nunca grave a senha real neste README, em scripts versionados ou em arquivos `.env` enviados ao GitHub.

## Pré-requisitos

- PostgreSQL de origem e destino em versão compatível com OCI GoldenGate.
- Conectividade TCP/5432 do Private Endpoint GoldenGate até origem e destino.
- DNS resolvendo os FQDNs privados, quando forem usados.
- Regras de NSG/Security List liberando apenas os IPs/CIDRs necessários.
- Chaves primárias ou índices únicos nas tabelas replicadas, sempre que possível.
- Capacidade de WAL suficiente na origem enquanto o CDC estiver ativo.
- Um OCI Vault e Secrets para as senhas das Connections, preferencialmente.

## 1. Preparar PostgreSQL de origem

### 1.1 Validar replicação lógica

Conecte como administrador:

```sql
SHOW wal_level;
SHOW max_replication_slots;
SHOW max_wal_senders;
SHOW track_commit_timestamp;
```

Valores esperados:

```text
wal_level = logical
max_replication_slots >= 1
max_wal_senders >= 1
track_commit_timestamp = on
```

Em PostgreSQL autogerenciado, ajuste o `postgresql.conf`:

```conf
listen_addresses = '*'
wal_level = logical
max_replication_slots = 4
max_wal_senders = 4
track_commit_timestamp = on
```

No `pg_hba.conf`, permita somente a rede do GoldenGate:

```conf
host    <DB_ORIGEM>    ggadmin    <CIDR_GOLDENGATE>    scram-sha-256
```

Reinicie o serviço após a alteração:

```bash
sudo systemctl restart postgresql
```

Para **OCI Database with PostgreSQL**, altere os parâmetros pela configuração do DB System; não tente editar `postgresql.conf` diretamente no serviço gerenciado.

### 1.2 Criar o usuário de captura

Execute no banco `<DB_ORIGEM>` como administrador:

```sql
CREATE ROLE ggadmin
  LOGIN
  PASSWORD '<SENHA_FORTE>'
  NOSUPERUSER
  NOCREATEDB
  NOCREATEROLE
  NOINHERIT;

GRANT CONNECT ON DATABASE <DB_ORIGEM> TO ggadmin;
ALTER ROLE ggadmin WITH REPLICATION;

GRANT USAGE ON SCHEMA <SCHEMA_APP> TO ggadmin;
GRANT SELECT ON ALL TABLES IN SCHEMA <SCHEMA_APP> TO ggadmin;

ALTER DEFAULT PRIVILEGES FOR ROLE <OWNER_APP>
  IN SCHEMA <SCHEMA_APP>
  GRANT SELECT ON TABLES TO ggadmin;

CREATE SCHEMA IF NOT EXISTS ogg AUTHORIZATION ggadmin;
```

Valide o usuário:

```sql
SELECT rolname, rolreplication, rolsuper
FROM pg_roles
WHERE rolname = 'ggadmin';
```

### 1.3 Adicionar TRANDATA

No **GoldenGate Deployment Console**:

1. Abra **DB Connections** e conecte em `PG_SOURCE`.
2. Em **TRANDATA Information**, clique no ícone `+`.
3. Informe `<SCHEMA_APP>.*`.
4. Clique em **Submit** e valide a lista de tabelas.

Em alguns ambientes, o usuário que executa a ação precisa de `SUPERUSER` temporariamente:

```sql
ALTER ROLE ggadmin WITH SUPERUSER;
```

Após adicionar TRANDATA, remova o privilégio:

```sql
ALTER ROLE ggadmin WITH NOSUPERUSER;
```

Em OCI Database with PostgreSQL, quando `SUPERUSER` não estiver disponível, use o administrador do DB System somente para habilitar TRANDATA e mantenha `ggadmin` com o menor privilégio possível.

## 2. Preparar PostgreSQL de destino

### 2.1 Exportar somente a estrutura

Em uma VM com cliente PostgreSQL compatível:

```bash
pg_dump \
  -h <HOST_ORIGEM> \
  -p 5432 \
  -U <OWNER_APP> \
  -d <DB_ORIGEM> \
  -n <SCHEMA_APP> \
  -s -E UTF8 \
  -f schema_only.sql
```

O parâmetro `-s` exporta apenas DDL. Os dados serão carregados pelo GoldenGate.

### 2.2 Restaurar a estrutura

```bash
psql \
  -h <HOST_DESTINO> \
  -p 5432 \
  -U <ADMIN_DESTINO> \
  -d <DB_DESTINO> \
  -f schema_only.sql
```

### 2.3 Criar o usuário de aplicação

Execute no banco `<DB_DESTINO>` como administrador:

```sql
CREATE ROLE ggadmin
  LOGIN
  PASSWORD '<SENHA_FORTE>'
  NOSUPERUSER
  NOCREATEDB
  NOCREATEROLE
  NOINHERIT;

GRANT CONNECT ON DATABASE <DB_DESTINO> TO ggadmin;
GRANT USAGE ON SCHEMA <SCHEMA_APP> TO ggadmin;

GRANT SELECT, INSERT, UPDATE, DELETE, TRUNCATE
ON ALL TABLES IN SCHEMA <SCHEMA_APP>
TO ggadmin;

GRANT USAGE, SELECT, UPDATE
ON ALL SEQUENCES IN SCHEMA <SCHEMA_APP>
TO ggadmin;

ALTER DEFAULT PRIVILEGES FOR ROLE <OWNER_APP>
  IN SCHEMA <SCHEMA_APP>
  GRANT SELECT, INSERT, UPDATE, DELETE, TRUNCATE
  ON TABLES TO ggadmin;

ALTER DEFAULT PRIVILEGES FOR ROLE <OWNER_APP>
  IN SCHEMA <SCHEMA_APP>
  GRANT USAGE, SELECT, UPDATE
  ON SEQUENCES TO ggadmin;

CREATE SCHEMA IF NOT EXISTS ogg AUTHORIZATION ggadmin;
```

No Deployment Console, conecte em `PG_TARGET` e crie a checkpoint table:

```text
ogg.checkpoint
```

## 3. Criar OCI GoldenGate Deployment e Connections

1. Acesse **Oracle Database > GoldenGate > Deployments**.
2. Clique em **Create deployment** e selecione a tecnologia **PostgreSQL**.
3. Escolha a VCN e a subnet privada que alcança ambos os bancos.
4. Crie o deployment e aguarde o status **Active**.
5. No menu principal do OCI GoldenGate, acesse **Connections > Create connection**. Esta é a tela de criação da Connection gerenciada pela OCI; ela é diferente de **DB Connections** dentro do Deployment Console.
6. Crie a Connection `PG_SOURCE` com os campos abaixo:

   ```text
   Type: OCI PostgreSQL
   Database details: Enter PostgreSQL database information
   Database name: <DB_ORIGEM>
   Host: <HOST_ORIGEM>
   Port: 5432
   Database username: ggadmin
   Security protocol: TLS
   SSL mode: Require
   ```

   Em **Advanced options > Security**, desmarque **Use vault secrets** se a senha for informada diretamente. Enquanto essa opção estiver marcada, o campo **Database user password secret** será obrigatório.

7. Crie a Connection `PG_TARGET` com os mesmos parâmetros, substituindo o banco e host pelos valores de destino:

   ```text
   Type: OCI PostgreSQL
   Database details: Enter PostgreSQL database information
   Database name: <DB_DESTINO>
   Host: <HOST_DESTINO>
   Port: 5432
   Database username: ggadmin
   Security protocol: TLS
   SSL mode: Require
   ```

   Em **Advanced options > Security**, também desmarque **Use vault secrets** caso não utilize OCI Vault.

   > Não informe `EncryptionMethod`, `sslMode` ou qualquer parâmetro ODBC nos campos **Host** ou **Database name** desta tela. Esses parâmetros são aplicáveis somente a conexões ODBC diretas no Console interno do GoldenGate.

8. Abra o deployment, acesse **Assigned connections** e clique em **Assign connection** para associar `PG_SOURCE` e `PG_TARGET`.
9. Abra **Launch Console**. Em **DB Connections**, confirme que as Connections atribuídas aparecem e teste ambas.

Para liberar temporariamente o console pela Internet, edite o deployment, marque **Enable GoldenGate console public access** e selecione uma subnet pública na mesma VCN. Esse recurso cria um Load Balancer, sujeito a cobrança adicional. A replicação com os bancos continua pelo Private Endpoint.

## 4. Initial Load

### 4.1 Criar o Initial Load Extract

No Deployment Console:

1. Acesse **Extracts > + Add Extract**.
2. Tipo: **Initial Load Extract**.
3. Nome: `EIL`.
4. Credenciais: alias `PG_SOURCE`.
5. Trail: `I1`.
6. Use os parâmetros:

```text
EXTRACT EIL
USERIDALIAS PG_SOURCE, DOMAIN OracleGoldenGate
EXTFILE I1, PURGE
TABLE <SCHEMA_APP>.*;
```

7. Clique em **Create and Run**.
8. Quando concluir, abra o Report do `EIL` e registre o **LSN** mostrado nele.

### 4.2 Criar o Initial Load Replicat

1. Acesse **Replicats > + Add Replicat**.
2. Tipo: **Classic Replicat**.
3. Nome: `RIL`.
4. Trail de origem: `I1`.
5. Credenciais de destino: `PG_TARGET`.
6. Checkpoint table: `ogg.checkpoint`.
7. Use os parâmetros:

```text
REPLICAT RIL
USERIDALIAS PG_TARGET, DOMAIN OracleGoldenGate
MAP <SCHEMA_APP>.*, TARGET <SCHEMA_APP>.*;
```

8. Clique em **Create and Run**.
9. Ao terminar, valide as contagens de linhas em origem e destino.

Exemplo:

```sql
SELECT COUNT(*) FROM <SCHEMA_APP>.<TABELA>;
```

## 5. CDC (captura contínua)

### 5.1 Criar o Extract CDC

1. Acesse **Extracts > + Add Extract**.
2. Tipo: **Change Data Capture Extract**.
3. Nome: `ECDC`.
4. Credenciais: `PG_SOURCE`.
5. Trail: `C1`.
6. Use os parâmetros:

```text
EXTRACT ECDC
USERIDALIAS PG_SOURCE, DOMAIN OracleGoldenGate
EXTTRAIL C1
TABLE <SCHEMA_APP>.*;
```

7. Posicione o Extract para iniciar no **LSN** anotado no passo do Initial Load Extract.
8. Clique em **Create and Run**.

### 5.2 Criar o Replicat CDC

1. Acesse **Replicats > + Add Replicat**.
2. Tipo: **Classic Replicat** inicialmente.
3. Nome: `RCDC`.
4. Trail: `C1`.
5. Credenciais: `PG_TARGET`.
6. Checkpoint table: `ogg.checkpoint`.
7. Use os parâmetros:

```text
REPLICAT RCDC
USERIDALIAS PG_TARGET, DOMAIN OracleGoldenGate
MAP <SCHEMA_APP>.*, TARGET <SCHEMA_APP>.*;
```

8. Clique em **Create**, revise os parâmetros e inicie o processo.
9. Monitore o lag até ficar próximo de zero.

## 6. Validações

### Processos GoldenGate

- `EIL` e `RIL`: concluídos sem erro.
- `ECDC`: `Running`.
- `RCDC`: `Running`.
- Lag do Replicat: próximo de zero.
- Sem erros nos Reports de Extract e Replicat.

### Dados

Faça ao menos validação de contagem para tabelas críticas:

```sql
SELECT '<SCHEMA_APP>.<TABELA>' AS tabela, COUNT(*) AS linhas
FROM <SCHEMA_APP>.<TABELA>;
```

Também valide amostras funcionais, chaves primárias, sequences, dados recentes e objetos dependentes, como views, funções, extensões e permissões da aplicação.

## 7. Cutover

1. Com `ECDC` e `RCDC` em execução, valide o lag.
2. Pare a aplicação ou bloqueie escritas na origem.
3. Aguarde o `RCDC` aplicar todas as alterações pendentes.
4. Confirme lag zero e valide os dados finais.
5. Atualize a connection string da aplicação para `<HOST_DESTINO>`.
6. Faça teste funcional da aplicação no destino.
7. Mantenha a origem em modo somente leitura durante a janela de contingência acordada.

## Referências oficiais

- [OCI GoldenGate: criar recursos de replicação](https://docs.oracle.com/en/cloud/paas/goldengate-service/ocigg/create-data-replication-resources.html)
- [Conectar OCI GoldenGate ao OCI Database with PostgreSQL](https://docs.oracle.com/en/cloud/paas/goldengate-service/ocigg/connections/connect-to-oci-database-with-postgresql.html)
- [Adicionar Extract para PostgreSQL](https://docs.oracle.com/en/cloud/paas/goldengate-service/ocigg/replicate/add-an-extract-for-postgresql.html)
- [Adicionar Replicat para PostgreSQL](https://docs.oracle.com/en/cloud/paas/goldengate-service/zrzeq/)
- [Instanciação consistente PostgreSQL com pg_dump e LSN](https://docs.oracle.com/en/database/goldengate/core/26/coredoc/instantiate-add-initial-load-extract-postgresql-ma-1.html)
