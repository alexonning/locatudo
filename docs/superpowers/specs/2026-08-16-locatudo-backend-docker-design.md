# LocaTudo — Backend, banco e frontend em Docker

**Data:** 2026-08-16
**Status:** aprovado, pronto para plano de implementação

## Contexto

O repositório contém hoje apenas um protótipo visual: `LocaTudo.dc.html` (1023 linhas) sobre um
runtime proprietário (`support.js`, `image-slot.js`), com dados mockados em memória. O protótipo
descreve um marketplace de aluguel de equipamentos de construção — LocaTudo Constru&CIA, atuando em
Capanema e Realeza, no sudoeste do Paraná — com quatro telas: Home, Resultados, Detalhe e Resumo.

Não existe backend, banco de dados nem configuração de containers.

## Objetivo

Criar a estrutura completa do projeto — backend NestJS, banco PostgreSQL, cache Redis e frontend
React migrado do protótipo — com todos os serviços em Docker, entregando um sistema que sobe com
`docker compose up` e cobre catálogo, disponibilidade, reservas, autenticação e administração.

## Decisões tomadas

| Decisão | Escolha |
|---|---|
| Stack do backend | NestJS + TypeScript + Prisma + PostgreSQL |
| Organização | Monólito modular por domínio (`src/modules/<dominio>`) |
| Frontend | Migração do protótipo para React + Vite (SPA) |
| Cache | Redis em read-through, com invalidação por versionamento de namespace |
| Pagamento | Apenas registro de método e status; sem gateway |
| Imagens | Upload para volume Docker, servido estaticamente pelo backend |
| Docker | `docker-compose.yml` (dev, hot reload) + `docker-compose.prod.yml` (multi-stage) |

Duas alternativas foram consideradas e descartadas para a organização do backend: arquitetura
hexagonal (triplica os arquivos sem benefício claro para um CRUD de marketplace regional) e camadas
globais `controllers/services/repositories` (vai contra o grão do NestJS e concentra crescimento em
pastas-depósito). Da hexagonal foi emprestado um ponto: as duas regras com lógica real — preço e
disponibilidade — ficam em classes puras, sem Prisma, testáveis sem banco.

## Arquitetura

### Serviços Docker

| Serviço | Dev | Produção |
|---|---|---|
| `db` | Postgres 16, volume `pgdata`, porta 5432 exposta | mesma imagem, porta não exposta |
| `redis` | Redis 7-alpine, `appendonly yes`, volume `redisdata` | idem, porta não exposta |
| `backend` | `node:20-alpine`, bind mount, `nest start --watch`, porta 3000 | multi-stage → `node dist/main` |
| `frontend` | `node:20-alpine`, Vite dev server (HMR), porta 5173, proxy `/api` → `backend:3000` | multi-stage → Nginx estático + `proxy_pass /api` |

`db` e `redis` têm `healthcheck` (`pg_isready` / `redis-cli ping`) e o `backend` depende deles com
`condition: service_healthy` — sem isso o Nest sobe antes do Postgres aceitar conexões e morre no
primeiro `up`. Migrations rodam no entrypoint do backend: `prisma migrate dev` em desenvolvimento,
`prisma migrate deploy` em produção. Uploads em volume nomeado montado em `/app/uploads`.

Um serviço adicional `db-test` (Postgres efêmero, sem volume) serve aos testes e2e.

### Estrutura de diretórios

