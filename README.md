# OCI GoldenGate: PostgreSQL para PostgreSQL

Guia genérico para configurar replicação PostgreSQL-to-PostgreSQL com OCI GoldenGate Data Replication.

> Substitua os valores entre `<...>` pelos dados do ambiente. Valide todo o procedimento em homologação antes de produção.

## Arquitetura

```text
PostgreSQL origem
        |
        | Extract / logical replication
        v
OCI GoldenGate Deployment (PostgreSQL)
        |
        | Distribution Path
        v
PostgreSQL destino
        |
        v
Replicat
```

Um único Deployment pode hospedar Extract e Replicat em ambientes simples. Para produção, avalie Deployments separados conforme volume, isolamento, disponibilidade e operação.

## 1. Pré-requisitos

Antes de iniciar, reúna:

- Versão, distribuição e patch level de PostgreSQL na origem e no destino. Confirme suporte na matriz do OCI GoldenGate e no build escolhido do Deployment.
- Host/DNS, porta (`5432` por padrão), database e schemas de origem e destino.
- CIDRs, VCNs, subnets, NSGs, firewall e rotas entre GoldenGate e os bancos.
- Usuários dedicados: `ggextract` na origem e `ggreplicat` no destino.
- Uma estratégia de carga inicial e ponto de início da captura.
- Chave primária ou chave única confiável nas tabelas replicadas.

Para PostgreSQL Server autogerenciado, a origem deve aceitar conexões remotas e estar preparada para logical replication. Para serviços gerenciados, siga os parâmetros equivalentes disponibilizados pelo provedor.

## 2. IAM

Este roteiro usa o grupo padrão `Administrators`, sem o prefixo do Identity Domain.

```text
allow group Administrators to manage all-resources in tenancy
```

O GoldenGate também precisa das policies abaixo para trabalhar com Vault, chaves e IAM Identity Domains:

```text
allow service goldengate to use keys in tenancy
allow service goldengate to use vaults in tenancy
allow service goldengate to {idcs_user_viewer, domain_resources_viewer} in tenancy
```

### Dynamic Group para secrets

1. Acesse **Identity & Security → Dynamic Groups → Create Dynamic Group**.
2. Crie, por exemplo, `dg-ogg-deployments`.
3. Use a regra abaixo, substituindo o OCID do compartment do Deployment:

```text
ALL {resource.type = 'goldengatedeployment', resource.compartment.id = '<compartment-ocid>'}
```

4. Crie a policy:

```text
allow dynamic-group dg-ogg-deployments to read secret-bundles in tenancy
```

## 3. Rede

Crie ou reserve uma subnet privada e sem sobreposição dentro da VCN. Exemplo:

```text
VCN:                  10.0.0.0/16
Subnet GoldenGate:    10.0.10.0/24
```

Configure NSGs ou Security Lists:

| Origem | Destino | Protocolo/porta | Finalidade |
|---|---|---|---|
| Bastion, VPN ou rede administrativa | Endpoint GoldenGate | TCP 443 | Console e Admin Client |
| Ingress IPs do GoldenGate | PostgreSQL origem | TCP 5432 ou porta real | Extract / captura lógica |
| Ingress IPs do GoldenGate | PostgreSQL destino | TCP 5432 ou porta real | Replicat |
| Deployment origem | Deployment destino | TCP 443 | Distribution Path remoto |

Para connections com **Shared endpoint**, permita os **Ingress IPs do Deployment**. Para **Dedicated endpoint**, permita os **Ingress IPs exibidos nos detalhes da connection**.

Se os bancos estiverem on-premises ou em outra VCN, valide DRG/LPG, VPN/FastConnect, DNS e rota de retorno. A conexão deve ser permitida também pelo `pg_hba.conf` da origem e do destino.

## 4. OCI Vault e secrets

### Criar Vault e chave

