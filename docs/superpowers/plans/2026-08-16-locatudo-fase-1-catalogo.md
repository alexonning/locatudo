# LocaTudo — Fase 1: Catálogo em Docker — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Entregar `docker compose up` que sobe Postgres, Redis, backend NestJS e frontend React, com o catálogo de equipamentos navegável (Home e Resultados) e servido pelo Redis em read-through.

**Architecture:** Monólito modular NestJS (`src/modules/<dominio>`) sobre Prisma/PostgreSQL, com uma camada de cache Redis read-through cujas chaves carregam um número de versão por namespace — invalidar é `INCR` no contador, nunca varredura de chaves. O frontend é uma SPA React+Vite que consome `/api` via proxy. Todos os serviços rodam em containers, com hot reload em desenvolvimento.

**Tech Stack:** NestJS 10 + TypeScript, Prisma + PostgreSQL 16, Redis 7 (ioredis), React 18 + Vite 5, Jest + supertest, Vitest + Testing Library, Docker Compose.

Spec de referência: `docs/superpowers/specs/2026-08-16-locatudo-backend-docker-design.md`.

## Global Constraints

- Node 20 LTS dentro dos containers (`node:20-alpine`). Nenhum comando de desenvolvimento roda fora do Docker.
- Todas as rotas HTTP do backend sob o prefixo `/api` (`app.setGlobalPrefix('api')`).
- Valores monetários em `Decimal(10, 2)` no Prisma. Nunca `Float`.
- Textos de interface e mensagens de erro voltadas ao usuário em português do Brasil.
- Paleta e tipografia herdadas do protótipo: navy `#14205A`, navy2 `#1D2B63`, amarelo `#FFCB05`, vermelho `#E4032E`, texto `#14203D`, cinza `#6B7280`, fundo `#F5F5F1`, branco `#FFFFFF`; títulos em Poppins, corpo em Plus Jakarta Sans.
- Acesso ao Redis nunca pode derrubar uma rota: toda operação de cache é envolvida em try/catch e cai no Postgres em caso de falha.
- Rotas autenticadas jamais entram no cache (relevante a partir da fase 3, mas a regra vale desde já).
- Nenhum segredo commitado. `.env` fica no `.gitignore`; só `.env.example` é versionado.

## Estrutura de arquivos da fase

**Raiz**
- `docker-compose.yml` — serviços de desenvolvimento: `db`, `db-test`, `redis`, `backend`, `frontend`
- `docker-compose.prod.yml` — build de produção
- `.env.example`, `.gitignore`, `README.md`
- `legacy/` — protótipo movido (`LocaTudo.dc.html`, `support.js`, `image-slot.js`, `uploads/`)

**Backend** (`backend/`)
- `Dockerfile.dev`, `Dockerfile`, `docker-entrypoint.sh`
- `prisma/schema.prisma` — todos os modelos, criados já na migration inicial
- `prisma/seed.ts` — categorias, unidades, equipamentos, admin
- `prisma/seed-assets/` — imagens de exemplo copiadas para o volume de uploads
- `src/main.ts`, `src/app.module.ts`
- `src/config/env.validation.ts` — validação das variáveis de ambiente no boot
- `src/prisma/{prisma.module.ts,prisma.service.ts}`
- `src/cache/{cache.module.ts,cache.service.ts,cache.constants.ts}` — read-through e versionamento
- `src/common/filters/all-exceptions.filter.ts` — payload de erro uniforme
- `src/common/dto/pagination.dto.ts`
- `src/health/{health.module.ts,health.controller.ts}`
- `src/modules/categories/`, `src/modules/units/` — catálogo de referência
- `src/modules/equipments/` — listagem com filtros e detalhe por slug
- `src/warmup/warmup.service.ts` — popula o cache no boot
- `test/` — e2e com supertest contra `db-test`

**Frontend** (`frontend/`)
- `Dockerfile.dev`, `Dockerfile`, `nginx.conf`, `vite.config.ts`
- `src/api/{client.ts,equipments.ts,categories.ts}` — acesso HTTP e construção de querystring
- `src/pages/{HomePage.tsx,ResultsPage.tsx}`
- `src/components/{Header.tsx,EquipmentCard.tsx,CategoryCard.tsx,FilterPanel.tsx}`
- `src/styles/tokens.css`, `src/styles/global.css`
- `src/App.tsx`, `src/main.tsx`

A separação por responsabilidade mantém cada arquivo pequeno: `cache.service.ts` não conhece domínio, os services de domínio não conhecem Redis (recebem `CacheService` injetado), e os componentes de UI não constroem URL (isso vive em `src/api/`).

---

### Task 1: Esqueleto do repositório, backend NestJS e Docker

Sobe `db` + `backend` com hot reload e um health check respondendo. Sem banco ainda.

**Files:**
- Create: `.gitignore`, `.env.example`, `docker-compose.yml`
- Create: `backend/package.json`, `backend/tsconfig.json`, `backend/nest-cli.json`, `backend/Dockerfile.dev`, `backend/docker-entrypoint.sh`
- Create: `backend/src/main.ts`, `backend/src/app.module.ts`, `backend/src/config/env.validation.ts`
- Create: `backend/src/common/filters/all-exceptions.filter.ts`
- Create: `backend/src/health/health.module.ts`, `backend/src/health/health.controller.ts`
- Create: `backend/test/health.e2e-spec.ts`, `backend/test/jest-e2e.json`
- Move: `LocaTudo.dc.html`, `support.js`, `image-slot.js`, `uploads/` → `legacy/`

**Interfaces:**
- Consumes: nada (primeira task).
- Produces: `AppModule`; `AllExceptionsFilter` (filtro global); `GET /api/health` → `{ status: 'ok', db: 'desconhecido', redis: 'desconhecido' }`.

- [ ] **Step 1: Mover o protótipo para `legacy/`**

```bash
mkdir -p legacy
git mv LocaTudo.dc.html support.js image-slot.js legacy/
git mv uploads legacy/uploads
git commit -m "chore: move prototipo para legacy/ como referencia visual"
```

`assets/logo-locatudo.png` **não** se move: o frontend vai usá-lo.

- [ ] **Step 2: Criar `.gitignore` e `.env.example`**

`.gitignore`:
```
node_modules/
dist/
.env
*.log
backend/uploads/
.thumbnail
graphify-out/cache/
```

`.env.example`:
```
POSTGRES_USER=locatudo
POSTGRES_PASSWORD=locatudo_dev
POSTGRES_DB=locatudo
DATABASE_URL=postgresql://locatudo:locatudo_dev@db:5432/locatudo?schema=public
REDIS_URL=redis://redis:6379
PORT=3000
NODE_ENV=development
```

- [ ] **Step 3: Gerar o projeto NestJS**

```bash
docker run --rm -v "$PWD":/w -w /w node:20-alpine \
  npx -y @nestjs/cli@10 new backend --package-manager npm --skip-git --skip-install
docker run --rm -v "$PWD/backend":/app -w /app node:20-alpine \
  npm install @nestjs/config class-validator class-transformer nestjs-pino pino-http
docker run --rm -v "$PWD/backend":/app -w /app node:20-alpine \
  npm install -D dotenv-cli
```

- [ ] **Step 4: Escrever o teste e2e do health check (vai falhar)**

`backend/test/health.e2e-spec.ts`:
```ts
import { Test } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import request from 'supertest';
import { AppModule } from '../src/app.module';

describe('Health (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({ imports: [AppModule] }).compile();
    app = moduleRef.createNestApplication();
    app.setGlobalPrefix('api');
    await app.init();
  });

  afterAll(async () => { await app.close(); });

  it('GET /api/health responde 200 com status ok', async () => {
    const res = await request(app.getHttpServer()).get('/api/health').expect(200);
    expect(res.body.status).toBe('ok');
  });

  it('rota inexistente devolve o payload de erro padronizado', async () => {
    const res = await request(app.getHttpServer()).get('/api/nao-existe').expect(404);
    expect(res.body).toMatchObject({
      statusCode: 404,
      path: '/api/nao-existe',
    });
    expect(typeof res.body.timestamp).toBe('string');
    expect(Array.isArray(res.body.message)).toBe(true);
  });
});
```

- [ ] **Step 5: Rodar o teste e confirmar que falha**

Run: `docker run --rm -v "$PWD/backend":/app -w /app node:20-alpine sh -c "npm ci && npm run test:e2e"`
Expected: FAIL — `Cannot find module '../src/health/health.controller'` ou 404 no `/api/health`.

- [ ] **Step 6: Implementar validação de env, filtro de erros, health e bootstrap**

`backend/src/config/env.validation.ts`:
```ts
import { plainToInstance } from 'class-transformer';
import { IsIn, IsNotEmpty, IsString, validateSync } from 'class-validator';

class EnvVars {
  @IsString() @IsNotEmpty() DATABASE_URL!: string;
  @IsString() @IsNotEmpty() REDIS_URL!: string;
  @IsString() @IsNotEmpty() PORT!: string;
  @IsIn(['development', 'test', 'production']) NODE_ENV!: string;
}

export function validateEnv(config: Record<string, unknown>) {
  const parsed = plainToInstance(EnvVars, config, { enableImplicitConversion: true });
  const errors = validateSync(parsed, { skipMissingProperties: false });
  if (errors.length > 0) {
    throw new Error(`Variáveis de ambiente inválidas: ${errors.map((e) => e.property).join(', ')}`);
  }
  return parsed;
}
```

`backend/src/common/filters/all-exceptions.filter.ts`:
```ts
import { ArgumentsHost, Catch, ExceptionFilter, HttpException, HttpStatus, Logger } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const status = exception instanceof HttpException
      ? exception.getStatus()
      : HttpStatus.INTERNAL_SERVER_ERROR;

    let message: string[] = ['Erro interno do servidor.'];
    let error = 'Internal Server Error';

    if (exception instanceof HttpException) {
      const body = exception.getResponse() as string | { message?: string | string[]; error?: string };
      if (typeof body === 'string') {
        message = [body];
      } else {
        message = Array.isArray(body.message) ? body.message : [body.message ?? exception.message];
        error = body.error ?? error;
      }
    }

    if (status >= 500) {
      this.logger.error(`${request.method} ${request.url}`, exception instanceof Error ? exception.stack : String(exception));
    }

    response.status(status).json({
      statusCode: status,
      error,
      message,
      path: request.url,
      timestamp: new Date().toISOString(),
    });
  }
}
```

`backend/src/health/health.controller.ts`:
```ts
import { Controller, Get } from '@nestjs/common';

@Controller('health')
export class HealthController {
  @Get()
  check() {
    return { status: 'ok', db: 'desconhecido', redis: 'desconhecido' };
  }
}
```

`backend/src/health/health.module.ts`:
```ts
import { Module } from '@nestjs/common';
import { HealthController } from './health.controller';

@Module({ controllers: [HealthController] })
export class HealthModule {}
```

`backend/src/app.module.ts`:
```ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { validateEnv } from './config/env.validation';
import { HealthModule } from './health/health.module';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, validate: validateEnv }),
    HealthModule,
  ],
})
export class AppModule {}
```

`backend/src/app.module.ts` também registra o logger estruturado, com request-id por requisição —
adicione ao array de `imports`:
```ts
import { LoggerModule } from 'nestjs-pino';
import { randomUUID } from 'node:crypto';

// dentro de imports:
    LoggerModule.forRoot({
      pinoHttp: {
        genReqId: (req) => (req.headers['x-request-id'] as string) ?? randomUUID(),
        autoLogging: true,
        transport: process.env.NODE_ENV === 'development'
          ? { target: 'pino-pretty', options: { singleLine: true } }
          : undefined,
        redact: ['req.headers.authorization', 'req.headers.cookie'],
      },
    }),
```
Instale o formatador de desenvolvimento: `docker run --rm -v "$PWD/backend":/app -w /app node:20-alpine npm install -D pino-pretty`

`backend/src/main.ts`:
```ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { Logger } from 'nestjs-pino';
import { AppModule } from './app.module';
import { AllExceptionsFilter } from './common/filters/all-exceptions.filter';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { bufferLogs: true });
  app.useLogger(app.get(Logger));
  app.setGlobalPrefix('api');
  app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
  app.useGlobalFilters(new AllExceptionsFilter());
  app.enableCors({ origin: true });
  await app.listen(Number(process.env.PORT ?? 3000), '0.0.0.0');
}
void bootstrap();
```

O filtro também é registrado no teste e2e — adicione `app.useGlobalFilters(new AllExceptionsFilter());` logo após `app.setGlobalPrefix('api')` em `test/health.e2e-spec.ts`.

- [ ] **Step 7: Criar Dockerfile de desenvolvimento e entrypoint**

`backend/Dockerfile.dev`:
```dockerfile
FROM node:20-alpine
WORKDIR /app
RUN apk add --no-cache bash
COPY package*.json ./
RUN npm ci
COPY . .
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["npm", "run", "start:dev"]
```

`backend/docker-entrypoint.sh`:
```bash
#!/usr/bin/env bash
set -e
exec "$@"
```

(As migrations entram neste entrypoint na Task 2.)

- [ ] **Step 8: Criar `docker-compose.yml` com `db` e `backend`**

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    env_file: [.env]
    environment:
      DATABASE_URL: ${DATABASE_URL}
    ports: ["3000:3000"]
    volumes:
      - ./backend:/app
      - /app/node_modules
      - uploads:/app/uploads
    depends_on:
      db: { condition: service_healthy }

volumes:
  pgdata:
  uploads:
