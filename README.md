# CEOM — Monitoramento ambiental

Sensores ESP32 enviam temperatura, umidade e CO2 de cada sala; o backend valida
contra os limites, registra alertas e notifica o Telegram; o frontend mostra
dashboard, gráficos e histórico.

| Pasta | O que é |
| --- | --- |
| [`backend/`](backend/) | API Node.js (Express 5 + Prisma 7) — detalhes em [`backend/CLAUDE.md`](backend/CLAUDE.md) |
| [`frontend/`](frontend/) | SPA React 19 + Vite — detalhes em [`frontend/CLAUDE.md`](frontend/CLAUDE.md) |
| `compose.yml` | stack base: postgres, mongo, migrate, backend, frontend |
| `compose.dev.yml` | override de desenvolvimento: código montado + hot reload |
| `compose.external.yml` | override para usar bancos externos no lugar dos contêineres |

Os dois bancos são obrigatórios: **PostgreSQL** guarda o domínio (salas,
sensores, tenants, usuários) e **MongoDB** guarda as leituras e os alertas.

## Pré-requisitos

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (ou Docker
  Engine + Compose **v2.24+**, necessário para o `!override`/`!reset` dos overrides).
- Não precisa de Node instalado na máquina — tudo roda nos contêineres.

## Configuração inicial (uma vez)

```bash
# 1. Variáveis do Compose (quais arquivos usar e, se for o caso, as URLs dos bancos)
cp .env.example .env

# 2. Segredos da aplicação
cp backend/.env.example backend/.env
```

No `backend/.env`, preencha pelo menos:

| Variável | Para quê |
| --- | --- |
| `JWT_SECRET` | assinatura dos tokens de login — qualquer string longa |
| `TENANT_ADMIN_PASSWORD` | senha do `admin@<tenant>` criado pelo seed |
| `TELEGRAM_BOT_TOKEN` | opcional; sem ele os alertas são gravados, mas não enviados |

> `DATABASE_URL` e `MONGODB_URL` do `backend/.env` são **ignoradas** pelo
> Docker — o compose sempre as sobrescreve. Para bancos externos, elas vão no
> `.env` da **raiz** (opção B abaixo).

---

## Opção A — Bancos locais no Docker (recomendado para dev)

Sobe Postgres 17 e Mongo 7 em contêineres, junto com a aplicação. É o padrão do
`.env.example`:

```dotenv
# .env (raiz)
COMPOSE_FILE=compose.yml:compose.dev.yml
```

### Passo a passo

```bash
# 1. Build e subida de tudo (a primeira vez demora alguns minutos)
docker compose up --build

# 2. Em outro terminal, crie o primeiro tenant + admin (só na primeira vez)
docker compose exec backend npm run seed:bootstrap
```

O seed imprime o login (`admin@ceom` com a senha de `TENANT_ADMIN_PASSWORD`) e a
**api key** do tenant, usada pelo ESP32/simulador no header `x-api-key`.

### Endereços

| Serviço | URL |
| --- | --- |
| Frontend (Vite, hot reload) | http://localhost:3000 |
| API (nodemon, hot reload) | http://localhost:6767/api — health em `/api/health` |
| Postgres | `postgresql://ceom:ceom@localhost:5432/ceom` |
| Mongo | `mongodb://localhost:27017` (banco `ceom`) |

Os bancos ficam expostos para você abrir no DBeaver, psql, MongoDB Compass etc.
Se já tiver Postgres/Mongo instalado na máquina ocupando a porta, troque no
`.env` da raiz: `POSTGRES_PORT=5433`, `MONGO_PORT=27018`.

Os dados ficam em volumes do Docker e sobrevivem a `down`/`up`. Para **zerar os
bancos**: `docker compose down -v`.

---

## Opção B — Bancos externos

Usa um Postgres e um Mongo que já existem (Supabase, MongoDB Atlas, servidor da
UFFS, ou instalados na sua máquina). Os contêineres `postgres` e `mongo` **não**
sobem.

### Passo a passo

1. No `.env` da raiz, inclua o `compose.external.yml` e informe as URLs:

   ```dotenv
   COMPOSE_FILE=compose.yml:compose.dev.yml:compose.external.yml

   DATABASE_URL="postgresql://usuario:senha@host:5432/postgres?sslmode=require"
   MONGODB_URL="mongodb+srv://usuario:senha@cluster0.xxxxx.mongodb.net/?retryWrites=true&w=majority"
   MONGODB_DB=ceom
   ```