1. Acesse **Identity & Security → Vault → Create Vault**.
2. Crie, por exemplo, `VAULT-OGG`, e aguarde `Active`.
3. No Vault, acesse **Master Encryption Keys → Create Key** e crie `KEY-OGG-SECRETS`.

### Criar secrets

Crie secrets separados, usando **Manual secret generation** e armazenando somente a senha:

```text
OGG-PG-SRC-PASSWORD  → senha de ggextract na origem
OGG-PG-TGT-PASSWORD  → senha de ggreplicat no destino
```

Ao trocar uma senha, crie uma nova versão do secret e execute **Refresh connection** na connection correspondente.

## 5. Preparar PostgreSQL origem

Execute os comandos como superusuário do PostgreSQL ou com uma conta administrativa equivalente. A configuração de logical replication requer reinício do serviço quando modificada.

### Ajustar `postgresql.conf`

No arquivo de configuração, defina valores adequados à quantidade de Extracts e à retenção de WAL do ambiente:

```text
listen_addresses = '*'
wal_level = logical
max_replication_slots = 4
max_wal_senders = 4
track_commit_timestamp = on
```

`max_replication_slots` e `max_wal_senders` devem acomodar todos os processos de replicação e uma margem operacional. Não use o valor de exemplo sem dimensionamento.

Reinicie PostgreSQL após a alteração:

```bash
sudo systemctl restart postgresql
```

### Ajustar `pg_hba.conf`

Permita somente a rede ou os Ingress IPs necessários do GoldenGate. Exemplo para uma subnet privada:

```text
host    <database-origem>    ggextract    10.0.10.0/24    scram-sha-256
```

Evite regras abertas como `0.0.0.0/0`. Recarregue a configuração após a alteração:

```sql
SELECT pg_reload_conf();
```

### Criar o usuário Extract

```sql
CREATE ROLE ggextract LOGIN PASSWORD '<senha-segura>' REPLICATION;
GRANT CONNECT ON DATABASE <database-origem> TO ggextract;
GRANT USAGE ON SCHEMA <schema-aplicacao> TO ggextract;
GRANT SELECT ON ALL TABLES IN SCHEMA <schema-aplicacao> TO ggextract;
ALTER DEFAULT PRIVILEGES IN SCHEMA <schema-aplicacao>
  GRANT SELECT ON TABLES TO ggextract;
```

Crie também um schema para objetos GoldenGate, caso o usuário não seja dono dele:

```sql
CREATE SCHEMA ggschema AUTHORIZATION ggextract;
GRANT CREATE ON DATABASE <database-origem> TO ggextract;
```

Para executar `ADD TRANDATA`, o usuário Extract pode necessitar de `SUPERUSER` temporariamente. Prefira que um DBA execute a etapa, ou conceda o privilégio apenas durante a configuração e revogue-o em seguida:

```sql
ALTER ROLE ggextract WITH SUPERUSER;
-- Execute ADD TRANDATA no GoldenGate para as tabelas necessárias.
ALTER ROLE ggextract WITH NOSUPERUSER;
```

Em serviços gerenciados, o superusuário pode não estar disponível. Nesse caso, habilite TRANDATA com a conta administradora do serviço e mantenha o usuário GoldenGate com o menor privilégio possível.

### Validar origem

```sql
SHOW wal_level;
SHOW max_replication_slots;
SHOW max_wal_senders;
SELECT rolname, rolreplication, rolsuper
FROM pg_roles
WHERE rolname = 'ggextract';
```

O esperado é `wal_level = logical`, capacidade disponível de slots/senders e `rolreplication = true` para `ggextract`.

## 6. Preparar PostgreSQL destino

Configure acesso remoto no `postgresql.conf` e `pg_hba.conf` do destino da mesma forma, restringindo a origem aos IPs necessários. Crie o usuário Replicat:

```sql
CREATE ROLE ggreplicat LOGIN PASSWORD '<senha-segura>';
GRANT CONNECT ON DATABASE <database-destino> TO ggreplicat;
GRANT USAGE ON SCHEMA <schema-aplicacao> TO ggreplicat;
GRANT INSERT, UPDATE, DELETE, TRUNCATE ON ALL TABLES IN SCHEMA <schema-aplicacao> TO ggreplicat;
ALTER DEFAULT PRIVILEGES IN SCHEMA <schema-aplicacao>
  GRANT INSERT, UPDATE, DELETE, TRUNCATE ON TABLES TO ggreplicat;
```

Crie objetos de checkpoint e heartbeat em um schema dedicado:

```sql
CREATE SCHEMA ggschema AUTHORIZATION ggreplicat;
GRANT CREATE ON DATABASE <database-destino> TO ggreplicat;
```

Se `ggschema` for propriedade de outro usuário, conceda no mínimo `CREATE, USAGE` no schema, `EXECUTE` nas funções e `SELECT, INSERT, UPDATE, DELETE` nas tabelas desse schema ao usuário GoldenGate.

Antes de iniciar o Replicat, confirme que schemas, tabelas, tipos, collation, sequences e chaves sejam compatíveis com a origem.

## 7. Criar o Deployment

1. Acesse **Oracle AI Database → GoldenGate → Deployments → Create deployment**.
2. Selecione:

```text
Deployment type:     Data replication
Technology:          PostgreSQL
Version:             build certificado para origem e destino
Private subnet:      <subnet-golden-gate>
Credential store:    OCI IAM ou GoldenGate
```

3. Aguarde o estado `Active`.
4. Registre a Console URL, os Ingress IPs e o endereço privado.

O acesso à console é HTTPS na porta `443`. Acesso público deve ser excepcional e protegido por regras restritivas.

## 8. Criar connections PostgreSQL

Em **GoldenGate → Connections → Create connection**, crie a connection de origem:

```text
Name:                          OGG-PG-SRC
Type:                          PostgreSQL Server ou OCI Database with PostgreSQL
Database host:                 <dns-ou-ip-postgresql-origem>
Port:                          5432 ou porta real
Database name:                 <database-origem>
Username:                      ggextract
Database user password secret: OGG-PG-SRC-PASSWORD
Security protocol:             Plain, TLS ou MTLS conforme o ambiente
Network connectivity:          Shared endpoint ou Dedicated endpoint
```

Repita para o destino:

```text
Name:                          OGG-PG-TGT
Type:                          PostgreSQL Server ou OCI Database with PostgreSQL
Database host:                 <dns-ou-ip-postgresql-destino>
Port:                          5432 ou porta real
Database name:                 <database-destino>
Username:                      ggreplicat
Database user password secret: OGG-PG-TGT-PASSWORD
```

Para TLS/MTLS, configure o protocolo e os certificados conforme a opção selecionada na connection. Não reutilize password secret como secret de certificado/wallet.

## 9. Atribuir e testar connections

1. Acesse **GoldenGate → Deployments → `<deployment>` → Assigned connections**.
2. Atribua `OGG-PG-SRC` e `OGG-PG-TGT`.
3. Nas ações de cada connection, clique em **Test connection**.

O teste deve confirmar:

```text
Network-level connectivity: host e porta alcançáveis
Application-level connectivity: usuário, senha, database e SSL válidos
```

### Diagnóstico de falha de conexão

Verifique nesta ordem:

1. Host/DNS, porta e database da connection.
2. PostgreSQL escutando na porta configurada: `ss -lnt | grep 5432`.
3. `listen_addresses` permite conexões remotas.
4. `pg_hba.conf` permite o usuário GoldenGate a partir do Ingress IP/subnet correta.
5. Firewall do host, NSG/Security List e rota de retorno permitem TCP 5432.
6. Se TLS estiver habilitado, certificado, CA e modo SSL correspondem à configuração do PostgreSQL.

No servidor PostgreSQL, o DBA pode validar conexões e regras com:

```sql
SELECT usename, datname, client_addr, state
FROM pg_stat_activity
WHERE usename IN ('ggextract', 'ggreplicat');
```