```
locatudo/
├── docker-compose.yml            # dev
├── docker-compose.prod.yml       # produção
├── .env.example                  # POSTGRES_*, DATABASE_URL, REDIS_URL, JWT_SECRET, PORT
├── backend/
│   ├── Dockerfile                # multi-stage (prod)
│   ├── Dockerfile.dev
│   ├── docker-entrypoint.sh
│   ├── prisma/
│   │   ├── schema.prisma
│   │   ├── migrations/
│   │   └── seed.ts
│   ├── test/                     # e2e (supertest)
│   └── src/
│       ├── main.ts
│       ├── app.module.ts
│       ├── config/               # validação de env
│       ├── prisma/               # PrismaModule + PrismaService
│       ├── cache/                # CacheModule, CacheService, CacheInvalidationService
│       ├── common/               # guards, filtros, decorators, pipes, DTOs de paginação
│       └── modules/
│           ├── auth/
│           ├── users/
│           ├── categories/
│           ├── units/
│           ├── equipments/       # inclui admin.controller.ts
│           ├── availability/     # availability.service.ts (puro)
│           ├── reservations/     # pricing.service.ts (puro)
│           └── uploads/
├── frontend/
│   ├── Dockerfile
│   ├── Dockerfile.dev
│   ├── nginx.conf
│   └── src/{pages,components,api,hooks,styles}/
├── legacy/                       # LocaTudo.dc.html, support.js, image-slot.js
└── docs/superpowers/specs/
```

O protótipo e seus runtimes são movidos para `legacy/` em vez de apagados: são a referência visual da
migração das telas.

## Camada de cache

### Estratégia

Leitura em **read-through**: o endpoint consulta o Redis e, apenas em caso de miss, vai ao Postgres,
devolve a resposta e popula a chave. Na prática quase toda leitura é servida pelo Redis; o banco é
fallback, não caminho normal.

A alternativa literal — ler exclusivamente do Redis — foi descartada porque um miss (primeira subida,
chave expirada, eviction por `maxmemory`, restart sem AOF íntegro) devolveria catálogo vazio com o
banco cheio. Complementarmente, um *warm-up* no boot do backend popula categorias e a listagem padrão
de equipamentos, de modo que o cache já sobe quente.

**O Postgres é a fonte da verdade. Nada é escrito apenas no Redis.**

### O que é cacheado

| Chave | Conteúdo | TTL |
|---|---|---|
| `cat:v{n}:all` | categorias com contagem | 1 h |
| `eq:v{n}:list:{hash-filtros}` | resultado paginado de busca/filtro | 10 min |
| `eq:v{n}:slug:{slug}` | detalhe do equipamento | 10 min |
| `avail:v{n}:{equipId}:{from}:{to}` | calendário de disponibilidade | 60 s |

Rotas autenticadas — reservas do usuário, `/auth/me`, área administrativa — **não são cacheadas**: o
ganho é irrelevante e o risco de servir dado de um cliente a outro é real.

### Invalidação por versionamento de namespace

As chaves de listagem são hasheadas pelos filtros da query, então não há como enumerar quais apagar
após uma escrita; e varrer com `KEYS *` bloqueia o Redis em produção. A solução é um contador por
namespace embutido na chave (`ver:eq`, `ver:cat`, `ver:avail:{equipId}`): qualquer escrita executa
`INCR` no contador, as chaves antigas ficam órfãs e expiram pelo TTL, e a próxima leitura monta a
chave nova a partir do banco. Invalidação O(1), sem varredura.

A invalidação é disparada **explicitamente** por um `CacheInvalidationService` chamado pelos services
após cada escrita — não por decorator implícito, onde esse tipo de bug se esconde:

```
admin cria/edita/remove equipamento   → INCR ver:eq  e  ver:avail:{id}
admin cria/edita/remove categoria     → INCR ver:cat e  ver:eq      (o card exibe contagem)
reserva criada / cancelada / bloqueio → INCR ver:avail:{equipId} e ver:eq  (muda "disponível")
seed                                  → INCR em todos os namespaces
```

O `INCR` ocorre após o commit da transação de escrita: se o commit falhar, nada é invalidado.

### Degradação

Todo acesso ao cache é envolvido em try/catch. Miss, timeout ou conexão recusada caem no Postgres,
registram warning e seguem. Cache é otimização, nunca dependência crítica.

## Modelo de dados