```

- [ ] **Step 9: Subir e rodar o teste — deve passar**

```bash
cp .env.example .env
docker compose up -d --build
docker compose exec backend npm run test:e2e
curl -s localhost:3000/api/health
```
Expected: testes PASS; `curl` devolve `{"status":"ok",...}`.

- [ ] **Step 10: Commit**

```bash
git add .gitignore .env.example docker-compose.yml backend/
git commit -m "feat: esqueleto NestJS com health check, filtro de erros e Docker"
```

---

### Task 2: Prisma, schema completo e migration inicial

**Files:**
- Create: `backend/prisma/schema.prisma`, `backend/src/prisma/prisma.module.ts`, `backend/src/prisma/prisma.service.ts`
- Create: `backend/.env.test`
- Modify: `backend/src/health/health.controller.ts`, `backend/src/health/health.module.ts`, `backend/src/app.module.ts`, `backend/docker-entrypoint.sh`, `backend/package.json`
- Modify: `docker-compose.yml` (adicionar `db-test`)
- Modify: `backend/test/health.e2e-spec.ts`

**Interfaces:**
- Consumes: `AppModule`, `GET /api/health` da Task 1.
- Produces: `PrismaService extends PrismaClient` (exportado por `PrismaModule`, global); modelos Prisma `User`, `Unit`, `Category`, `Equipment`, `EquipmentImage`, `EquipmentBlock`, `Reservation`; `GET /api/health` passa a devolver `db: 'ok'`.

- [ ] **Step 1: Instalar Prisma e adicionar `db-test` ao compose**

```bash
docker compose exec backend npm install @prisma/client
docker compose exec backend npm install -D prisma
```

No `docker-compose.yml`, acrescente o serviço (banco efêmero, em `tmpfs`, sem volume — cada `up` começa limpo):
```yaml
  db-test:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: locatudo
      POSTGRES_PASSWORD: locatudo_test
      POSTGRES_DB: locatudo_test
    ports: ["5433:5432"]
    tmpfs: ["/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U locatudo -d locatudo_test"]
      interval: 5s
      timeout: 5s
      retries: 10
```
e adicione `db-test: { condition: service_healthy }` ao `depends_on` do `backend`.

`backend/.env.test`:
```
DATABASE_URL=postgresql://locatudo:locatudo_test@db-test:5432/locatudo_test?schema=public
REDIS_URL=redis://redis:6379
PORT=3000
NODE_ENV=test
```

Em `backend/package.json`, ajuste os scripts:
```json
"test:e2e": "dotenv -e .env.test -- jest --config ./test/jest-e2e.json --runInBand",
"prisma:migrate:test": "dotenv -e .env.test -- prisma migrate deploy",
"prisma:seed": "prisma db seed"
```

- [ ] **Step 2: Escrever o teste (vai falhar)**

Em `backend/test/health.e2e-spec.ts`, acrescente:
```ts
  it('GET /api/health reporta o banco conectado', async () => {
    const res = await request(app.getHttpServer()).get('/api/health').expect(200);
    expect(res.body.db).toBe('ok');
  });
```

Crie `backend/test/prisma.e2e-spec.ts`:
```ts
import { Test } from '@nestjs/testing';
import { PrismaService } from '../src/prisma/prisma.service';
import { PrismaModule } from '../src/prisma/prisma.module';

describe('PrismaService (e2e)', () => {
  let prisma: PrismaService;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({ imports: [PrismaModule] }).compile();
    prisma = moduleRef.get(PrismaService);
    await prisma.$connect();
  });

  afterAll(async () => { await prisma.$disconnect(); });

  it('todas as tabelas do dominio existem apos a migration inicial', async () => {
    const rows = await prisma.$queryRaw<{ table_name: string }[]>`
      SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'
    `;
    const tables = rows.map((r) => r.table_name);
    expect(tables).toEqual(expect.arrayContaining([
      'User', 'Unit', 'Category', 'Equipment', 'EquipmentImage', 'EquipmentBlock', 'Reservation',
    ]));
  });
});
```

- [ ] **Step 3: Rodar e confirmar a falha**

Run: `docker compose exec backend npm run test:e2e`
Expected: FAIL — `Cannot find module '../src/prisma/prisma.service'`.

- [ ] **Step 4: Escrever o schema**

`backend/prisma/schema.prisma`:
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum Role            { CLIENTE ADMIN }
enum BillingMode     { HORA DIA }
enum ReservationStatus { PENDENTE CONFIRMADA EM_ANDAMENTO DEVOLVIDA CANCELADA }
enum PaymentMethod   { PIX CARTAO DINHEIRO }
enum PaymentStatus   { PENDENTE PAGO ESTORNADO }

model User {
  id           String   @id @default(uuid())
  nome         String
  email        String   @unique
  senhaHash    String
  telefone     String?
  documento    String?
  role         Role     @default(CLIENTE)
  reservations Reservation[]
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
}

model Unit {
  id         String      @id @default(uuid())
  nome       String
  cidade     String
  uf         String      @db.Char(2)
  telefone   String?
  equipments Equipment[]

  @@unique([cidade, uf])
}

model Category {
  id         String      @id @default(uuid())
  slug       String      @unique
  nome       String
  mono       String      @db.VarChar(4)
  ordem      Int         @default(0)
  equipments Equipment[]
}

model Equipment {
  id             String  @id @default(uuid())
  slug           String  @unique
  nome           String
  categoryId     String
  category       Category @relation(fields: [categoryId], references: [id])
  unitId         String
  unit           Unit     @relation(fields: [unitId], references: [id])
  marca          String
  modelo         String
  descricao      String
  specs          String[]
  quantidade     Int      @default(1)
  precoHora      Decimal? @db.Decimal(10, 2)
  precoDia       Decimal? @db.Decimal(10, 2)
  hasHour        Boolean  @default(false)
  hasDay         Boolean  @default(true)
  ativo          Boolean  @default(true)
  avaliacaoMedia Decimal  @default(0) @db.Decimal(3, 2)
  avaliacaoQtd   Int      @default(0)
  images         EquipmentImage[]
  blocks         EquipmentBlock[]
  reservations   Reservation[]
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt

  @@index([categoryId, ativo])
  @@index([unitId, ativo])
}

model EquipmentImage {
  id          String    @id @default(uuid())
  equipmentId String
  equipment   Equipment @relation(fields: [equipmentId], references: [id], onDelete: Cascade)
  path        String
  alt         String?
  ordem       Int       @default(0)

  @@index([equipmentId, ordem])
}

model EquipmentBlock {
  id          String    @id @default(uuid())
  equipmentId String
  equipment   Equipment @relation(fields: [equipmentId], references: [id], onDelete: Cascade)
  dataInicio  DateTime  @db.Date
  dataFim     DateTime  @db.Date
  motivo      String?

  @@index([equipmentId, dataInicio, dataFim])
}

model Reservation {
  id            String            @id @default(uuid())
  codigo        String            @unique
  userId        String
  user          User              @relation(fields: [userId], references: [id])
  equipmentId   String
  equipment     Equipment         @relation(fields: [equipmentId], references: [id])
  dataInicio    DateTime          @db.Date
  dataFim       DateTime          @db.Date
  billingMode   BillingMode
  dias          Int
  precoUnitario Decimal           @db.Decimal(10, 2)
  subtotal      Decimal           @db.Decimal(10, 2)
  taxa          Decimal           @db.Decimal(10, 2)
  total         Decimal           @db.Decimal(10, 2)
  status        ReservationStatus @default(PENDENTE)
  paymentMethod PaymentMethod
  paymentStatus PaymentStatus     @default(PENDENTE)
  nomeCliente   String
  telefone      String
  documento     String
  email         String
  cep           String
  endereco      String
  numero        String
  cidade        String
  createdAt     DateTime          @default(now())
  updatedAt     DateTime          @updatedAt

  @@index([equipmentId, dataInicio, dataFim])
  @@index([userId])
}
```

Todas as tabelas nascem aqui, inclusive as que só serão usadas nas fases 2 e 3 — o que muda por fase são os módulos que as consomem, não o schema.

- [ ] **Step 5: Criar `PrismaService` e `PrismaModule`**

`backend/src/prisma/prisma.service.ts`:
```ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() { await this.$connect(); }
  async onModuleDestroy() { await this.$disconnect(); }
}
```

`backend/src/prisma/prisma.module.ts`:
```ts
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({ providers: [PrismaService], exports: [PrismaService] })
export class PrismaModule {}
```

Importe `PrismaModule` em `AppModule`.

- [ ] **Step 6: Fazer o health check consultar o banco**

`backend/src/health/health.controller.ts`:
```ts
import { Controller, Get } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

@Controller('health')
export class HealthController {
  constructor(private readonly prisma: PrismaService) {}

  @Get()
  async check() {
    let db = 'ok';
    try {
      await this.prisma.$queryRaw`SELECT 1`;
    } catch {
      db = 'indisponivel';
    }
    return { status: 'ok', db, redis: 'desconhecido' };
  }
}
```

- [ ] **Step 7: Gerar a migration e rodar migrations no entrypoint**

```bash
docker compose exec backend npx prisma migrate dev --name init
```

`backend/docker-entrypoint.sh`:
```bash
#!/usr/bin/env bash
set -e

if [ "$NODE_ENV" = "production" ]; then
  npx prisma migrate deploy
else
  npx prisma generate
  npx prisma migrate deploy
fi

exec "$@"
```

- [ ] **Step 8: Aplicar a migration no banco de teste e rodar os testes**

```bash
docker compose exec backend npm run prisma:migrate:test
docker compose exec backend npm run test:e2e
```
Expected: PASS — as sete tabelas existem e `db` é `ok`.

- [ ] **Step 9: Commit**

```bash
git add backend/prisma backend/src backend/.env.test backend/package.json backend/docker-entrypoint.sh docker-compose.yml
git commit -m "feat: schema Prisma completo, PrismaService e migration inicial"
```

---

### Task 3: Seed do catálogo

**Files:**
- Create: `backend/prisma/seed.ts`, `backend/prisma/seed-assets/` (5 SVGs)
- Create: `backend/test/seed.e2e-spec.ts`
- Modify: `backend/package.json` (bloco `prisma.seed`), `backend/docker-entrypoint.sh`

**Interfaces:**
- Consumes: modelos Prisma da Task 2.
- Produces: banco populado com 8 categorias, 2 unidades, 5 equipamentos (cada um com 1 imagem) e 1 usuário admin (`admin@locatudo.com.br`); arquivos servidos em `/uploads/seed/<slug>.svg`.

Os dados vêm do protótipo (`legacy/LocaTudo.dc.html`, linhas 533–549). As fotos reais dos equipamentos entram pela administração na fase 3; aqui cada equipamento recebe um SVG neutro com o próprio nome, para o catálogo já aparecer ilustrado. Os JPEGs em `legacy/uploads/` são posts institucionais do Instagram, não fotos de catálogo — ficam como referência de marca.

- [ ] **Step 1: Escrever o teste (vai falhar)**

`backend/test/seed.e2e-spec.ts`:
```ts
import { Test } from '@nestjs/testing';
import { PrismaModule } from '../src/prisma/prisma.module';
import { PrismaService } from '../src/prisma/prisma.service';

describe('Seed (e2e)', () => {
  let prisma: PrismaService;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({ imports: [PrismaModule] }).compile();
    prisma = moduleRef.get(PrismaService);
    await prisma.$connect();
  });

  afterAll(async () => { await prisma.$disconnect(); });

  it('carrega 8 categorias, 2 unidades e 5 equipamentos', async () => {
    expect(await prisma.category.count()).toBe(8);
    expect(await prisma.unit.count()).toBe(2);
    expect(await prisma.equipment.count()).toBe(5);
  });

  it('o andaime tem multiplas unidades, para permitir disponibilidade parcial', async () => {
    const andaime = await prisma.equipment.findUnique({ where: { slug: 'andaime' } });
    expect(andaime?.quantidade).toBeGreaterThan(1);
  });

  it('equipamento sem locacao por hora nao tem precoHora', async () => {
    const andaime = await prisma.equipment.findUnique({ where: { slug: 'andaime' } });
    expect(andaime?.hasHour).toBe(false);
    expect(andaime?.precoHora).toBeNull();
  });

  it('todo equipamento tem ao menos uma imagem', async () => {
    const semImagem = await prisma.equipment.count({ where: { images: { none: {} } } });
    expect(semImagem).toBe(0);
  });

  it('rodar o seed duas vezes nao duplica registros (idempotente)', async () => {
    const antes = await prisma.equipment.count();
    const { execSync } = await import('node:child_process');
    execSync('npx prisma db seed', { env: process.env, stdio: 'inherit' });
    expect(await prisma.equipment.count()).toBe(antes);
  });
});
```

- [ ] **Step 2: Rodar e confirmar a falha**

Run: `docker compose exec backend npm run test:e2e -- seed`
Expected: FAIL — contagens em 0.

- [ ] **Step 3: Criar os SVGs de exemplo**

`backend/prisma/seed-assets/betoneira.svg` (repita trocando o texto e o nome do arquivo para `andaime`, `gerador`, `martelete`, `pistola`):
```svg
<svg xmlns="http://www.w3.org/2000/svg" width="800" height="600" viewBox="0 0 800 600">
  <rect width="800" height="600" fill="#14205A"/>
  <rect x="0" y="540" width="800" height="60" fill="#FFCB05"/>
  <text x="400" y="300" fill="#FFFFFF" font-family="Poppins, sans-serif" font-size="46"
        font-weight="800" text-anchor="middle">Betoneira 400L</text>
  <text x="400" y="350" fill="#FFCB05" font-family="Poppins, sans-serif" font-size="22"
        text-anchor="middle">LocaTudo Constru&amp;CIA</text>
</svg>
```

- [ ] **Step 4: Escrever o seed**