## 10. Configurar a replicação

Abra **Launch console** no Deployment. A sequência geral é:

```text
Extract → Distribution Path → Replicat
```

### Origem

1. Crie um Extract para PostgreSQL e selecione `OGG-PG-SRC`.
2. Defina o ponto inicial: hora atual ou posição coordenada com a carga inicial.
3. Habilite `TRANDATA` para os schemas/tabelas necessários. Esta etapa prepara a publicação/captura lógica das tabelas.
4. Configure o trail local.
5. Crie e inicie o Distribution Path para o destino.

### Destino

1. Crie um Replicat e selecione `OGG-PG-TGT`.
2. Selecione o trail recebido.
3. Defina mapeamentos de schema e tabelas.
4. Configure checkpoint/heartbeat no schema `ggschema`, quando solicitado pelo assistente.
5. Inicie o Replicat.

## 11. Carga inicial

Escolha uma estratégia antes de iniciar a captura contínua:

- `pg_dump`/`pg_restore` para schema e dados;
- cópia coordenada por ferramenta externa;
- Extract de carga inicial;
- início em posição consistente após a carga.

Não inicie a replicação contínua sem coordenar o ponto de início do Extract com a cópia inicial. O objetivo é evitar lacunas ou duplicação de alterações entre a carga e o CDC.

Para bancos grandes, valide capacidade de WAL e a retenção: um slot de replicação inativo pode reter WAL e consumir espaço em disco.

## 12. Operação e validação

No Admin Client:

```text
INFO ALL
VIEW MESSAGES
```

Valide:

```text
Extract:  RUNNING
Replicat: RUNNING
Lag:      aceitável para o SLA
Erros:    ausentes em VIEW MESSAGES
```

No PostgreSQL origem, monitore slots e retenção de WAL:

```sql
SELECT slot_name, slot_type, active, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots;
```

Execute testes controlados de `INSERT`, `UPDATE`, `DELETE` e, se aplicável, `TRUNCATE`, e valide as alterações no destino.

## Troubleshooting rápido

| Sintoma | Causa provável | Ação |
|---|---|---|
| Timeout de conexão | Porta, firewall, NSG ou rota | Validar TCP 5432 e Ingress IPs |
| `no pg_hba.conf entry` | Regra `pg_hba.conf` ausente ou CIDR incorreto | Adicionar regra restritiva e recarregar configuração |
| `wal_level` não é `logical` | Logical replication não habilitada | Ajustar `postgresql.conf` e reiniciar PostgreSQL |
| Sem slot/sender disponível | `max_replication_slots` ou `max_wal_senders` baixo | Dimensionar parâmetros e reiniciar serviço |
| Falha em `ADD TRANDATA` | Privilégio insuficiente | DBA habilita a etapa ou concede `SUPERUSER` temporariamente |
| WAL cresce continuamente | Slot inativo ou Extract parado | Corrigir Extract/slot; monitorar espaço antes de remover recursos |
| `unable to access secrets using resource principal` | Dynamic Group ou policy ausente | Revisar `read secret-bundles` |
| Falha TLS | CA, certificado ou SSL mode incompatível | Conferir detalhes TLS/MTLS da connection e do servidor |

## Referências

- [OCI GoldenGate — What's supported](https://docs.oracle.com/en/cloud/paas/goldengate-service/ocigg/whats-supported.html)
- [Prepare PostgreSQL for Oracle GoldenGate](https://docs.oracle.com/en/middleware/goldengate/core/21.3/gghdb/preparing-database-oracle-goldengate-postgresql.html)
- [Connect to OCI Database with PostgreSQL](https://docs.oracle.com/en/cloud/paas/goldengate-service/yoyip/)
- [OCI GoldenGate Policies](https://docs.oracle.com/en/cloud/paas/goldengate-service/ocigg/oracle-cloud-infrastructure-goldengate-policies.html)
