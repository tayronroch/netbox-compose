# netbox-compose

NetBox 4.7 com a identidade visual da Direct, pronto para deploy no [Coolify](https://coolify.io/).
Baseado na imagem oficial [`netboxcommunity/netbox`](https://github.com/netbox-community/netbox-docker):
o `infra/Dockerfile` só adiciona logos, favicon, o título "NetBox - Direct" e a pasta `configuration/`.

PostgreSQL e Redis ficam fora da stack (recursos já existentes no Coolify) e são configurados por URL.

## Estrutura

| Arquivo | Uso |
|---|---|
| `infra/docker-compose.yml` | Produção (Coolify): `netbox` (Granian) + `netbox-worker` (rqworker) |
| `infra/docker-compose.local.yml` | Desenvolvimento: adiciona PostgreSQL 18 e Valkey |
| `infra/Dockerfile` | Imagem com a identidade visual da Direct (build a partir da raiz) |
| `configuration/` | Configuração do NetBox, lida das variáveis de ambiente |
| `src/img/` | Logo e ícone da Direct (SVG) |
| `.env.production.example` | Modelo das variáveis do Coolify |
| `.env.local.example` | Modelo do ambiente local |

## Instalação

### Coolify (produção)

Passo a passo completo, incluindo preparo do banco e restore de backup, em **[docs/coolify.md](docs/coolify.md)**. Resumo:

1. No PostgreSQL do Coolify, crie o usuário e o banco `netbox`.
2. (Opcional) Restaure um backup em **Import Backups** com `pg_restore -U $POSTGRES_USER -d netbox --no-acl --exit-on-error`.
3. **New Resource → Docker Compose** apontando para este repositório, arquivo `infra/docker-compose.yml`,
   com **Connect to Predefined Network** marcado.
4. Preencha as variáveis conforme `.env.production.example`. As principais:

   ```env
   ALLOWED_HOSTS=netbox.suaempresa.com.br
   CSRF_TRUSTED_ORIGINS=https://netbox.suaempresa.com.br
   SECRET_KEY=            # openssl rand -base64 48
   API_TOKEN_PEPPER_1=    # openssl rand -hex 32 (ou o antigo, para manter os tokens de API)
   DATABASE_URL=postgres://netbox:SENHA@HOST_INTERNO:5432/netbox
   REDIS_URL=redis://default:SENHA@HOST_INTERNO:6379/0
   ```

5. Defina o domínio do serviço `netbox` como `https://netbox.suaempresa.com.br:8080` e faça o deploy.

### Local

```bash
cp .env.local.example .env.local
nb() { docker compose -f infra/docker-compose.yml -f infra/docker-compose.local.yml --env-file .env.local "$@"; }
nb up -d --build        # http://localhost:8080  (admin / admin)
nb down -v              # remove tudo, inclusive dados
```

### Usando o netbox-docker oficial

Se preferir o fluxo do projeto original (imagem pronta, `docker-compose.override.yml`, scripts de build e testes),
veja o README do netbox-docker em [docs/netbox-docker.md](docs/netbox-docker.md)
e o [repositório oficial](https://github.com/netbox-community/netbox-docker).

## Licença

Apache 2.0 — veja [LICENSE](LICENSE). Derivado do [netbox-docker](https://github.com/netbox-community/netbox-docker).