`backend/prisma/seed.ts`:
```ts
import { PrismaClient, Prisma } from '@prisma/client';
import { copyFile, mkdir, readdir } from 'node:fs/promises';
import { join } from 'node:path';

const prisma = new PrismaClient();

const UPLOADS_SEED_DIR = join(process.cwd(), 'uploads', 'seed');
const SEED_ASSETS_DIR = join(process.cwd(), 'prisma', 'seed-assets');

const categorias = [
  { slug: 'eletricas',    nome: 'Ferramentas elétricas',        mono: 'FE', ordem: 1 },
  { slug: 'manuais',      nome: 'Ferramentas manuais',          mono: 'FM', ordem: 2 },
  { slug: 'maquinas',     nome: 'Máquinas para construção',     mono: 'MC', ordem: 3 },
  { slug: 'elevacao',     nome: 'Equipamentos de elevação',     mono: 'EL', ordem: 4 },
  { slug: 'compactacao',  nome: 'Equipamentos de compactação',  mono: 'CP', ordem: 5 },
  { slug: 'concreto',     nome: 'Equipamentos para concreto',   mono: 'CO', ordem: 6 },
  { slug: 'geradores',    nome: 'Geradores',                    mono: 'GE', ordem: 7 },
  { slug: 'jardinagem',   nome: 'Jardinagem',                   mono: 'JD', ordem: 8 },
];

const unidades = [
  { cidade: 'Capanema', uf: 'PR', nome: 'LocaTudo Capanema', telefone: '(46) 99105-4226' },
  { cidade: 'Realeza',  uf: 'PR', nome: 'LocaTudo Realeza',  telefone: '(46) 99917-0071' },
];

const equipamentos = [
  {
    slug: 'betoneira', nome: 'Betoneira 400L', categoria: 'concreto', cidade: 'Capanema',
    marca: 'CSM', modelo: 'RentalForce 400', quantidade: 3,
    precoHora: '18.00', precoDia: '120.00', hasHour: true, hasDay: true,
    avaliacaoMedia: '4.90', avaliacaoQtd: 128,
    descricao: 'Betoneira de 400 litros para preparo de concreto e argamassa em obras de médio e grande porte. Motor robusto, fácil operação e boa produtividade.',
    specs: ['Capacidade: 400 litros', 'Motor: 1.5 CV monofásico', 'Rodízio para transporte na obra', 'Peso: 95 kg'],
  },
  {
    slug: 'andaime', nome: 'Andaime Tubular (módulo 1,5x1m)', categoria: 'elevacao', cidade: 'Realeza',
    marca: 'LocaTudo', modelo: 'Tubular Multidirecional', quantidade: 40,
    precoHora: null, precoDia: '45.00', hasHour: false, hasDay: true,
    avaliacaoMedia: '4.80', avaliacaoQtd: 64,
    descricao: 'Módulo de andaime tubular multidirecional, ideal para fachadas e obras verticais. Conformidade com NR-35. Alugado por módulo/dia.',
    specs: ['Altura por módulo: 2m', 'Carga: até 150kg/m²', 'Estrutura em aço galvanizado', 'Inclui base e sapatas niveladoras'],
  },
  {
    slug: 'gerador', nome: 'Gerador a Diesel 5kVA', categoria: 'geradores', cidade: 'Capanema',
    marca: 'Toyama', modelo: 'TDG5000', quantidade: 2,
    precoHora: '25.00', precoDia: '140.00', hasHour: true, hasDay: true,
    avaliacaoMedia: '4.70', avaliacaoQtd: 41,
    descricao: 'Gerador a diesel 5kVA silenciado, ideal para obras sem acesso à rede elétrica ou como backup de energia.',
    specs: ['Potência: 5kVA', 'Autonomia: até 10h', 'Partida elétrica', 'Nível de ruído reduzido'],
  },
  {
    slug: 'martelete', nome: 'Martelete Rompedor SDS-Max', categoria: 'eletricas', cidade: 'Realeza',
    marca: 'Bosch', modelo: 'GSH 5', quantidade: 4,
    precoHora: '15.00', precoDia: '90.00', hasHour: true, hasDay: true,
    avaliacaoMedia: '4.90', avaliacaoQtd: 97,
    descricao: 'Martelete rompedor profissional para demolição de concreto, alvenaria e pisos. Alta força de impacto e vibração reduzida.',
    specs: ['Energia de impacto: 12,5 J', 'Encaixe: SDS-Max', 'Sistema anti-vibração', 'Inclui 1 ponteiro e 1 cinzel'],
  },
  {
    slug: 'pistola', nome: 'Pistola Elétrica de Pintura', categoria: 'eletricas', cidade: 'Capanema',
    marca: 'Vonder', modelo: 'PEV 400', quantidade: 2,
    precoHora: '12.00', precoDia: '70.00', hasHour: true, hasDay: true,
    avaliacaoMedia: '4.80', avaliacaoQtd: 53,
    descricao: 'Pistola elétrica para pintura sem uso de compressor. Bico com regulagem vertical e horizontal, compatível com tintas látex, automotivas e vernizes.',
    specs: ['Vazão ajustável', 'Bico com regulagem vertical/horizontal', 'Reservatório de 800ml', 'Compatível com tintas à base de água e solvente'],
  },
];

async function copiarImagens() {
  await mkdir(UPLOADS_SEED_DIR, { recursive: true });
  for (const arquivo of await readdir(SEED_ASSETS_DIR)) {
    await copyFile(join(SEED_ASSETS_DIR, arquivo), join(UPLOADS_SEED_DIR, arquivo));
  }
}

async function main() {
  await copiarImagens();

  for (const c of categorias) {
    await prisma.category.upsert({ where: { slug: c.slug }, update: c, create: c });
  }

  for (const u of unidades) {
    await prisma.unit.upsert({
      where: { cidade_uf: { cidade: u.cidade, uf: u.uf } },
      update: u,
      create: u,
    });
  }

  for (const e of equipamentos) {
    const categoria = await prisma.category.findUniqueOrThrow({ where: { slug: e.categoria } });
    const unidade = await prisma.unit.findUniqueOrThrow({
      where: { cidade_uf: { cidade: e.cidade, uf: 'PR' } },
    });

    const dados = {
      nome: e.nome,
      categoryId: categoria.id,
      unitId: unidade.id,
      marca: e.marca,
      modelo: e.modelo,
      descricao: e.descricao,
      specs: e.specs,
      quantidade: e.quantidade,
      precoHora: e.precoHora ? new Prisma.Decimal(e.precoHora) : null,
      precoDia: e.precoDia ? new Prisma.Decimal(e.precoDia) : null,
      hasHour: e.hasHour,
      hasDay: e.hasDay,
      avaliacaoMedia: new Prisma.Decimal(e.avaliacaoMedia),
      avaliacaoQtd: e.avaliacaoQtd,
    };

    const equipamento = await prisma.equipment.upsert({
      where: { slug: e.slug },
      update: dados,
      create: { slug: e.slug, ...dados },
    });

    const path = `/uploads/seed/${e.slug}.svg`;
    const jaTem = await prisma.equipmentImage.findFirst({ where: { equipmentId: equipamento.id, path } });
    if (!jaTem) {
      await prisma.equipmentImage.create({
        data: { equipmentId: equipamento.id, path, alt: e.nome, ordem: 0 },
      });
    }
  }

  await prisma.user.upsert({
    where: { email: 'admin@locatudo.com.br' },
    update: {},
    create: {
      nome: 'Administrador LocaTudo',
      email: 'admin@locatudo.com.br',
      // Substituído por hash argon2 na fase 3, quando o módulo auth existir.
      senhaHash: 'trocar-na-fase-3',
      role: 'ADMIN',
    },
  });

  console.log('Seed concluído.');
}

main()
  .catch((e) => { console.error(e); process.exit(1); })
  .finally(() => prisma.$disconnect());
```

O `upsert` em toda entidade é o que torna o seed idempotente — rodar no boot de cada `docker compose up` não duplica nada.

- [ ] **Step 5: Registrar o seed e rodá-lo no entrypoint**

Em `backend/package.json`, no nível raiz do JSON:
```json
"prisma": { "seed": "ts-node prisma/seed.ts" }
```
Instale o executor: `docker compose exec backend npm install -D ts-node`

Em `backend/docker-entrypoint.sh`, após as migrations:
```bash
if [ "$NODE_ENV" != "production" ]; then
  npx prisma db seed
fi
```

- [ ] **Step 6: Rodar seed e testes**

```bash
docker compose exec backend npx prisma db seed
docker compose exec backend npm run prisma:migrate:test
docker compose exec backend sh -c "dotenv -e .env.test -- npx prisma db seed"
docker compose exec backend npm run test:e2e -- seed
```
Expected: PASS nos cinco casos.

- [ ] **Step 7: Commit**

```bash
git add backend/prisma backend/package.json backend/docker-entrypoint.sh backend/test
git commit -m "feat: seed idempotente do catalogo com categorias, unidades e equipamentos"
```

---

### Task 4: Camada de cache Redis (read-through e versionamento)

O coração da fase. Sem domínio: só chave, versão, TTL e degradação.

**Files:**
- Create: `backend/src/cache/cache.constants.ts`, `backend/src/cache/cache.service.ts`, `backend/src/cache/cache.module.ts`
- Create: `backend/src/cache/cache.service.spec.ts` (unitário), `backend/test/cache.e2e-spec.ts` (com Redis real)
- Modify: `docker-compose.yml` (serviço `redis`), `backend/src/app.module.ts`, `backend/src/health/health.controller.ts`, `backend/src/health/health.module.ts`

**Interfaces:**
- Consumes: `AppModule` (Task 1).
- Produces: `CacheModule` (global) exportando `CacheService` e o token `REDIS_CLIENT`, com a API:
  - `getOrSet<T>(namespace: string, suffix: string, ttlSeconds: number, loader: () => Promise<T>): Promise<T>`
  - `bump(namespace: string): Promise<void>`
  - `hashFilters(filtros: Record<string, unknown>): string`
  - Constantes `CACHE_NS = { CATEGORIES: 'cat', EQUIPMENTS: 'eq', UNITS: 'unit' }` e `CACHE_TTL = { CATEGORIES: 3600, UNITS: 3600, EQUIPMENTS: 600, AVAILABILITY: 60 }`
  - `GET /api/health` passa a devolver `redis: 'ok'`.

- [ ] **Step 1: Adicionar o Redis ao compose e instalar o cliente**

```yaml
  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    ports: ["6379:6379"]
    volumes: ["redisdata:/data"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10
```
Adicione `redis: { condition: service_healthy }` ao `depends_on` do `backend` e `redisdata:` à lista de `volumes:`.

```bash
docker compose exec backend npm install ioredis
```

- [ ] **Step 2: Escrever os testes unitários (vão falhar)**

`backend/src/cache/cache.service.spec.ts`:
```ts
import { Test } from '@nestjs/testing';
import { CacheService } from './cache.service';
import { REDIS_CLIENT } from './cache.constants';

function fakeRedis() {
  const store = new Map<string, string>();
  return {
    store,
    get: jest.fn(async (k: string) => store.get(k) ?? null),
    set: jest.fn(async (k: string, v: string) => { store.set(k, v); return 'OK'; }),
    incr: jest.fn(async (k: string) => {
      const next = Number(store.get(k) ?? '0') + 1;
      store.set(k, String(next));
      return next;
    }),
  };
}

describe('CacheService', () => {
  let redis: ReturnType<typeof fakeRedis>;
  let cache: CacheService;

  beforeEach(async () => {
    redis = fakeRedis();
    const moduleRef = await Test.createTestingModule({
      providers: [CacheService, { provide: REDIS_CLIENT, useValue: redis }],
    }).compile();
    cache = moduleRef.get(CacheService);
  });

  it('no primeiro acesso chama o loader e grava no Redis', async () => {
    const loader = jest.fn(async () => ({ nome: 'Betoneira' }));
    const result = await cache.getOrSet('eq', 'slug:betoneira', 600, loader);

    expect(result).toEqual({ nome: 'Betoneira' });
    expect(loader).toHaveBeenCalledTimes(1);
    expect(redis.set).toHaveBeenCalledWith('eq:v0:slug:betoneira', JSON.stringify({ nome: 'Betoneira' }), 'EX', 600);
  });

  it('no segundo acesso serve do cache sem chamar o loader', async () => {
    const loader = jest.fn(async () => ({ nome: 'Betoneira' }));
    await cache.getOrSet('eq', 'slug:betoneira', 600, loader);
    const result = await cache.getOrSet('eq', 'slug:betoneira', 600, loader);

    expect(result).toEqual({ nome: 'Betoneira' });
    expect(loader).toHaveBeenCalledTimes(1);
  });

  it('bump troca a versao e forca uma nova leitura do loader', async () => {
    const loader = jest.fn()
      .mockResolvedValueOnce({ preco: 120 })
      .mockResolvedValueOnce({ preco: 150 });

    await cache.getOrSet('eq', 'slug:betoneira', 600, loader);
    await cache.bump('eq');
    const result = await cache.getOrSet('eq', 'slug:betoneira', 600, loader);

    expect(result).toEqual({ preco: 150 });
    expect(loader).toHaveBeenCalledTimes(2);
    expect(redis.store.has('eq:v1:slug:betoneira')).toBe(true);
  });

  it('se o Redis falha na leitura, cai no loader sem propagar erro', async () => {
    redis.get.mockRejectedValue(new Error('ECONNREFUSED'));
    const loader = jest.fn(async () => ({ nome: 'Gerador' }));

    await expect(cache.getOrSet('eq', 'slug:gerador', 600, loader)).resolves.toEqual({ nome: 'Gerador' });
    expect(loader).toHaveBeenCalledTimes(1);
  });

  it('se o Redis falha na escrita, ainda devolve o valor do loader', async () => {
    redis.set.mockRejectedValue(new Error('OOM'));
    const loader = jest.fn(async () => ({ nome: 'Gerador' }));

    await expect(cache.getOrSet('eq', 'slug:gerador', 600, loader)).resolves.toEqual({ nome: 'Gerador' });
  });

  it('bump nao lanca quando o Redis esta fora', async () => {
    redis.incr.mockRejectedValue(new Error('ECONNREFUSED'));
    await expect(cache.bump('eq')).resolves.toBeUndefined();
  });

  it('hashFilters ignora a ordem das chaves e distingue valores diferentes', () => {
    expect(cache.hashFilters({ q: 'betoneira', page: 1 })).toBe(cache.hashFilters({ page: 1, q: 'betoneira' }));
    expect(cache.hashFilters({ q: 'betoneira' })).not.toBe(cache.hashFilters({ q: 'gerador' }));
  });

  it('hashFilters descarta chaves indefinidas para nao fragmentar o cache', () => {
    expect(cache.hashFilters({ q: 'betoneira', cidade: undefined })).toBe(cache.hashFilters({ q: 'betoneira' }));
  });
});
```

O terceiro teste é o que trava o bug sutil do versionamento: se a versão ausente fosse lida como `1`, o primeiro `INCR` produziria `1` de novo e a chave antiga seria reaproveitada — servindo dado velho para sempre. Por isso a versão ausente vale `0`.

- [ ] **Step 3: Rodar e confirmar a falha**

Run: `docker compose exec backend npm test -- cache.service`
Expected: FAIL — `Cannot find module './cache.service'`.

- [ ] **Step 4: Implementar**

`backend/src/cache/cache.constants.ts`:
```ts
export const REDIS_CLIENT = Symbol('REDIS_CLIENT');

export const CACHE_NS = {
  CATEGORIES: 'cat',
  UNITS: 'unit',
  EQUIPMENTS: 'eq',
} as const;

export const CACHE_TTL = {
  CATEGORIES: 3600,
  UNITS: 3600,
  EQUIPMENTS: 600,
  AVAILABILITY: 60,
} as const;
```