```
User          id, nome, email @unique, senhaHash, telefone, documento, role(CLIENTE|ADMIN)
Unit          id, nome, cidade, uf, telefone
Category      id, slug @unique, nome, mono, ordem
Equipment     id, slug @unique, nome, categoryId, unitId, marca, modelo,
              descricao, specs String[], quantidade @default(1),
              precoHora Decimal?, precoDia Decimal?, hasHour, hasDay,
              ativo, avaliacaoMedia, avaliacaoQtd
EquipmentImage  id, equipmentId, path, alt, ordem
EquipmentBlock  id, equipmentId, dataInicio, dataFim, motivo
Reservation   id, codigo @unique, userId, equipmentId,
              dataInicio, dataFim, billingMode(HORA|DIA), dias,
              precoUnitario, subtotal, taxa, total,
              status(PENDENTE|CONFIRMADA|EM_ANDAMENTO|DEVOLVIDA|CANCELADA),
              paymentMethod(PIX|CARTAO|DINHEIRO), paymentStatus(PENDENTE|PAGO|ESTORNADO),
              nomeCliente, telefone, documento, email, cep, endereco, numero, cidade
```

**`Equipment.quantidade`** dá significado real ao status `parcial` do protótipo: num dia, `reservado
== 0` é disponível, `0 < reservado < quantidade` é parcial e `reservado == quantidade` é reservado.
O catálogo tem vários exemplares do mesmo item (módulos de andaime, por exemplo), então o campo
reflete a operação real.

**Snapshot do cliente na `Reservation`** (além de `userId`): o cliente pode alterar telefone ou
endereço depois, e a reserva precisa preservar os dados vigentes na contratação — é documento de
locação, não visão do cadastro.

Valores monetários em `Decimal(10,2)`, nunca `Float`. Índices em
`Reservation(equipmentId, dataInicio, dataFim)` — a consulta de sobreposição é a mais quente do
sistema —, `Equipment(slug)` e `Equipment(categoryId, ativo)`.

O seed carrega as 8 categorias, as 2 unidades e os 5 equipamentos do protótipo, além de um usuário
admin. As imagens do seed apontam para arquivos estáticos versionados no repositório, de modo que o
catálogo já aparece ilustrado na fase 1 — o upload pela administração só existe a partir da fase 3.

Todas as tabelas, incluindo `EquipmentBlock` e `Reservation`, são criadas na migration inicial da
fase 1. O que muda por fase são os módulos que as consomem, não o schema.

## API

```
público    GET  /api/categories
           GET  /api/units
           GET  /api/equipments?q&categoria&cidade&precoMax&apenasDisponiveis&ordenar&page&limit
           GET  /api/equipments/:slug
           GET  /api/equipments/:slug/availability?from&to
           POST /api/reservations/quote

auth       POST /api/auth/register · /login · /refresh · /logout
           GET  /api/auth/me

cliente    POST /api/reservations
           GET  /api/reservations
           GET  /api/reservations/:codigo
           POST /api/reservations/:codigo/cancel

admin      CRUD /api/admin/categories · /api/admin/units · /api/admin/equipments
           POST/DELETE /api/admin/equipments/:id/images
           POST/DELETE /api/admin/equipments/:id/blocks
           GET   /api/admin/reservations?status
           PATCH /api/admin/reservations/:codigo
```

O filtro `apenasDisponiveis` de `GET /api/equipments` depende do `AvailabilityService` e por isso só
entra na fase 2; na fase 1 o parâmetro é aceito e ignorado, e a interface não o oferece.

JWT de acesso com validade de 15 minutos e refresh de 7 dias; o `jti` do refresh fica no Redis com
TTL igual à expiração, o que permite logout e revogação sem tabela adicional. Senhas com argon2.
Validação de entrada com `class-validator` e `ValidationPipe` global (`whitelist: true`). Swagger em
`/api/docs`, gerado a partir dos DTOs.

## Regras de negócio

### PricingService (puro)

