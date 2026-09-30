# Deploy do NetBox no Coolify

Stack: `netbox` (Granian) + `netbox-worker` (rqworker), imagem própria (`infra/Dockerfile`) com
logos da Direct e `configuration/` embutidos. PostgreSQL e Redis são recursos já existentes no Coolify.

| Arquivo | Uso |
|---|---|
| `infra/docker-compose.yml` | Produção (Coolify) — só NetBox + worker |
| `infra/docker-compose.local.yml` | Local — adiciona PostgreSQL 18 e Valkey |
| `.env.production.example` | Modelo das variáveis do Coolify |
| `.env.local.example` | Modelo do ambiente local |
| `backup/` | Dumps do banco (ignorado pelo git) |

## 1. Preparar o PostgreSQL do Coolify

Versão **18.x** (o backup foi gerado pelo `pg_dump` 18.6). Como superusuário (`postgres`):

```sql
CREATE USER netbox WITH PASSWORD '...';
CREATE DATABASE netbox OWNER netbox;
\c netbox
CREATE EXTENSION IF NOT EXISTS ltree;   -- exigida pelo NetBox 4.7; o usuário netbox não pode criá-la
```

Tuning recomendado (VPS 16 vCPU / 16 GB compartilhada), em "Custom PostgreSQL Configuration":

```
shared_buffers = 4GB
effective_cache_size = 10GB
work_mem = 32MB
maintenance_work_mem = 1GB
random_page_cost = 1.1
effective_io_concurrency = 200
max_connections = 100
```

## 2. Restaurar o backup (antes do primeiro deploy do NetBox)

Use um dump no formato custom (`pg_dump -Fc`, arquivo `.dump`). Ele não inclui usuário nem banco,
por isso o passo 1 vem antes.

**Pelo Coolify:** no banco, abra **Import Backups**, troque o comando por

```
pg_restore -U $POSTGRES_USER -d netbox --no-acl --exit-on-error
```

e envie o arquivo `.dump`. O resultado esperado é `Import finished with exit code 0`.

**Por SSH no host** (container = UUID do banco no Coolify):

```bash
docker exec -i <container-postgres> pg_restore -U postgres -d netbox --no-acl --exit-on-error < netbox_AAAA-MM-DD.dump
```

Conferência, no terminal do banco: `psql -U "$POSTGRES_USER" -d netbox -c "SELECT count(*) FROM users_user;"`

O backup é de NetBox **4.7.x** — mesma versão da imagem (4.7.2), não há migração pendente.
Os usuários do backup continuam válidos, então mantenha `SKIP_SUPERUSER=true`.

Se o NetBox antigo tinha arquivos em media (imagens anexadas), copie o volume
`netbox-media-files` antigo para o volume de media novo.

## 3. Criar o recurso no Coolify

1. **New Resource → Docker Compose** apontando para este repositório, arquivo `infra/docker-compose.yml`.
2. Marque **Connect to Predefined Network** para alcançar o PostgreSQL e o Redis.
3. Em **Environment Variables**, preencha conforme `.env.production.example`.
   `DATABASE_URL`/`REDIS_URL` = URL interna exibida em cada banco no Coolify. No `DATABASE_URL`, troque
   usuário, senha e banco para `netbox` (a URL do Coolify vem com o `postgres`). Senha com `@`, `:`, `/` ou `#`
   precisa ser percent-encoded (`@` → `%40`).
4. No serviço `netbox`, defina o domínio como `https://netbox.seudominio.com.br:8080`
   (a porta indica ao Traefik onde o Granian escuta). Use o mesmo domínio em `ALLOWED_HOSTS` e `CSRF_TRUSTED_ORIGINS`.
5. Deploy. O primeiro start pode levar alguns minutos (healthcheck com `start_period` de 300s).

Gerar segredos: `openssl rand -base64 48` (SECRET_KEY) e `openssl rand -hex 32` (API_TOKEN_PEPPER_1).
Com backup restaurado, reutilize o `API_TOKEN_PEPPER_1` antigo, senão os tokens de API existentes deixam de funcionar.

## Desempenho

- `GRANIAN_WORKERS=8` × `GRANIAN_BLOCKING_THREADS=2` → até 16 requisições simultâneas (~3–4 GB de RAM).
- Conexões persistentes (`DB_CONN_MAX_AGE=300`) com `CONN_HEALTH_CHECKS` ativo.
- Redis único com bancos separados: `0` = filas, `1` = cache. Se o Redis for compartilhado, troque os números.
- Muitos webhooks/scripts? Duplique o serviço `netbox-worker` no compose.

## Ambiente local

```bash
cp .env.local.example .env.local
nb() { docker compose -f infra/docker-compose.yml -f infra/docker-compose.local.yml --env-file .env.local "$@"; }
nb up -d postgres redis
nb exec -T postgres pg_restore -U netbox -d netbox --no-acl /backup/netbox_AAAA-MM-DD.dump   # opcional
nb up -d --build        # http://localhost:8080
nb down -v              # remove tudo, inclusive dados
```

Sem restaurar backup, use `SKIP_SUPERUSER=false` no `.env.local` para criar o admin (`admin`/`admin`).