`backend/src/cache/cache.service.ts`:
```ts
import { Inject, Injectable, Logger } from '@nestjs/common';
import { createHash } from 'node:crypto';
import type Redis from 'ioredis';
import { REDIS_CLIENT } from './cache.constants';

@Injectable()
export class CacheService {
  private readonly logger = new Logger(CacheService.name);

  constructor(@Inject(REDIS_CLIENT) private readonly redis: Redis) {}

  /**
   * Leitura read-through: devolve do Redis quando houver, senão executa o loader,
   * grava e devolve. Qualquer falha do Redis degrada para o loader.
   */
  async getOrSet<T>(namespace: string, suffix: string, ttlSeconds: number, loader: () => Promise<T>): Promise<T> {
    let key: string | null = null;

    try {
      // Versão ausente vale 0: o primeiro bump gera 1 e invalida de fato.
      const version = (await this.redis.get(this.versionKey(namespace))) ?? '0';
      key = `${namespace}:v${version}:${suffix}`;
      const cached = await this.redis.get(key);
      if (cached !== null) {
        return JSON.parse(cached) as T;
      }
    } catch (err) {
      this.logger.warn(`Leitura de cache falhou (${namespace}:${suffix}), consultando o banco: ${String(err)}`);
      key = null;
    }

    const value = await loader();

    if (key !== null) {
      try {
        await this.redis.set(key, JSON.stringify(value), 'EX', ttlSeconds);
      } catch (err) {
        this.logger.warn(`Escrita de cache falhou (${key}): ${String(err)}`);
      }
    }

    return value;
  }

  /** Invalida um namespace inteiro em O(1), sem varrer chaves. */
  async bump(namespace: string): Promise<void> {
    try {
      await this.redis.incr(this.versionKey(namespace));
    } catch (err) {
      this.logger.warn(`Falha ao invalidar o namespace ${namespace}: ${String(err)}`);
    }
  }

  /** Assinatura estável de um conjunto de filtros, para compor a chave de listagem. */
  hashFilters(filtros: Record<string, unknown>): string {
    const normalizado = Object.entries(filtros)
      .filter(([, v]) => v !== undefined && v !== null && v !== '')
      .sort(([a], [b]) => a.localeCompare(b))
      .map(([k, v]) => `${k}=${String(v)}`)
      .join('&');
    return createHash('sha1').update(normalizado).digest('hex').slice(0, 16);
  }

  private versionKey(namespace: string): string {
    return `ver:${namespace}`;
  }
}
```

`backend/src/cache/cache.module.ts`:
```ts
import { Global, Module, OnApplicationShutdown } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import Redis from 'ioredis';
import { CacheService } from './cache.service';
import { REDIS_CLIENT } from './cache.constants';

@Global()
@Module({
  providers: [
    {
      provide: REDIS_CLIENT,
      inject: [ConfigService],
      useFactory: (config: ConfigService) =>
        new Redis(config.getOrThrow<string>('REDIS_URL'), {
          maxRetriesPerRequest: 2,
          // Sem enableOfflineQueue: false, comandos enviados com o Redis fora ficam
          // enfileirados e o try/catch do CacheService nunca dispara — a rota trava
          // em vez de degradar para o Postgres.
          enableOfflineQueue: false,
        }),
    },
    CacheService,
  ],
  exports: [CacheService, REDIS_CLIENT],
})
export class CacheModule implements OnApplicationShutdown {
  constructor(@Inject(REDIS_CLIENT) private readonly redis: Redis) {}

  async onApplicationShutdown(): Promise<void> {
    await this.redis.quit();
  }
}
```

Importe `Inject` de `@nestjs/common` junto com `Global`, `Module` e `OnApplicationShutdown`, e
habilite os hooks de shutdown em `main.ts`, antes do `listen`:
```ts
  app.enableShutdownHooks();
```

Importe `CacheModule` em `AppModule`.

- [ ] **Step 5: Rodar os unitários — devem passar**

Run: `docker compose exec backend npm test -- cache.service`
Expected: PASS nos 8 casos.

- [ ] **Step 6: Health check reportando o Redis + teste e2e com Redis real**

`backend/src/health/health.controller.ts` — injete o cliente e teste o `ping`:
```ts
import { Controller, Get, Inject } from '@nestjs/common';
import type Redis from 'ioredis';
import { PrismaService } from '../prisma/prisma.service';
import { REDIS_CLIENT } from '../cache/cache.constants';

@Controller('health')
export class HealthController {
  constructor(
    private readonly prisma: PrismaService,
    @Inject(REDIS_CLIENT) private readonly redis: Redis,
  ) {}

  @Get()
  async check() {
    let db = 'ok';
    let redis = 'ok';
    try { await this.prisma.$queryRaw`SELECT 1`; } catch { db = 'indisponivel'; }
    try { await this.redis.ping(); } catch { redis = 'indisponivel'; }
    return { status: db === 'ok' ? 'ok' : 'degradado', db, redis };
  }
}
```

`backend/test/cache.e2e-spec.ts`:
```ts
import { Test } from '@nestjs/testing';
import { ConfigModule } from '@nestjs/config';
import { CacheModule } from '../src/cache/cache.module';
import { CacheService } from '../src/cache/cache.service';
import { REDIS_CLIENT } from '../src/cache/cache.constants';
import type Redis from 'ioredis';

describe('CacheService contra Redis real (e2e)', () => {
  let cache: CacheService;
  let redis: Redis;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [ConfigModule.forRoot({ isGlobal: true }), CacheModule],
    }).compile();
    cache = moduleRef.get(CacheService);
    redis = moduleRef.get(REDIS_CLIENT);
  });

  afterAll(async () => { await redis.quit(); });

  beforeEach(async () => { await redis.flushdb(); });

  it('grava com TTL e serve do cache na segunda chamada', async () => {
    const loader = jest.fn(async () => ({ total: 5 }));
    await cache.getOrSet('eq', 'teste', 600, loader);
    await cache.getOrSet('eq', 'teste', 600, loader);

    expect(loader).toHaveBeenCalledTimes(1);
    const ttl = await redis.ttl('eq:v0:teste');
    expect(ttl).toBeGreaterThan(0);
    expect(ttl).toBeLessThanOrEqual(600);
  });

  it('apos bump, a chave antiga continua existindo mas nao e mais consultada', async () => {
    await cache.getOrSet('eq', 'teste', 600, async () => ({ total: 5 }));
    await cache.bump('eq');
    const novo = await cache.getOrSet('eq', 'teste', 600, async () => ({ total: 9 }));

    expect(novo).toEqual({ total: 9 });
    expect(await redis.get('eq:v0:teste')).not.toBeNull();
    expect(await redis.get('eq:v1:teste')).toBe(JSON.stringify({ total: 9 }));
  });
});
```

- [ ] **Step 7: Rodar todos os testes**

```bash
docker compose up -d --build
docker compose exec backend npm test
docker compose exec backend npm run test:e2e
curl -s localhost:3000/api/health
```
Expected: PASS; `curl` devolve `{"status":"ok","db":"ok","redis":"ok"}`.

- [ ] **Step 8: Commit**

```bash
git add backend/src/cache backend/src/health backend/test/cache.e2e-spec.ts docker-compose.yml backend/package.json
git commit -m "feat: cache Redis read-through com versionamento de namespace e degradacao"
```

---

### Task 5: Módulos de catálogo de referência (categorias e unidades)

Duas rotas pequenas e simétricas — a primeira aplicação real do `CacheService`.

**Files:**
- Create: `backend/src/modules/categories/{categories.module.ts,categories.controller.ts,categories.service.ts,dto/category-response.dto.ts}`
- Create: `backend/src/modules/units/{units.module.ts,units.controller.ts,units.service.ts}`
- Create: `backend/test/catalog.e2e-spec.ts`
- Modify: `backend/src/app.module.ts`

**Interfaces:**
- Consumes: `PrismaService` (Task 2), `CacheService`, `CACHE_NS`, `CACHE_TTL` (Task 4), dados do seed (Task 3).
- Produces:
  - `GET /api/categories` → `CategoryResponseDto[]` = `{ slug, nome, mono, ordem, quantidadeEquipamentos }[]`, ordenado por `ordem`.
  - `GET /api/units` → `{ id, nome, cidade, uf, telefone }[]`.
  - `CategoriesService.findAll(): Promise<CategoryResponseDto[]>`, `UnitsService.findAll()`.

- [ ] **Step 1: Escrever o teste e2e (vai falhar)**

`backend/test/catalog.e2e-spec.ts`:
```ts
import { Test } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import request from 'supertest';
import type Redis from 'ioredis';
import { AppModule } from '../src/app.module';
import { AllExceptionsFilter } from '../src/common/filters/all-exceptions.filter';
import { REDIS_CLIENT } from '../src/cache/cache.constants';

describe('Catálogo de referência (e2e)', () => {
  let app: INestApplication;
  let redis: Redis;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({ imports: [AppModule] }).compile();
    app = moduleRef.createNestApplication();
    app.setGlobalPrefix('api');
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
    app.useGlobalFilters(new AllExceptionsFilter());
    await app.init();
    redis = app.get(REDIS_CLIENT);
  });

  afterAll(async () => { await app.close(); });

  beforeEach(async () => { await redis.flushdb(); });

  it('GET /api/categories devolve as 8 categorias ordenadas com a contagem', async () => {
    const res = await request(app.getHttpServer()).get('/api/categories').expect(200);

    expect(res.body).toHaveLength(8);
    expect(res.body[0]).toEqual({
      slug: 'eletricas',
      nome: 'Ferramentas elétricas',
      mono: 'FE',
      ordem: 1,
      quantidadeEquipamentos: 2,
    });
    expect(res.body.map((c: { ordem: number }) => c.ordem)).toEqual([1, 2, 3, 4, 5, 6, 7, 8]);
  });

  it('a segunda chamada e servida pelo Redis', async () => {
    await request(app.getHttpServer()).get('/api/categories').expect(200);
    const chaves = await redis.keys('cat:v0:*');
    expect(chaves).toHaveLength(1);
  });

  it('GET /api/units devolve as duas unidades do Paraná', async () => {
    const res = await request(app.getHttpServer()).get('/api/units').expect(200);
    expect(res.body).toHaveLength(2);
    expect(res.body.map((u: { cidade: string }) => u.cidade).sort()).toEqual(['Capanema', 'Realeza']);
    expect(res.body[0]).toHaveProperty('telefone');
  });
});
```

- [ ] **Step 2: Rodar e confirmar a falha**

Run: `docker compose exec backend npm run test:e2e -- catalog`
Expected: FAIL — 404 em `/api/categories`.

- [ ] **Step 3: Implementar categorias**

`backend/src/modules/categories/dto/category-response.dto.ts`:
```ts
export class CategoryResponseDto {
  slug!: string;
  nome!: string;
  mono!: string;
  ordem!: number;
  quantidadeEquipamentos!: number;
}
```

`backend/src/modules/categories/categories.service.ts`:
```ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service';
import { CacheService } from '../../cache/cache.service';
import { CACHE_NS, CACHE_TTL } from '../../cache/cache.constants';
import { CategoryResponseDto } from './dto/category-response.dto';

@Injectable()
export class CategoriesService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly cache: CacheService,
  ) {}

  async findAll(): Promise<CategoryResponseDto[]> {
    return this.cache.getOrSet(CACHE_NS.CATEGORIES, 'all', CACHE_TTL.CATEGORIES, async () => {
      const categorias = await this.prisma.category.findMany({
        orderBy: { ordem: 'asc' },
        include: { _count: { select: { equipments: { where: { ativo: true } } } } },
      });

      return categorias.map((c) => ({
        slug: c.slug,
        nome: c.nome,
        mono: c.mono,
        ordem: c.ordem,
        quantidadeEquipamentos: c._count.equipments,
      }));
    });
  }
}
```

`backend/src/modules/categories/categories.controller.ts`:
```ts
import { Controller, Get } from '@nestjs/common';
import { CategoriesService } from './categories.service';
import { CategoryResponseDto } from './dto/category-response.dto';

@Controller('categories')
export class CategoriesController {
  constructor(private readonly categories: CategoriesService) {}

  @Get()
  findAll(): Promise<CategoryResponseDto[]> {
    return this.categories.findAll();
  }
}
```

`backend/src/modules/categories/categories.module.ts`:
```ts
import { Module } from '@nestjs/common';
import { CategoriesController } from './categories.controller';
import { CategoriesService } from './categories.service';

@Module({
  controllers: [CategoriesController],
  providers: [CategoriesService],
  exports: [CategoriesService],
})
export class CategoriesModule {}
```

- [ ] **Step 4: Implementar unidades**

`backend/src/modules/units/units.service.ts`:
```ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service';
import { CacheService } from '../../cache/cache.service';
import { CACHE_NS, CACHE_TTL } from '../../cache/cache.constants';

@Injectable()
export class UnitsService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly cache: CacheService,
  ) {}

  findAll() {
    return this.cache.getOrSet(CACHE_NS.UNITS, 'all', CACHE_TTL.UNITS, () =>
      this.prisma.unit.findMany({
        orderBy: { cidade: 'asc' },
        select: { id: true, nome: true, cidade: true, uf: true, telefone: true },
      }),
    );
  }
}
```

`backend/src/modules/units/units.controller.ts`:
```ts
import { Controller, Get } from '@nestjs/common';
import { UnitsService } from './units.service';

@Controller('units')
export class UnitsController {
  constructor(private readonly units: UnitsService) {}

  @Get()
  findAll() {
    return this.units.findAll();
  }
}
```

`backend/src/modules/units/units.module.ts`:
```ts
import { Module } from '@nestjs/common';
import { UnitsController } from './units.controller';
import { UnitsService } from './units.service';

@Module({
  controllers: [UnitsController],
  providers: [UnitsService],
  exports: [UnitsService],
})
export class UnitsModule {}
```

Importe `CategoriesModule` e `UnitsModule` em `AppModule`.

- [ ] **Step 5: Rodar os testes — devem passar**

Run: `docker compose exec backend npm run test:e2e -- catalog`
Expected: PASS nos três casos.

- [ ] **Step 6: Commit**

```bash
git add backend/src/modules backend/src/app.module.ts backend/test/catalog.e2e-spec.ts
git commit -m "feat: rotas de categorias e unidades servidas pelo cache"
```

---

### Task 6: Listagem de equipamentos com filtros, ordenação e paginação

