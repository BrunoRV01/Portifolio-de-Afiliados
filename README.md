# Portifolio-de-Afiliados
Projeto que cria um portifolio de afiliados de produtos cadastrados em sites confiaveis: Amazon, shoppe. Sistema totalmente criado por AI (Claude) e feita as devidas edições.
# Backend v2 — Plataforma de Afiliados

Reescrita com arquitetura em camadas, banco assíncrono, migrações
versionadas, revogação/rotação de refresh tokens, rate limiting,
logging estruturado, testes automatizados e Docker.

## O que mudou em relação à v1

| Aspecto              | v1                                  | v2                                                             |
|-----------------------|--------------------------------------|-----------------------------------------------------------------|
| Arquitetura           | rotas → modelo direto                | rotas → **service** (regra de negócio) → **repository** (dados) |
| Banco                 | SQLAlchemy síncrono                  | SQLAlchemy 2.0 **assíncrono** (asyncpg/aiosqlite)               |
| Schema do banco       | `Base.metadata.create_all` no boot   | **Alembic** (migrações versionadas)                             |
| Refresh token         | JWT sem estado, nunca revogável      | persistido no banco, **rotação a cada uso + revogação**         |
| Senha                 | só tamanho mínimo                    | política de força (maiúscula/minúscula/número)                  |
| Erros                 | `HTTPException` espalhada nas rotas  | exceções de domínio + handler central → JSON padronizado        |
| Login/registro        | sem limite de tentativas             | **rate limiting** (slowapi)                                     |
| Logs                  | prints/logs default do uvicorn       | **logging estruturado em JSON** com request-id por requisição   |
| Listagem de produtos  | retorna tudo de uma vez              | **paginada** (`limit`/`offset` + total)                         |
| Atualização de produto| não existia                          | `PATCH` parcial (ex: atualizar só `clicks`/`sales_trend`)        |
| Testes                | nenhum                               | suíte pytest cobrindo auth e produtos                            |
| Deploy                | manual                               | Dockerfile multi-stage + docker-compose (API + Postgres)         |

## Arquitetura

```
app/
├── core/
│   ├── config.py            # Settings validadas (pydantic-settings)
│   ├── security.py          # hash, política de senha, JWT (com jti)
│   ├── logging.py           # logging JSON + request_id
│   ├── exceptions.py        # exceções de domínio (sem depender de HTTP)
│   ├── exception_handlers.py# traduz exceções -> respostas HTTP padronizadas
│   └── rate_limit.py        # instância do slowapi
├── db/
│   ├── base.py               # Base declarativa + TimestampMixin
│   └── session.py            # engine/sessão assíncrona
├── models/                   # SQLAlchemy ORM (User, Product, RefreshToken)
├── schemas/                  # Pydantic (validação de entrada/saída)
├── repositories/              # SÓ queries — sem regra de negócio
│   ├── user_repository.py
│   ├── product_repository.py
│   └── refresh_token_repository.py
├── services/                  # Regra de negócio, orquestra repositórios
│   ├── auth_service.py        # registro, login, refresh, logout
│   └── product_service.py     # CRUD + agregações do dashboard
├── api/
│   ├── deps.py                 # get_current_user, injeção de serviços
│   └── v1/routes/               # auth.py, users.py, products.py
├── middleware.py               # request-id + log de cada requisição
└── main.py                     # monta app, middlewares, handlers, rotas

alembic/                        # migrações versionadas
tests/                          # pytest + httpx.AsyncClient
```

### Por que separar `services` de `repositories`?

- **Repository**: só sabe fazer queries (`SELECT`, `INSERT`, agregações). Não
  sabe se um e-mail duplicado é um erro — só busca e retorna dados.
- **Service**: sabe as regras (e-mail duplicado é `ConflictError`, senha
  fraca é rejeitada, refresh token revogado derruba a sessão inteira).
  Isso deixa a regra de negócio testável sem precisar montar requisições
  HTTP, e trocar o banco de dados no futuro não exigiria tocar em regra
  de negócio nenhuma.

## Segurança implementada

1. **Senha**: bcrypt + política de força mínima (8+ caracteres, maiúscula,
   minúscula, número) — `app/core/security.py`.
2. **Refresh token com rotação**: a cada `/auth/refresh`, o token antigo é
   revogado e um novo é emitido (`jti` marcado como `revoked=True` no
   banco). Se um token já revogado for reapresentado — sinal clássico de
   token vazado sendo reusado por um atacante — a API revoga **toda** a
   sessão do usuário automaticamente.
3. **Rate limiting**: `/auth/login` (5/min) e `/auth/register` (3/min) por
   IP, configurável via `.env`.
4. **Isolamento por dono**: toda query de produto filtra por `owner_id`;
   tentar acessar produto de outro usuário retorna 404 (não 403, para não
   revelar que o recurso existe).
5. **SECRET_KEY validada**: a aplicação recusa subir em `ENV=production`
   com a chave padrão de desenvolvimento.

## Como rodar localmente (SQLite, sem Docker)

```bash
cd backend
python3 -m venv venv && source venv/bin/activate
pip install -r requirements-dev.txt

cp .env.example .env
python3 -c "import secrets; print(secrets.token_hex(32))"   # cole em SECRET_KEY

alembic upgrade head        # cria as tabelas via migração
uvicorn app.main:app --reload
```

Docs interativas: `http://localhost:8000/docs`
Healthcheck (com verificação do banco): `http://localhost:8000/api/health`

## Como rodar com Docker (API + PostgreSQL + Frontend)

Assumindo que as pastas `backend/` e `frontend/` estão lado a lado (como
nos dois pacotes entregues):