2. Suba a aplicação — as migrations rodam no banco externo antes do backend:

   ```bash
   docker compose up --build
   ```

3. Se o banco for novo (sem nenhum tenant), crie o primeiro admin:

   ```bash
   docker compose exec backend npm run seed:bootstrap
   ```

Os endereços do frontend e da API são os mesmos da opção A.

### Cuidados com bancos externos

- **Banco na sua própria máquina** (fora do Docker): use `host.docker.internal`
  no lugar de `localhost` — dentro do contêiner, `localhost` é o próprio contêiner.
  Ex.: `postgresql://user:senha@host.docker.internal:5432/ceom`.
- **Supabase**: use a URL do **pooler** (`*.pooler.supabase.com`), não a conexão direta.
- **MongoDB Atlas**: libere o seu IP em *Network Access*. Mongo 7.0+ é o
  recomendado (time series e retenção dependem dele).
- **Banco compartilhado com o time**: as migrations rodam a cada `up`. Elas são
  idempotentes, mas mudanças de schema que você ainda está testando vão direto
  para o banco de todos — prefira a opção A para isso.

---

## Comandos do dia a dia

Com o `COMPOSE_FILE` no `.env` da raiz, não precisa repetir os `-f`.

```bash
docker compose up --build          # sobe tudo (Ctrl+C para parar)
docker compose up -d               # sobe em segundo plano
docker compose logs -f backend     # acompanha os logs de um serviço
docker compose ps                  # status dos contêineres
docker compose down                # para tudo (mantém os dados)
docker compose down -v             # para e apaga os bancos locais

docker compose exec backend npm run tenants:migrate     # tenant-migrations em todos os tenants
docker compose exec backend npx prisma migrate dev --name minha_migration
docker compose exec postgres psql -U ceom -d ceom       # psql no banco local (opção A)
docker compose exec mongo mongosh ceom                  # mongosh no banco local (opção A)
```

Simulador do ESP32 (api key vem da saída do seed):

```bash
docker compose exec -e API_KEY=<api key> backend npm run simulate
```

### Mudou `package.json` ou um `schema.prisma`?

`node_modules` e os clients do Prisma vivem dentro da imagem, não na sua pasta.
Depois de adicionar dependência ou mudar schema, reconstrua e recrie esses volumes:

```bash
docker compose up --build -V
```

---

## Produção local (imagens como na pipeline)

Para testar as imagens finais — frontend estático servido pelo nginx, backend
sem root e sem hot reload — use só o `compose.yml`:

```bash
COMPOSE_FILE=compose.yml docker compose up --build
```

| Serviço | URL |
| --- | --- |
| Frontend (nginx) | http://localhost:8080 — `/api/*` é repassado ao backend |
| API | http://localhost:6767/api |

Para bancos externos neste modo: `COMPOSE_FILE=compose.yml:compose.external.yml`.

---

## Problemas comuns

| Sintoma | Causa / solução |
| --- | --- |
| `port is already allocated` | Porta em uso na máquina. Troque `POSTGRES_PORT`/`MONGO_PORT` no `.env`, ou pare o serviço local. |
| `Cannot find module ...` após `git pull` | Dependência nova. Rode `docker compose up --build -V`. |
| Backend reinicia em loop com erro de conexão | URL do banco errada ou inacessível. Veja `docker compose logs backend`; em banco externo, confira IP liberado e `host.docker.internal`. |
| `migrate` termina com erro e o backend não sobe | Veja `docker compose logs migrate` — geralmente banco inacessível ou migration com conflito. |
| Login responde 401 | Não existe usuário ainda: rode o `seed:bootstrap`. O login é `usuario@tenant` (ex.: `admin@ceom`). |
| Frontend abre mas não carrega dados | Confira se a API responde em http://localhost:6767/api/health. |

## Logs (padrão UFFS)

Todos os serviços do compose declaram as labels `uffs.log.APP_NAME` e
`uffs.log.APP_VERSION` e usam o driver `json-file` (`max-size: 10m`,
`max-file: 3`), conforme o guia de infraestrutura. Ao adicionar um serviço novo,
reutilize a âncora `*logging` do `compose.yml` e declare as duas labels.