**Files:**
- Create: `backend/src/common/dto/pagination.dto.ts`
- Create: `backend/src/modules/equipments/{equipments.module.ts,equipments.controller.ts,equipments.service.ts}`
- Create: `backend/src/modules/equipments/dto/{list-equipments.dto.ts,equipment-list-item.dto.ts}`
- Create: `backend/test/equipments-list.e2e-spec.ts`
- Modify: `backend/src/app.module.ts`

**Interfaces:**
- Consumes: `PrismaService`, `CacheService`, `CACHE_NS.EQUIPMENTS`, `CACHE_TTL.EQUIPMENTS`.
- Produces:
  - `GET /api/equipments?q&categoria&cidade&precoMax&ordenar&page&limit` → `{ data: EquipmentListItemDto[], meta: { page, limit, total, totalPages } }`
  - `EquipmentListItemDto` = `{ id, slug, nome, marca, modelo, categoria: { slug, nome }, cidade, uf, precoHora: number|null, precoDia: number|null, hasHour, hasDay, avaliacaoMedia: number, avaliacaoQtd: number, imagem: string|null }`
  - `EquipmentsService.list(filtros: ListEquipmentsDto)`

- [ ] **Step 1: Escrever o teste e2e (vai falhar)**

`backend/test/equipments-list.e2e-spec.ts`:
```ts
import { Test } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import request from 'supertest';
import type Redis from 'ioredis';
import { AppModule } from '../src/app.module';
import { AllExceptionsFilter } from '../src/common/filters/all-exceptions.filter';
import { REDIS_CLIENT } from '../src/cache/cache.constants';

describe('Listagem de equipamentos (e2e)', () => {
  let app: INestApplication;
  let redis: Redis;
  const get = (url: string) => request(app.getHttpServer()).get(url);

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({ imports: [AppModule] }).compile();
    app = moduleRef.createNestApplication();
    app.setGlobalPrefix('api');
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
    app.useGlobalFilters(new AllExceptionsFilter());
    await app.init();
    redis = app.get(REDIS_CLIENT);
  });

  afterAll(async () => { await app.close(); });
  beforeEach(async () => { await redis.flushdb(); });

  it('sem filtros devolve os 5 equipamentos com envelope de paginacao', async () => {
    const res = await get('/api/equipments').expect(200);
    expect(res.body.data).toHaveLength(5);
    expect(res.body.meta).toEqual({ page: 1, limit: 12, total: 5, totalPages: 1 });
  });

  it('cada item traz preco numerico, categoria aninhada e a primeira imagem', async () => {
    const res = await get('/api/equipments?q=betoneira').expect(200);
    expect(res.body.data[0]).toMatchObject({
      slug: 'betoneira',
      marca: 'CSM',
      categoria: { slug: 'concreto', nome: 'Equipamentos para concreto' },
      cidade: 'Capanema',
      uf: 'PR',
      precoDia: 120,
      precoHora: 18,
      hasHour: true,
      hasDay: true,
      imagem: '/uploads/seed/betoneira.svg',
    });
    expect(typeof res.body.data[0].avaliacaoMedia).toBe('number');
  });

  it('busca textual encontra por marca, ignorando maiusculas', async () => {
    const res = await get('/api/equipments?q=BOSCH').expect(200);
    expect(res.body.data.map((e: { slug: string }) => e.slug)).toEqual(['martelete']);
  });

  it('filtra por categoria', async () => {
    const res = await get('/api/equipments?categoria=eletricas').expect(200);
    expect(res.body.data.map((e: { slug: string }) => e.slug).sort()).toEqual(['martelete', 'pistola']);
  });

  it('filtra por cidade', async () => {
    const res = await get('/api/equipments?cidade=Realeza').expect(200);
    expect(res.body.data.map((e: { slug: string }) => e.slug).sort()).toEqual(['andaime', 'martelete']);
  });

  it('precoMax compara pela diaria quando o item tem diaria', async () => {
    const res = await get('/api/equipments?precoMax=90').expect(200);
    expect(res.body.data.map((e: { slug: string }) => e.slug).sort()).toEqual(['andaime', 'martelete', 'pistola']);
  });

  it('ordenar=preco ordena da menor diaria para a maior', async () => {
    const res = await get('/api/equipments?ordenar=preco').expect(200);
    expect(res.body.data.map((e: { slug: string }) => e.slug)).toEqual(['andaime', 'pistola', 'martelete', 'betoneira', 'gerador']);
  });

  it('pagina os resultados', async () => {
    const res = await get('/api/equipments?page=2&limit=2').expect(200);
    expect(res.body.data).toHaveLength(2);
    expect(res.body.meta).toEqual({ page: 2, limit: 2, total: 5, totalPages: 3 });
  });

  it('filtros diferentes geram chaves de cache diferentes; a ordem dos parametros nao', async () => {
    await get('/api/equipments?q=betoneira&page=1').expect(200);
    await get('/api/equipments?page=1&q=betoneira').expect(200);
    expect(await redis.keys('eq:v0:list:*')).toHaveLength(1);

    await get('/api/equipments?q=gerador').expect(200);
    expect(await redis.keys('eq:v0:list:*')).toHaveLength(2);
  });

  it('parametro invalido devolve 400 com mensagem em portugues', async () => {
    const res = await get('/api/equipments?page=zero').expect(400);
    expect(res.body.statusCode).toBe(400);
    expect(res.body.message.join(' ')).toMatch(/page/);
  });
});
```

- [ ] **Step 2: Rodar e confirmar a falha**

Run: `docker compose exec backend npm run test:e2e -- equipments-list`
Expected: FAIL — 404 em `/api/equipments`.

- [ ] **Step 3: Implementar os DTOs**

`backend/src/common/dto/pagination.dto.ts`:
```ts
import { Type } from 'class-transformer';
import { IsInt, IsOptional, Max, Min } from 'class-validator';

export class PaginationDto {
  @IsOptional() @Type(() => Number) @IsInt({ message: 'page deve ser um número inteiro.' }) @Min(1)
  page: number = 1;

  @IsOptional() @Type(() => Number) @IsInt({ message: 'limit deve ser um número inteiro.' }) @Min(1) @Max(48)
  limit: number = 12;
}

export interface Paginated<T> {
  data: T[];
  meta: { page: number; limit: number; total: number; totalPages: number };
}
```

`backend/src/modules/equipments/dto/list-equipments.dto.ts`:
```ts
import { Type } from 'class-transformer';
import { IsIn, IsNumber, IsOptional, IsString, Min } from 'class-validator';
import { PaginationDto } from '../../../common/dto/pagination.dto';

export const ORDENACOES = ['relevancia', 'preco', 'avaliacao'] as const;
export type Ordenacao = (typeof ORDENACOES)[number];

export class ListEquipmentsDto extends PaginationDto {
  @IsOptional() @IsString() q?: string;
  @IsOptional() @IsString() categoria?: string;
  @IsOptional() @IsString() cidade?: string;

  @IsOptional() @Type(() => Number) @IsNumber({}, { message: 'precoMax deve ser numérico.' }) @Min(0)
  precoMax?: number;

  @IsOptional() @IsIn(ORDENACOES, { message: 'ordenar deve ser relevancia, preco ou avaliacao.' })
  ordenar: Ordenacao = 'relevancia';
}
```

`backend/src/modules/equipments/dto/equipment-list-item.dto.ts`:
```ts
export class EquipmentListItemDto {
  id!: string;
  slug!: string;
  nome!: string;
  marca!: string;
  modelo!: string;
  categoria!: { slug: string; nome: string };
  cidade!: string;
  uf!: string;
  precoHora!: number | null;
  precoDia!: number | null;
  hasHour!: boolean;
  hasDay!: boolean;
  avaliacaoMedia!: number;
  avaliacaoQtd!: number;
  imagem!: string | null;
}
```

- [ ] **Step 4: Implementar o service**

`backend/src/modules/equipments/equipments.service.ts`:
```ts
import { Injectable } from '@nestjs/common';
import { Prisma } from '@prisma/client';
import { PrismaService } from '../../prisma/prisma.service';
import { CacheService } from '../../cache/cache.service';
import { CACHE_NS, CACHE_TTL } from '../../cache/cache.constants';
import { Paginated } from '../../common/dto/pagination.dto';
import { ListEquipmentsDto } from './dto/list-equipments.dto';
import { EquipmentListItemDto } from './dto/equipment-list-item.dto';

const SELECAO_LISTA = {
  id: true, slug: true, nome: true, marca: true, modelo: true,
  precoHora: true, precoDia: true, hasHour: true, hasDay: true,
  avaliacaoMedia: true, avaliacaoQtd: true,
  category: { select: { slug: true, nome: true } },
  unit: { select: { cidade: true, uf: true } },
  images: { select: { path: true }, orderBy: { ordem: 'asc' as const }, take: 1 },
} satisfies Prisma.EquipmentSelect;

type LinhaEquipamento = Prisma.EquipmentGetPayload<{ select: typeof SELECAO_LISTA }>;

@Injectable()
export class EquipmentsService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly cache: CacheService,
  ) {}

  async list(filtros: ListEquipmentsDto): Promise<Paginated<EquipmentListItemDto>> {
    const suffix = `list:${this.cache.hashFilters({ ...filtros })}`;

    return this.cache.getOrSet(CACHE_NS.EQUIPMENTS, suffix, CACHE_TTL.EQUIPMENTS, async () => {
      const where = this.montarWhere(filtros);
      const skip = (filtros.page - 1) * filtros.limit;

      const [linhas, total] = await this.prisma.$transaction([
        this.prisma.equipment.findMany({
          where,
          select: SELECAO_LISTA,
          orderBy: this.montarOrdenacao(filtros.ordenar),
          skip,
          take: filtros.limit,
        }),
        this.prisma.equipment.count({ where }),
      ]);

      return {
        data: linhas.map((l) => this.paraDto(l)),
        meta: {
          page: filtros.page,
          limit: filtros.limit,
          total,
          totalPages: Math.max(1, Math.ceil(total / filtros.limit)),
        },
      };
    });
  }

  private montarWhere(filtros: ListEquipmentsDto): Prisma.EquipmentWhereInput {
    const where: Prisma.EquipmentWhereInput = { ativo: true };

    if (filtros.q) {
      const contem = { contains: filtros.q, mode: 'insensitive' as const };
      where.OR = [{ nome: contem }, { marca: contem }, { modelo: contem }, { descricao: contem }];
    }
    if (filtros.categoria) where.category = { slug: filtros.categoria };
    if (filtros.cidade) where.unit = { cidade: filtros.cidade };

    if (filtros.precoMax !== undefined) {
      // O protótipo compara pela diária quando ela existe, senão pelo preço/hora.
      where.AND = [{
        OR: [
          { hasDay: true, precoDia: { lte: filtros.precoMax } },
          { hasDay: false, precoHora: { lte: filtros.precoMax } },
        ],
      }];
    }

    return where;
  }

  private montarOrdenacao(ordenar: ListEquipmentsDto['ordenar']): Prisma.EquipmentOrderByWithRelationInput[] {
    if (ordenar === 'preco') {
      return [{ precoDia: { sort: 'asc', nulls: 'last' } }, { precoHora: { sort: 'asc', nulls: 'last' } }];
    }
    if (ordenar === 'avaliacao') {
      return [{ avaliacaoMedia: 'desc' }, { avaliacaoQtd: 'desc' }];
    }
    return [{ avaliacaoQtd: 'desc' }, { nome: 'asc' }];
  }

  private paraDto(l: LinhaEquipamento): EquipmentListItemDto {
    return {
      id: l.id,
      slug: l.slug,
      nome: l.nome,
      marca: l.marca,
      modelo: l.modelo,
      categoria: { slug: l.category.slug, nome: l.category.nome },
      cidade: l.unit.cidade,
      uf: l.unit.uf,
      // Decimal do Prisma vira string no JSON; o cliente espera número.
      precoHora: l.precoHora === null ? null : Number(l.precoHora),
      precoDia: l.precoDia === null ? null : Number(l.precoDia),
      hasHour: l.hasHour,
      hasDay: l.hasDay,
      avaliacaoMedia: Number(l.avaliacaoMedia),
      avaliacaoQtd: l.avaliacaoQtd,
      imagem: l.images[0]?.path ?? null,
    };
  }
}
```

A conversão de `Decimal` para `number` no `paraDto` é obrigatória: sem ela o JSON sai com preço em string e o frontend faz `"120" < "45"` em comparação textual.

- [ ] **Step 5: Implementar controller e module**

`backend/src/modules/equipments/equipments.controller.ts`:
```ts
import { Controller, Get, Query } from '@nestjs/common';
import { EquipmentsService } from './equipments.service';
import { ListEquipmentsDto } from './dto/list-equipments.dto';

@Controller('equipments')
export class EquipmentsController {
  constructor(private readonly equipments: EquipmentsService) {}

  @Get()
  list(@Query() filtros: ListEquipmentsDto) {
    return this.equipments.list(filtros);
  }
}
```

`backend/src/modules/equipments/equipments.module.ts`:
```ts
import { Module } from '@nestjs/common';
import { EquipmentsController } from './equipments.controller';
import { EquipmentsService } from './equipments.service';

@Module({
  controllers: [EquipmentsController],
  providers: [EquipmentsService],
  exports: [EquipmentsService],
})
export class EquipmentsModule {}
```

Importe `EquipmentsModule` em `AppModule`.

- [ ] **Step 6: Rodar os testes — devem passar**

Run: `docker compose exec backend npm run test:e2e -- equipments-list`
Expected: PASS nos 10 casos.

- [ ] **Step 7: Commit**

```bash
git add backend/src/common backend/src/modules/equipments backend/src/app.module.ts backend/test/equipments-list.e2e-spec.ts
git commit -m "feat: listagem de equipamentos com filtros, ordenacao, paginacao e cache"
```

---

### Task 7: Detalhe do equipamento, arquivos estáticos e warm-up do cache

Fecha o backend da fase: a rota de detalhe, o `/uploads` servido e o cache já quente no boot.

**Files:**
- Create: `backend/src/modules/equipments/dto/equipment-detail.dto.ts`
- Create: `backend/src/warmup/warmup.service.ts`, `backend/src/warmup/warmup.module.ts`
- Create: `backend/test/equipment-detail.e2e-spec.ts`
- Modify: `backend/src/modules/equipments/{equipments.service.ts,equipments.controller.ts}`, `backend/src/main.ts`, `backend/src/app.module.ts`, `backend/package.json`