`dias = (dataFim − dataInicio) + 1`, inclusivo. Modo hora: `precoHora × dias × 8`. Modo dia:
`precoDia × dias`. `total = subtotal + taxa`, com taxa de serviço de R$ 20.

O frontend nunca envia preço: chama `POST /reservations/quote` para exibir, e o servidor recalcula na
criação da reserva. Preço vindo do cliente é preço negociável por quem abre o DevTools.

### AvailabilityService (puro)

Recebe reservas e bloqueios já carregados e, para cada dia do intervalo, conta unidades ocupadas por
reservas em `PENDENTE|CONFIRMADA|EM_ANDAMENTO` que se sobrepõem, soma bloqueios de manutenção e
classifica o dia em disponível / parcial / reservado / bloqueado conforme a `quantidade`.

### Concorrência na criação de reserva

Dois clientes podem fechar a última unidade no mesmo instante. A criação roda dentro de
`$transaction` com `SELECT ... FOR UPDATE` na linha do equipamento: serializa apenas as reservas
daquele item, revalida a disponibilidade dentro do lock e só então insere. Sem isso o overbooking
aparece em produção e é praticamente irreproduzível depois.

O `codigo` da reserva vem de uma sequence do Postgres (`LT-00042`), não de `Math.random()` como no
protótipo, que colide.

## Tratamento de erros

`AllExceptionsFilter` global com payload uniforme (`statusCode`, `error`, `message[]`, `path`,
`timestamp`), mapeando erros do Prisma (`P2002` → 409, `P2025` → 404) e exceções de domínio
(`EquipmentUnavailableException` → 409 com os dias em conflito). Logs estruturados com `nestjs-pino`
e request-id por requisição.

## Testes

Desenvolvimento orientado a testes: teste antes da implementação, em cada fase.

**Unitários (Jest, sem banco)**
- `PricingService` em tabela de casos: 1 dia, N dias, item apenas com diária, modo hora sem `precoHora`.
- `AvailabilityService`: bordas de sobreposição, parcial com quantidade > 1, bloqueio de manutenção.
- `CacheService`: montagem de chave e incremento de versão.

**E2E (supertest contra o serviço `db-test`)**
- Fluxo completo: busca → detalhe → quote → reserva → cancelamento.
- Autorização: cliente recebe 403 nas rotas `/admin`.
- Invalidação de cache: criar reserva e verificar que `GET availability` reflete imediatamente.
- Concorrência: duas reservas simultâneas da última unidade devem produzir exatamente um 201 e um 409.
- Degradação: com o Redis indisponível, as rotas de leitura continuam respondendo pelo Postgres.

## Fases de implementação

| Fase | Escopo | Estado ao final |
|---|---|---|
| 1 | Compose (`db`, `redis`, `backend`, `frontend`, `db-test`), Prisma + migrations + seed, módulos `categories`/`units`/`equipments` (leitura), cache read-through com warm-up, React+Vite com Home e Resultados | `docker compose up` entrega catálogo navegável servido pelo Redis |
| 2 | `availability`, `pricing`, `reservations` (quote, criar, cancelar) com lock transacional, invalidação de cache, telas Detalhe e Resumo | fluxo completo de reserva |
| 3 | `auth` (JWT + refresh no Redis, argon2), `RolesGuard`, CRUD administrativo, upload de imagens no volume, telas de login e minhas reservas | marketplace administrável |

## Fora de escopo

- Gateway de pagamento real (Pix com QR Code, cartão). A reserva registra método e status; a
  confirmação é manual pelo admin. A troca por um gateway não exige mudança de modelo de dados.
- Avaliações de clientes: `avaliacaoMedia` e `avaliacaoQtd` são campos denormalizados alimentados
  pelo seed, sem fluxo de escrita.
- Notificações por e-mail ou WhatsApp.
- Storage de objetos em nuvem (S3/MinIO): imagens ficam em volume Docker.