**Interfaces:**
- Consumes: tudo das Tasks 4–6.
- Produces:
  - `GET /api/equipments/:slug` → `EquipmentDetailDto` = campos do `EquipmentListItemDto` mais `{ descricao: string, specs: string[], quantidade: number, imagens: string[], unidade: { nome, cidade, uf, telefone } }`; 404 quando o slug não existe ou está inativo.
  - `GET /uploads/...` serve o volume de uploads.
  - `WarmupService` implementa `OnApplicationBootstrap` e popula categorias, unidades e a listagem padrão.

- [ ] **Step 1: Escrever o teste e2e (vai falhar)**

`backend/test/equipment-detail.e2e-spec.ts`:
```ts
import { Test } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import request from 'supertest';
import type Redis from 'ioredis';
import { AppModule } from '../src/app.module';
import { AllExceptionsFilter } from '../src/common/filters/all-exceptions.filter';
import { REDIS_CLIENT } from '../src/cache/cache.constants';

describe('Detalhe do equipamento (e2e)', () => {
  let app: INestApplication;
  let redis: Redis;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({ imports: [AppModule] }).compile();
    app = moduleRef.createNestApplication();
    app.setGlobalPrefix('api');
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
    app.useGlobalFilters(new AllExceptionsFilter());
    await app.init();
    redis = app.get(REDIS_CLIENT);
  });

  afterAll(async () => { await app.close(); });
  beforeEach(async () => { await redis.flushdb(); });

  it('devolve o equipamento completo pelo slug', async () => {
    const res = await request(app.getHttpServer()).get('/api/equipments/andaime').expect(200);

    expect(res.body).toMatchObject({
      slug: 'andaime',
      quantidade: 40,
      precoHora: null,
      hasHour: false,
      unidade: { cidade: 'Realeza', uf: 'PR' },
    });
    expect(res.body.specs.length).toBeGreaterThan(0);
    expect(res.body.imagens).toContain('/uploads/seed/andaime.svg');
    expect(res.body.unidade.telefone).toBe('(46) 99917-0071');
  });

  it('slug inexistente devolve 404 com mensagem em portugues', async () => {
    const res = await request(app.getHttpServer()).get('/api/equipments/nao-existe').expect(404);
    expect(res.body.message.join(' ')).toMatch(/não encontrado/i);
  });

  it('a segunda leitura vem do cache', async () => {
    await request(app.getHttpServer()).get('/api/equipments/andaime').expect(200);
    expect(await redis.get('eq:v0:slug:andaime')).not.toBeNull();
  });

  it('o warm-up deixa categorias e listagem padrao no cache logo apos o boot', async () => {
    await redis.flushdb();
    const warmup = app.get(await import('../src/warmup/warmup.service').then((m) => m.WarmupService));
    await warmup.onApplicationBootstrap();

    expect(await redis.get('cat:v0:all')).not.toBeNull();
    expect(await redis.get('unit:v0:all')).not.toBeNull();
    expect((await redis.keys('eq:v0:list:*')).length).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 2: Rodar e confirmar a falha**

Run: `docker compose exec backend npm run test:e2e -- equipment-detail`
Expected: FAIL — 404 em `/api/equipments/andaime` (capturado pela rota de listagem ou inexistente).

- [ ] **Step 3: Implementar o DTO e o método de detalhe**

`backend/src/modules/equipments/dto/equipment-detail.dto.ts`:
```ts
import { EquipmentListItemDto } from './equipment-list-item.dto';

export class EquipmentDetailDto extends EquipmentListItemDto {
  descricao!: string;
  specs!: string[];
  quantidade!: number;
  imagens!: string[];
  unidade!: { nome: string; cidade: string; uf: string; telefone: string | null };
}
```

Em `equipments.service.ts`, acrescente:
```ts
  async findBySlug(slug: string): Promise<EquipmentDetailDto> {
    return this.cache.getOrSet(CACHE_NS.EQUIPMENTS, `slug:${slug}`, CACHE_TTL.EQUIPMENTS, async () => {
      const e = await this.prisma.equipment.findFirst({
        where: { slug, ativo: true },
        select: {
          ...SELECAO_LISTA,
          descricao: true,
          specs: true,
          quantidade: true,
          images: { select: { path: true }, orderBy: { ordem: 'asc' } },
          unit: { select: { nome: true, cidade: true, uf: true, telefone: true } },
        },
      });

      if (!e) {
        throw new NotFoundException(`Equipamento "${slug}" não encontrado.`);
      }

      return {
        ...this.paraDto({ ...e, images: e.images.slice(0, 1) } as LinhaEquipamento),
        descricao: e.descricao,
        specs: e.specs,
        quantidade: e.quantidade,
        imagens: e.images.map((i) => i.path),
        unidade: e.unit,
      };
    });
  }
```
Importe `NotFoundException` de `@nestjs/common`.

Uma exceção lançada dentro do `loader` **não** é cacheada: o `getOrSet` só grava depois que o loader retorna, então o 404 não fica preso no Redis.

Em `equipments.controller.ts`:
```ts
  @Get(':slug')
  findBySlug(@Param('slug') slug: string) {
    return this.equipments.findBySlug(slug);
  }
```
Importe `Param`. O método `@Get()` sem parâmetro deve vir **antes** de `@Get(':slug')` no arquivo.

- [ ] **Step 4: Servir `/uploads` estaticamente**

```bash
docker compose exec backend npm install @nestjs/serve-static
```

Em `app.module.ts`:
```ts
import { ServeStaticModule } from '@nestjs/serve-static';
import { join } from 'node:path';

// dentro de imports:
    ServeStaticModule.forRoot({
      rootPath: join(process.cwd(), 'uploads'),
      serveRoot: '/uploads',
      serveStaticOptions: { index: false, fallthrough: false },
    }),
```

Como o prefixo global `/api` não se aplica ao `ServeStaticModule`, a URL final é `http://localhost:3000/uploads/seed/andaime.svg` — igual ao `path` gravado no banco.

- [ ] **Step 5: Implementar o warm-up**

`backend/src/warmup/warmup.service.ts`:
```ts
import { Injectable, Logger, OnApplicationBootstrap } from '@nestjs/common';
import { CategoriesService } from '../modules/categories/categories.service';
import { UnitsService } from '../modules/units/units.service';
import { EquipmentsService } from '../modules/equipments/equipments.service';
import { ListEquipmentsDto } from '../modules/equipments/dto/list-equipments.dto';

@Injectable()
export class WarmupService implements OnApplicationBootstrap {
  private readonly logger = new Logger(WarmupService.name);

  constructor(
    private readonly categories: CategoriesService,
    private readonly units: UnitsService,
    private readonly equipments: EquipmentsService,
  ) {}

  /**
   * Popula o cache no boot para que a primeira visita já seja servida pelo Redis.
   * Falhas aqui são registradas e ignoradas: o warm-up é otimização, não pré-requisito.
   */
  async onApplicationBootstrap(): Promise<void> {
    const padrao = new ListEquipmentsDto();
    padrao.page = 1;
    padrao.limit = 12;
    padrao.ordenar = 'relevancia';

    try {
      await Promise.all([
        this.categories.findAll(),
        this.units.findAll(),
        this.equipments.list(padrao),
      ]);
      this.logger.log('Cache aquecido: categorias, unidades e listagem padrão.');
    } catch (err) {
      this.logger.warn(`Warm-up do cache falhou, seguindo sem ele: ${String(err)}`);
    }
  }
}
```

`backend/src/warmup/warmup.module.ts`:
```ts
import { Module } from '@nestjs/common';
import { WarmupService } from './warmup.service';
import { CategoriesModule } from '../modules/categories/categories.module';
import { UnitsModule } from '../modules/units/units.module';
import { EquipmentsModule } from '../modules/equipments/equipments.module';

@Module({
  imports: [CategoriesModule, UnitsModule, EquipmentsModule],
  providers: [WarmupService],
})
export class WarmupModule {}
```

Importe `WarmupModule` em `AppModule`.

- [ ] **Step 6: Publicar a documentação Swagger**

```bash
docker compose exec backend npm install @nestjs/swagger
```

Em `main.ts`, antes do `listen`:
```ts
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';

// dentro de bootstrap(), após os pipes e filtros:
  const config = new DocumentBuilder()
    .setTitle('LocaTudo API')
    .setDescription('Marketplace de aluguel de equipamentos para construção.')
    .setVersion('1.0')
    .build();
  SwaggerModule.setup('api/docs', app, SwaggerModule.createDocument(app, config));
```

Como os DTOs usam `class-validator`, habilite o plugin em `backend/nest-cli.json` para que os
campos apareçam na documentação sem anotação manual:
```json
{
  "compilerOptions": {
    "plugins": ["@nestjs/swagger"]
  }
}
```
(preserve as chaves já existentes do arquivo, como `"sourceRoot"` e `"collection"`).

- [ ] **Step 7: Rodar tudo**

```bash
docker compose restart backend
docker compose exec backend npm test
docker compose exec backend npm run test:e2e
curl -s localhost:3000/api/equipments/andaime | head -c 200
curl -s -o /dev/null -w "%{http_code}" localhost:3000/uploads/seed/andaime.svg
curl -s -o /dev/null -w "%{http_code}" localhost:3000/api/docs
```
Expected: testes PASS; detalhe em JSON; `200` no SVG e em `/api/docs`.

- [ ] **Step 8: Verificar a degradação sem Redis**

```bash
docker compose stop redis
curl -s localhost:3000/api/equipments | head -c 120
curl -s localhost:3000/api/health
docker compose start redis
```
Expected: a listagem responde normalmente (vinda do Postgres) e o health check mostra `"redis":"indisponivel"` com `"status":"degradado"` apenas se o banco também cair — com o banco de pé, `status` continua `ok`. Se a listagem falhar, o try/catch da Task 4 está incompleto — corrija antes de seguir.

- [ ] **Step 9: Commit**

```bash
git add backend/src backend/test/equipment-detail.e2e-spec.ts backend/package.json backend/nest-cli.json
git commit -m "feat: detalhe do equipamento, uploads estaticos, warm-up do cache e Swagger"
```

---

### Task 8: Frontend React + Vite em Docker, com a Home

**Files:**
- Create: `frontend/` (scaffolding Vite), `frontend/Dockerfile.dev`, `frontend/vite.config.ts`
- Create: `frontend/src/styles/{tokens.css,global.css}`
- Create: `frontend/src/api/{client.ts,catalog.ts}`
- Create: `frontend/src/components/{Header.tsx,CategoryCard.tsx,EquipmentCard.tsx}`
- Create: `frontend/src/pages/HomePage.tsx`, `frontend/src/App.tsx`, `frontend/src/main.tsx`
- Create: `frontend/src/api/client.test.ts`
- Modify: `docker-compose.yml`
- Copy: `assets/logo-locatudo.png` → `frontend/public/logo-locatudo.png`

**Interfaces:**
- Consumes: `GET /api/categories`, `GET /api/units`, `GET /api/equipments` (Tasks 5–7).
- Produces:
  - `buildQuery(filtros: Record<string, unknown>): string` em `src/api/client.ts`
  - `apiGet<T>(path: string): Promise<T>`
  - Tipos `Category`, `Equipment`, `Paginated<T>` em `src/api/catalog.ts`
  - `fetchCategories()`, `fetchEquipments(filtros)`
  - Rotas `/` (Home) e `/resultados` (Task 9)

- [ ] **Step 1: Gerar o projeto e instalar dependências**

```bash
docker run --rm -v "$PWD":/w -w /w node:20-alpine \
  npx -y create-vite@latest frontend -- --template react-ts
docker run --rm -v "$PWD/frontend":/app -w /app node:20-alpine \
  sh -c "npm install && npm install react-router-dom && npm install -D vitest @testing-library/react @testing-library/jest-dom jsdom"
cp assets/logo-locatudo.png frontend/public/logo-locatudo.png
```

Em `frontend/package.json`, acrescente `"test": "vitest run"` aos scripts.

- [ ] **Step 2: Escrever o teste do construtor de querystring (vai falhar)**

`frontend/src/api/client.test.ts`:
```ts
import { describe, expect, it } from 'vitest';
import { buildQuery } from './client';

describe('buildQuery', () => {
  it('monta a querystring a partir dos filtros preenchidos', () => {
    expect(buildQuery({ q: 'betoneira', page: 2 })).toBe('?q=betoneira&page=2');
  });

  it('descarta valores vazios, nulos e indefinidos', () => {
    expect(buildQuery({ q: '', categoria: undefined, cidade: null, page: 1 })).toBe('?page=1');
  });

  it('devolve string vazia quando nao ha filtro', () => {
    expect(buildQuery({})).toBe('');
  });

  it('escapa caracteres especiais', () => {
    expect(buildQuery({ q: 'martelete & broca' })).toBe('?q=martelete+%26+broca');
  });
});
```

- [ ] **Step 3: Rodar e confirmar a falha**

Run: `docker compose run --rm frontend npm test` (após o Step 6; por ora `docker run --rm -v "$PWD/frontend":/app -w /app node:20-alpine npm test`)
Expected: FAIL — `Cannot find module './client'`.

- [ ] **Step 4: Implementar a camada de API**

`frontend/src/api/client.ts`:
```ts
const BASE = '/api';

export function buildQuery(filtros: Record<string, unknown>): string {
  const params = new URLSearchParams();
  for (const [chave, valor] of Object.entries(filtros)) {
    if (valor === undefined || valor === null || valor === '') continue;
    params.append(chave, String(valor));
  }
  const qs = params.toString();
  return qs ? `?${qs}` : '';
}

export async function apiGet<T>(path: string): Promise<T> {
  const res = await fetch(`${BASE}${path}`);
  if (!res.ok) {
    const corpo = await res.json().catch(() => null);
    const mensagem = corpo?.message?.join?.(' ') ?? 'Não foi possível carregar os dados.';
    throw new Error(mensagem);
  }
  return (await res.json()) as T;
}
```

`frontend/src/api/catalog.ts`:
```ts
import { apiGet, buildQuery } from './client';

export interface Category {
  slug: string;
  nome: string;
  mono: string;
  ordem: number;
  quantidadeEquipamentos: number;
}

export interface Equipment {
  id: string;
  slug: string;
  nome: string;
  marca: string;
  modelo: string;
  categoria: { slug: string; nome: string };
  cidade: string;
  uf: string;
  precoHora: number | null;
  precoDia: number | null;
  hasHour: boolean;
  hasDay: boolean;
  avaliacaoMedia: number;
  avaliacaoQtd: number;
  imagem: string | null;
}

export interface Paginated<T> {
  data: T[];
  meta: { page: number; limit: number; total: number; totalPages: number };
}

export interface EquipmentFilters {
  q?: string;
  categoria?: string;
  cidade?: string;
  precoMax?: number;
  ordenar?: 'relevancia' | 'preco' | 'avaliacao';
  page?: number;
  limit?: number;
}

export const fetchCategories = () => apiGet<Category[]>('/categories');

export const fetchEquipments = (filtros: EquipmentFilters = {}) =>
  apiGet<Paginated<Equipment>>(`/equipments${buildQuery(filtros as Record<string, unknown>)}`);

export function formatarMoeda(valor: number): string {
  return valor.toLocaleString('pt-BR', { style: 'currency', currency: 'BRL' });
}

export function precoResumido(e: Equipment): string {
  if (e.hasDay && e.precoDia !== null) return `${formatarMoeda(e.precoDia)} / dia`;
  if (e.precoHora !== null) return `${formatarMoeda(e.precoHora)} / hora`;
  return 'Sob consulta';
}
```

- [ ] **Step 5: Criar os tokens visuais**

`frontend/src/styles/tokens.css`:
```css
:root {
  --navy: #14205A;
  --navy-2: #1D2B63;
  --amarelo: #FFCB05;
  --vermelho: #E4032E;
  --texto: #14203D;
  --texto-cinza: #6B7280;
  --fundo: #F5F5F1;
  --branco: #FFFFFF;
  --raio: 14px;
  --sombra: 0 2px 12px rgba(20, 32, 90, .08);
  --fonte-titulo: 'Poppins', sans-serif;
  --fonte-corpo: 'Plus Jakarta Sans', sans-serif;
}
```

`frontend/src/styles/global.css`:
```css
@import './tokens.css';

* { box-sizing: border-box; }

body {
  margin: 0;
  background: var(--fundo);
  color: var(--texto);
  font-family: var(--fonte-corpo);
}

h1, h2, h3 { font-family: var(--fonte-titulo); font-weight: 800; }

.lt-container { max-width: 1180px; margin: 0 auto; padding: 0 20px; }
.lt-grid { display: grid; gap: 18px; grid-template-columns: repeat(auto-fill, minmax(240px, 1fr)); }
.lt-card { background: var(--branco); border-radius: var(--raio); box-shadow: var(--sombra); overflow: hidden; }
```

Em `frontend/index.html`, dentro do `<head>`, os fontes do protótipo:
```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Poppins:wght@700;800;900&family=Plus+Jakarta+Sans:wght@400;500;600;700;800&display=swap" rel="stylesheet">
```

- [ ] **Step 6: Implementar componentes e Home**

`frontend/src/components/Header.tsx`:
```tsx
import { FormEvent, useState } from 'react';
import { useNavigate } from 'react-router-dom';

export function Header() {
  const [termo, setTermo] = useState('');
  const navigate = useNavigate();

  function buscar(e: FormEvent) {
    e.preventDefault();
    navigate(`/resultados?q=${encodeURIComponent(termo)}`);
  }

  return (
    <header style={{ background: 'var(--navy)', padding: '14px 0' }}>
      <div className="lt-container" style={{ display: 'flex', gap: 20, alignItems: 'center' }}>
        <a href="/" aria-label="LocaTudo Constru&CIA">
          <img src="/logo-locatudo.png" alt="LocaTudo Constru&CIA" style={{ height: 42 }} />
        </a>
        <form onSubmit={buscar} style={{ flex: 1, display: 'flex', gap: 8 }}>
          <input
            aria-label="Buscar equipamento"
            placeholder="Busque por betoneira, gerador, andaime, martelete..."
            value={termo}
            onChange={(e) => setTermo(e.target.value)}
            style={{ flex: 1, padding: '12px 16px', borderRadius: 999, border: 'none', fontFamily: 'var(--fonte-corpo)' }}
          />
          <button type="submit" style={{ background: 'var(--amarelo)', color: 'var(--navy)', border: 'none', borderRadius: 999, padding: '12px 24px', fontWeight: 700, cursor: 'pointer' }}>
            Buscar
          </button>
        </form>
      </div>
    </header>
  );
}
```

`frontend/src/components/CategoryCard.tsx`:
```tsx
import { Link } from 'react-router-dom';
import type { Category } from '../api/catalog';

export function CategoryCard({ categoria }: { categoria: Category }) {
  return (
    <Link to={`/resultados?categoria=${categoria.slug}`} className="lt-card" style={{ padding: 18, textDecoration: 'none', color: 'var(--texto)', display: 'block' }}>
      <div style={{ width: 44, height: 44, borderRadius: 12, background: 'var(--navy)', color: 'var(--amarelo)', display: 'grid', placeItems: 'center', fontFamily: 'var(--fonte-titulo)', fontWeight: 800 }}>
        {categoria.mono}
      </div>
      <h3 style={{ margin: '12px 0 4px', fontSize: 16 }}>{categoria.nome}</h3>
      <span style={{ color: 'var(--texto-cinza)', fontSize: 14 }}>
        {categoria.quantidadeEquipamentos} equipamentos
      </span>
    </Link>
  );
}
```

`frontend/src/components/EquipmentCard.tsx`:
```tsx
import { Link } from 'react-router-dom';
import { precoResumido, type Equipment } from '../api/catalog';

export function EquipmentCard({ equipamento }: { equipamento: Equipment }) {
  return (
    <Link to={`/equipamento/${equipamento.slug}`} className="lt-card" style={{ textDecoration: 'none', color: 'var(--texto)', display: 'block' }}>
      {equipamento.imagem && (
        <img src={equipamento.imagem} alt={equipamento.nome} style={{ width: '100%', height: 160, objectFit: 'cover' }} />
      )}
      <div style={{ padding: 16 }}>
        <span style={{ fontSize: 12, color: 'var(--texto-cinza)' }}>{equipamento.categoria.nome}</span>
        <h3 style={{ margin: '6px 0', fontSize: 16 }}>{equipamento.nome}</h3>
        <div style={{ fontSize: 13, color: 'var(--texto-cinza)' }}>
          {equipamento.marca} · {equipamento.cidade} - {equipamento.uf}
        </div>
        <div style={{ marginTop: 10, fontFamily: 'var(--fonte-titulo)', fontWeight: 800, color: 'var(--navy)' }}>
          {precoResumido(equipamento)}
        </div>
      </div>
    </Link>
  );
}
```

`frontend/src/pages/HomePage.tsx`:
```tsx
import { useEffect, useState } from 'react';
import { fetchCategories, fetchEquipments, type Category, type Equipment } from '../api/catalog';
import { CategoryCard } from '../components/CategoryCard';
import { EquipmentCard } from '../components/EquipmentCard';

export function HomePage() {
  const [categorias, setCategorias] = useState<Category[]>([]);
  const [destaques, setDestaques] = useState<Equipment[]>([]);
  const [erro, setErro] = useState<string | null>(null);
  const [carregando, setCarregando] = useState(true);

  useEffect(() => {
    Promise.all([fetchCategories(), fetchEquipments({ limit: 4, ordenar: 'avaliacao' })])
      .then(([cats, eqs]) => { setCategorias(cats); setDestaques(eqs.data); })
      .catch((e: Error) => setErro(e.message))
      .finally(() => setCarregando(false));
  }, []);

  if (carregando) return <main className="lt-container"><p>Carregando catálogo...</p></main>;
  if (erro) return <main className="lt-container"><p role="alert">{erro}</p></main>;

  return (
    <main className="lt-container" style={{ paddingBottom: 60 }}>
      <section style={{ padding: '48px 0' }}>
        <h1 style={{ fontSize: 40, margin: 0, color: 'var(--navy)' }}>Alugue equipamentos para sua obra</h1>
        <p style={{ color: 'var(--texto-cinza)', fontSize: 18 }}>
          Capanema e Realeza, sudoeste do Paraná. Retirada no mesmo dia.
        </p>
      </section>

      <section id="lt-categories">
        <h2 style={{ color: 'var(--navy)' }}>Categorias</h2>
        <div className="lt-grid">
          {categorias.map((c) => <CategoryCard key={c.slug} categoria={c} />)}
        </div>
      </section>

      <section style={{ marginTop: 48 }}>
        <h2 style={{ color: 'var(--navy)' }}>Mais alugados</h2>
        <div className="lt-grid">
          {destaques.map((e) => <EquipmentCard key={e.id} equipamento={e} />)}
        </div>
      </section>
    </main>
  );
}
```

`frontend/src/App.tsx`:
```tsx
import { BrowserRouter, Route, Routes } from 'react-router-dom';
import { Header } from './components/Header';
import { HomePage } from './pages/HomePage';
import './styles/global.css';

export default function App() {
  return (
    <BrowserRouter>
      <Header />
      <Routes>
        <Route path="/" element={<HomePage />} />
      </Routes>
    </BrowserRouter>
  );
}
```

`frontend/src/main.tsx`:
```tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';

createRoot(document.getElementById('root')!).render(
  <StrictMode><App /></StrictMode>,
);
```

Apague `frontend/src/App.css` e `frontend/src/index.css` gerados pelo template.

- [ ] **Step 7: Configurar Vite, Vitest e Docker**

`frontend/vite.config.ts`:
```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    host: '0.0.0.0',
    port: 5173,
    // O container do frontend fala com o backend pelo nome do serviço.
    proxy: {
      '/api': { target: 'http://backend:3000', changeOrigin: true },
      '/uploads': { target: 'http://backend:3000', changeOrigin: true },
    },
    watch: { usePolling: true },
  },
  test: { environment: 'jsdom', globals: true },
});
```

Adicione `/// <reference types="vitest" />` na primeira linha do arquivo.

`frontend/Dockerfile.dev`:
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
CMD ["npm", "run", "dev"]
```

No `docker-compose.yml`:
```yaml
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    ports: ["5173:5173"]
    volumes:
      - ./frontend:/app
      - /app/node_modules
    depends_on: [backend]
```

- [ ] **Step 8: Subir e verificar**

```bash
docker compose up -d --build
docker compose exec frontend npm test
```
Expected: 4 testes PASS. Abra `http://localhost:5173` — a Home mostra as 8 categorias e 4 destaques com imagem.

- [ ] **Step 9: Commit**

```bash
git add frontend docker-compose.yml
git commit -m "feat: frontend React+Vite em Docker com a Home do catalogo"
```

---

### Task 9: Tela de Resultados com filtros

**Files:**
- Create: `frontend/src/pages/ResultsPage.tsx`, `frontend/src/components/FilterPanel.tsx`
- Create: `frontend/src/hooks/useEquipmentFilters.ts`
- Create: `frontend/src/pages/ResultsPage.test.tsx`
- Modify: `frontend/src/App.tsx`, `frontend/src/api/catalog.ts`

**Interfaces:**
- Consumes: `fetchEquipments`, `fetchCategories`, `fetchUnits`, tipos de `src/api/catalog.ts` (Task 8).
- Produces: rota `/resultados`; `useEquipmentFilters()` devolvendo `{ filtros, atualizar(parcial), limpar() }` sincronizado com a querystring; `fetchUnits(): Promise<Unit[]>`.

- [ ] **Step 1: Escrever o teste de componente (vai falhar)**

`frontend/src/pages/ResultsPage.test.tsx`:
```tsx
import { render, screen, waitFor } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { ResultsPage } from './ResultsPage';

const respostas: Record<string, unknown> = {
  '/api/categories': [
    { slug: 'concreto', nome: 'Equipamentos para concreto', mono: 'CO', ordem: 6, quantidadeEquipamentos: 1 },
  ],
  '/api/units': [{ id: '1', nome: 'LocaTudo Capanema', cidade: 'Capanema', uf: 'PR', telefone: null }],
};

const equipamento = {
  id: '1', slug: 'betoneira', nome: 'Betoneira 400L', marca: 'CSM', modelo: 'RentalForce 400',
  categoria: { slug: 'concreto', nome: 'Equipamentos para concreto' },
  cidade: 'Capanema', uf: 'PR', precoHora: 18, precoDia: 120, hasHour: true, hasDay: true,
  avaliacaoMedia: 4.9, avaliacaoQtd: 128, imagem: '/uploads/seed/betoneira.svg',
};

beforeEach(() => {
  vi.stubGlobal('fetch', vi.fn(async (url: string) => {
    const path = url.split('?')[0];
    const corpo = path === '/api/equipments'
      ? { data: [equipamento], meta: { page: 1, limit: 12, total: 1, totalPages: 1 } }
      : respostas[path];
    return { ok: true, json: async () => corpo } as Response;
  }));
});

describe('ResultsPage', () => {
  it('lista os equipamentos retornados pela API', async () => {
    render(<MemoryRouter initialEntries={['/resultados?q=betoneira']}><ResultsPage /></MemoryRouter>);
    expect(await screen.findByText('Betoneira 400L')).toBeInTheDocument();
    expect(screen.getByText(/1 equipamento encontrado/i)).toBeInTheDocument();
  });

  it('envia os filtros da querystring para a API', async () => {
    render(<MemoryRouter initialEntries={['/resultados?categoria=concreto&cidade=Capanema']}><ResultsPage /></MemoryRouter>);
    await waitFor(() => {
      const chamadas = (fetch as unknown as ReturnType<typeof vi.fn>).mock.calls.map((c) => String(c[0]));
      expect(chamadas.some((u) => u.includes('categoria=concreto') && u.includes('cidade=Capanema'))).toBe(true);
    });
  });

  it('mostra estado vazio quando nao ha resultado', async () => {
    vi.stubGlobal('fetch', vi.fn(async (url: string) => {
      const path = url.split('?')[0];
      const corpo = path === '/api/equipments'
        ? { data: [], meta: { page: 1, limit: 12, total: 0, totalPages: 1 } }
        : respostas[path];
      return { ok: true, json: async () => corpo } as Response;
    }));
    render(<MemoryRouter initialEntries={['/resultados?q=xyz']}><ResultsPage /></MemoryRouter>);
    expect(await screen.findByText(/nenhum equipamento encontrado/i)).toBeInTheDocument();
  });
});
```

Crie `frontend/src/setupTests.ts` com `import '@testing-library/jest-dom';` e aponte `test.setupFiles: './src/setupTests.ts'` em `vite.config.ts`.

- [ ] **Step 2: Rodar e confirmar a falha**

Run: `docker compose exec frontend npm test`
Expected: FAIL — `Cannot find module './ResultsPage'`.

- [ ] **Step 3: Acrescentar `fetchUnits` à camada de API**

Em `frontend/src/api/catalog.ts`:
```ts
export interface Unit {
  id: string;
  nome: string;
  cidade: string;
  uf: string;
  telefone: string | null;
}

export const fetchUnits = () => apiGet<Unit[]>('/units');
```

- [ ] **Step 4: Implementar o hook de filtros**

`frontend/src/hooks/useEquipmentFilters.ts`:
```ts
import { useCallback, useMemo } from 'react';
import { useSearchParams } from 'react-router-dom';
import type { EquipmentFilters } from '../api/catalog';

export function useEquipmentFilters() {
  const [params, setParams] = useSearchParams();

  const filtros = useMemo<EquipmentFilters>(() => ({
    q: params.get('q') ?? undefined,
    categoria: params.get('categoria') ?? undefined,
    cidade: params.get('cidade') ?? undefined,
    precoMax: params.get('precoMax') ? Number(params.get('precoMax')) : undefined,
    ordenar: (params.get('ordenar') as EquipmentFilters['ordenar']) ?? 'relevancia',
    page: params.get('page') ? Number(params.get('page')) : 1,
  }), [params]);

  const atualizar = useCallback((parcial: Partial<EquipmentFilters>) => {
    const proximos = new URLSearchParams(params);
    for (const [chave, valor] of Object.entries(parcial)) {
      if (valor === undefined || valor === '' || valor === null) proximos.delete(chave);
      else proximos.set(chave, String(valor));
    }
    // Qualquer mudança de filtro volta para a primeira página.
    if (!('page' in parcial)) proximos.delete('page');
    setParams(proximos);
  }, [params, setParams]);

  const limpar = useCallback(() => setParams(new URLSearchParams()), [setParams]);

  return { filtros, atualizar, limpar };
}
```

- [ ] **Step 5: Implementar o painel de filtros**

`frontend/src/components/FilterPanel.tsx`:
```tsx
import type { Category, EquipmentFilters, Unit } from '../api/catalog';

interface Props {
  categorias: Category[];
  unidades: Unit[];
  filtros: EquipmentFilters;
  onChange: (parcial: Partial<EquipmentFilters>) => void;
  onLimpar: () => void;
}

export function FilterPanel({ categorias, unidades, filtros, onChange, onLimpar }: Props) {
  return (
    <aside className="lt-card" style={{ padding: 18, alignSelf: 'start' }}>
      <h2 style={{ fontSize: 18, marginTop: 0 }}>Filtros</h2>

      <label style={{ display: 'block', marginBottom: 14 }}>
        <span style={{ fontSize: 13, color: 'var(--texto-cinza)' }}>Categoria</span>
        <select
          value={filtros.categoria ?? ''}
          onChange={(e) => onChange({ categoria: e.target.value || undefined })}
          style={{ width: '100%', padding: 10, borderRadius: 10 }}
        >
          <option value="">Todas</option>
          {categorias.map((c) => <option key={c.slug} value={c.slug}>{c.nome}</option>)}
        </select>
      </label>

      <label style={{ display: 'block', marginBottom: 14 }}>
        <span style={{ fontSize: 13, color: 'var(--texto-cinza)' }}>Cidade</span>
        <select
          value={filtros.cidade ?? ''}
          onChange={(e) => onChange({ cidade: e.target.value || undefined })}
          style={{ width: '100%', padding: 10, borderRadius: 10 }}
        >
          <option value="">Todas</option>
          {unidades.map((u) => <option key={u.id} value={u.cidade}>{u.cidade} - {u.uf}</option>)}
        </select>
      </label>

      <label style={{ display: 'block', marginBottom: 14 }}>
        <span style={{ fontSize: 13, color: 'var(--texto-cinza)' }}>
          Preço máximo por dia: {filtros.precoMax ?? 200}
        </span>
        <input
          type="range" min={20} max={200} step={10}
          value={filtros.precoMax ?? 200}
          onChange={(e) => onChange({ precoMax: Number(e.target.value) })}
          style={{ width: '100%' }}
        />
      </label>

      <button onClick={onLimpar} style={{ background: 'none', border: '1px solid var(--navy)', color: 'var(--navy)', borderRadius: 999, padding: '8px 18px', cursor: 'pointer' }}>
        Limpar filtros
      </button>
    </aside>
  );
}
```

- [ ] **Step 6: Implementar a página de resultados**

`frontend/src/pages/ResultsPage.tsx`:
```tsx
import { useEffect, useState } from 'react';
import {
  fetchCategories, fetchEquipments, fetchUnits,
  type Category, type Equipment, type Unit,
} from '../api/catalog';
import { EquipmentCard } from '../components/EquipmentCard';
import { FilterPanel } from '../components/FilterPanel';
import { useEquipmentFilters } from '../hooks/useEquipmentFilters';

export function ResultsPage() {
  const { filtros, atualizar, limpar } = useEquipmentFilters();
  const [categorias, setCategorias] = useState<Category[]>([]);
  const [unidades, setUnidades] = useState<Unit[]>([]);
  const [equipamentos, setEquipamentos] = useState<Equipment[]>([]);
  const [total, setTotal] = useState(0);
  const [carregando, setCarregando] = useState(true);
  const [erro, setErro] = useState<string | null>(null);

  useEffect(() => {
    Promise.all([fetchCategories(), fetchUnits()])
      .then(([cats, uns]) => { setCategorias(cats); setUnidades(uns); })
      .catch((e: Error) => setErro(e.message));
  }, []);

  useEffect(() => {
    setCarregando(true);
    fetchEquipments(filtros)
      .then((r) => { setEquipamentos(r.data); setTotal(r.meta.total); setErro(null); })
      .catch((e: Error) => setErro(e.message))
      .finally(() => setCarregando(false));
  }, [filtros]);

  return (
    <main className="lt-container" style={{ display: 'grid', gridTemplateColumns: '260px 1fr', gap: 24, padding: '28px 20px 60px' }}>
      <FilterPanel
        categorias={categorias}
        unidades={unidades}
        filtros={filtros}
        onChange={atualizar}
        onLimpar={limpar}
      />

      <section>
        <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: 16 }}>
          {/* Uma única expressão, não `{total} {texto}`: interpolações separadas viram nós de
              texto distintos no DOM e quebram a busca por texto completo nos testes. */}
          <span style={{ color: 'var(--texto-cinza)' }}>
            {`${total} ${total === 1 ? 'equipamento encontrado' : 'equipamentos encontrados'}`}
          </span>
          <select
            aria-label="Ordenar por"
            value={filtros.ordenar}
            onChange={(e) => atualizar({ ordenar: e.target.value as typeof filtros.ordenar })}
            style={{ padding: 8, borderRadius: 10 }}
          >
            <option value="relevancia">Relevância</option>
            <option value="preco">Menor preço</option>
            <option value="avaliacao">Melhor avaliação</option>
          </select>
        </div>

        {erro && <p role="alert">{erro}</p>}
        {carregando && <p>Carregando equipamentos...</p>}
        {!carregando && !erro && equipamentos.length === 0 && (
          <p>Nenhum equipamento encontrado com esses filtros.</p>
        )}

        <div className="lt-grid">
          {equipamentos.map((e) => <EquipmentCard key={e.id} equipamento={e} />)}
        </div>
      </section>
    </main>
  );
}
```

Em `App.tsx`, acrescente a rota:
```tsx
        <Route path="/resultados" element={<ResultsPage />} />
```

- [ ] **Step 7: Rodar os testes — devem passar**

Run: `docker compose exec frontend npm test`
Expected: PASS nos 7 casos (4 de `client.test.ts` + 3 de `ResultsPage.test.tsx`).

- [ ] **Step 8: Verificar no navegador**

Abra `http://localhost:5173`, busque "betoneira" no cabeçalho, troque categoria e cidade no painel e confirme que a URL reflete os filtros e a lista muda.

- [ ] **Step 9: Commit**

```bash
git add frontend/src
git commit -m "feat: tela de resultados com filtros sincronizados na querystring"
```

---

### Task 10: Build de produção e documentação

**Files:**
- Create: `backend/Dockerfile`, `frontend/Dockerfile`, `frontend/nginx.conf`, `docker-compose.prod.yml`, `README.md`
- Modify: `.env.example`

**Interfaces:**
- Consumes: todo o resultado das Tasks 1–9.
- Produces: `docker compose -f docker-compose.prod.yml up --build` servindo a aplicação em `http://localhost` com a API sob `/api`.

- [ ] **Step 1: Dockerfile de produção do backend**

`backend/Dockerfile`:
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npx prisma generate && npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
RUN apk add --no-cache bash
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules/.prisma ./node_modules/.prisma
COPY prisma ./prisma
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh
EXPOSE 3000
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["node", "dist/main"]
```

- [ ] **Step 2: Dockerfile de produção e Nginx do frontend**

`frontend/Dockerfile`:
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:1.27-alpine AS runtime
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

`frontend/nginx.conf`:
```nginx
server {
  listen 80;
  server_name _;
  root /usr/share/nginx/html;

  # SPA: qualquer rota desconhecida cai no index.html
  location / {
    try_files $uri $uri/ /index.html;
  }

  location /api/ {
    proxy_pass http://backend:3000/api/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }

  location /uploads/ {
    proxy_pass http://backend:3000/uploads/;
  }

  gzip on;
  gzip_types text/css application/javascript image/svg+xml;
}
```

- [ ] **Step 3: Compose de produção**

`docker-compose.prod.yml`:
```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    volumes: ["redisdata:/data"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10
    restart: unless-stopped

  backend:
    build: { context: ./backend }
    environment:
      NODE_ENV: production
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
      PORT: 3000
    volumes: ["uploads:/app/uploads"]
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_healthy }
    restart: unless-stopped

  frontend:
    build: { context: ./frontend }
    ports: ["80:80"]
    depends_on: [backend]
    restart: unless-stopped

volumes:
  pgdata:
  redisdata:
  uploads:
```

Nenhum dos serviços de dados expõe porta ao host — só o `frontend` na 80.

O seed **não** roda em produção (o entrypoint da Task 3 o restringe a `NODE_ENV != production`). Para popular a primeira instalação:
`docker compose -f docker-compose.prod.yml exec backend npx prisma db seed`

- [ ] **Step 4: Escrever o README**

`README.md`:
```markdown
# LocaTudo Constru&CIA

Marketplace de aluguel de equipamentos para construção — Capanema e Realeza, PR.

## Stack

Backend NestJS + Prisma · PostgreSQL 16 · Redis 7 · Frontend React + Vite · Docker Compose.

## Desenvolvimento

```bash
cp .env.example .env
docker compose up --build
```

- Frontend: http://localhost:5173
- API: http://localhost:3000/api
- Health check: http://localhost:3000/api/health
- Postgres: localhost:5432 (usuário e senha no `.env`)

Migrations e seed rodam automaticamente no boot do backend.

## Testes

```bash
docker compose exec backend npm test          # unitários
docker compose exec backend npm run test:e2e  # e2e contra o serviço db-test
docker compose exec frontend npm test         # frontend
```

## Produção

```bash
docker compose -f docker-compose.prod.yml up --build -d
docker compose -f docker-compose.prod.yml exec backend npx prisma db seed  # só na primeira vez
```

Aplicação em http://localhost (Nginx serve a SPA e faz proxy de `/api` e `/uploads`).

## Cache

As leituras públicas passam pelo Redis em read-through: miss consulta o Postgres, grava e devolve.
As chaves carregam um número de versão por namespace (`eq:v3:list:<hash>`); invalidar é `INCR` no
contador do namespace, sem varrer chaves. O Postgres é sempre a fonte da verdade, e uma falha do
Redis degrada para consulta direta ao banco sem derrubar rota.

## Estrutura

- `backend/` — API NestJS, Prisma e seed
- `frontend/` — SPA React + Vite
- `legacy/` — protótipo original (`LocaTudo.dc.html`), mantido como referência visual
- `docs/superpowers/` — specs e planos
```

- [ ] **Step 5: Validar o build de produção de ponta a ponta**

```bash
docker compose down
docker compose -f docker-compose.prod.yml up --build -d
docker compose -f docker-compose.prod.yml exec backend npx prisma db seed
curl -s localhost/api/health
curl -s -o /dev/null -w "%{http_code}" localhost/
curl -s -o /dev/null -w "%{http_code}" localhost/uploads/seed/betoneira.svg
curl -s -o /dev/null -w "%{http_code}" localhost/resultados
```
Expected: health `{"status":"ok","db":"ok","redis":"ok"}`; `200` na raiz, no SVG e em `/resultados` (prova que o `try_files` da SPA funciona).

- [ ] **Step 6: Derrubar produção e voltar ao ambiente de desenvolvimento**

```bash
docker compose -f docker-compose.prod.yml down
docker compose up -d
```

- [ ] **Step 7: Commit**

```bash
git add backend/Dockerfile frontend/Dockerfile frontend/nginx.conf docker-compose.prod.yml README.md .env.example
git commit -m "feat: build de producao multi-stage com Nginx e documentacao"
```

---

## Verificação final da fase

Antes de considerar a fase 1 concluída, rode e confira a saída de cada comando:

```bash
docker compose down -v && docker compose up -d --build
docker compose exec backend npm test
docker compose exec backend npm run test:e2e
docker compose exec frontend npm test
curl -s localhost:3000/api/health
```

Critérios de aceitação:
- `docker compose up` sobe cinco serviços saudáveis a partir de um clone limpo, sem passo manual.
- Home lista as 8 categorias e destaques; `/resultados` filtra por texto, categoria, cidade e preço.
- Segunda leitura de qualquer rota pública é servida pelo Redis (`docker compose exec redis redis-cli keys '*'` mostra as chaves versionadas).
- Com o Redis parado, as rotas de leitura continuam respondendo.
- Todos os testes passam.

## Fora do escopo desta fase

Disponibilidade, cálculo de preço, reservas, autenticação, área administrativa e upload de imagens
pertencem às fases 2 e 3, que ganham seus próprios planos após esta fase estar em execução. O filtro
`apenasDisponiveis` é aceito e ignorado até a fase 2, conforme a spec.
