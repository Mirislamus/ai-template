# Развертывание бэкенда (docs/setup/backend.md)

Инициализация `apps/api` (Bun, Elysia, Drizzle, PostgreSQL), эталонные конфиги, локальное окружение и продакшен в Docker Compose. Выполняется после [common.md](common.md) и [frontend.md](frontend.md), только если бэкенд выбран в [PRODUCT.md](../../PRODUCT.md). Правила — [docs/decisions/backend/](../decisions/backend/).

---

## 1. Инициализация `apps/api`

```bash
# 1. Пакеты бэкенда (ADR 0010 §3)
cd apps/api
bun add elysia @elysia/openapi zod drizzle-orm better-auth @better-auth/drizzle-adapter pino file-type
bun add -d drizzle-kit @types/bun typescript eslint @eslint/js typescript-eslint eslint-plugin-perfectionist eslint-config-prettier prettier

# 2. Клиент API во фронтенде (ADR 0020 §1)
cd ../web
bun add @elysia/eden
```

`apps/api/package.json`:
```json
{
  "name": "api",
  "private": true,
  "type": "module",
  "exports": { ".": "./src/app.ts" },
  "imports": { "#src/*": "./src/*.ts" },
  "dependencies": { "shared": "workspace:*" }
}
```
- `exports` отдает фронтенду только `src/app.ts`, из которого импортируется тип `App` ([ADR 0020](../decisions/backend/0020-api-contract-and-errors.md) §1).
- Внутренние импорты API идут через `#src/*` (`import { db } from '#src/db/client'`), а не через `paths` в `tsconfig.json`: при проверке типов `apps/web` файлы API разбираются с настройками фронтенда, и алиас `@/*` указал бы на `apps/web/src` ([ADR 0019](../decisions/backend/0019-backend-architecture.md) §3).

Таблицы Better Auth генерируются после создания `src/shared/auth.ts` командой `bunx auth@latest generate`; результат сохраняется в `src/db/schema/auth.ts` ([ADR 0022](../decisions/backend/0022-auth-and-access.md) §1).

---

## 2. `tsconfig.json` и `eslint.config.mjs`

`apps/api/tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "Preserve",
    "moduleResolution": "bundler",
    "lib": ["ESNext"],
    "types": ["bun"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true,
    "noEmit": true
  },
  "include": ["src", "drizzle.config.ts"]
}
```

`apps/api/eslint.config.mjs` — правила TypeScript совпадают с фронтендом ([frontend.md](frontend.md) §2.2), без плагинов Astro и React:
```javascript
import eslint from '@eslint/js';
import { defineConfig, globalIgnores } from 'eslint/config';
import prettier from 'eslint-config-prettier';
import perfectionist from 'eslint-plugin-perfectionist';
import tseslint from 'typescript-eslint';

export default defineConfig([
  globalIgnores(['drizzle', 'uploads']),
  eslint.configs.recommended,
  tseslint.configs.recommended,
  {
    files: ['**/*.ts'],
    extends: [tseslint.configs.recommendedTypeChecked],
    languageOptions: {
      parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname },
    },
    rules: {
      '@typescript-eslint/no-floating-promises': 'error',
    },
  },
  {
    plugins: { perfectionist },
    rules: {
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/no-non-null-assertion': 'error',
      '@typescript-eslint/consistent-type-imports': ['error', { prefer: 'type-imports', fixStyle: 'separate-type-imports' }],
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_', varsIgnorePattern: '^_' }],
      'no-console': ['warn', { allow: ['warn', 'error'] }],
      'no-restricted-syntax': [
        'error',
        { selector: 'TSEnumDeclaration', message: 'enum запрещен (ADR 0004 §4): используйте union-типы или объект as const.' },
      ],
      'no-restricted-properties': [
        'error',
        { object: 'process', property: 'env', message: 'Переменные окружения читаются через src/config/env.ts (ADR 0023 §4).' },
      ],
      'perfectionist/sort-imports': [
        'error',
        {
          type: 'natural',
          newlinesBetween: 'always',
          internalPattern: ['^#src/.*'],
          groups: [
            ['type-builtin', 'value-builtin', 'type-external', 'value-external'],
            ['type-internal', 'value-internal'],
            ['type-parent', 'type-sibling', 'type-index', 'value-parent', 'value-sibling', 'value-index'],
            'unknown',
          ],
        },
      ],
    },
  },
  {
    // Единственные места чтения окружения (ADR 0023 §4)
    files: ['src/config/env.ts', 'drizzle.config.ts'],
    rules: { 'no-restricted-properties': 'off' },
  },
  prettier,
]);
```

---

## 3. Секция `scripts` в `apps/api/package.json`

```json
{
  "scripts": {
    "dev": "bun --watch src/index.ts",
    "start": "bun src/index.ts",
    "worker": "bun src/worker.ts",
    "lint": "eslint . --max-warnings 0",
    "format": "prettier --write \"src/**/*.ts\"",
    "format:check": "prettier --check \"src/**/*.ts\"",
    "typecheck": "tsc --noEmit",
    "check": "bun run format:check && bun run lint && bun run typecheck",
    "test": "bun test",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "bun src/db/migrate.ts",
    "db:push": "drizzle-kit push",
    "db:seed": "bun src/db/seed.ts"
  }
}
```
`db:push` и `db:seed` — только для локальной базы ([ADR 0021](../decisions/backend/0021-database-and-migrations.md) §5).

---

## 4. `src/config/env.ts`

```ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().int().default(3000),
  DATABASE_URL: z.url(),
  BETTER_AUTH_SECRET: z.string().min(32),
  BETTER_AUTH_URL: z.url(),
  UPLOADS_DIR: z.string().min(1),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
  ALERT_TELEGRAM_BOT_TOKEN: z.string().min(1).optional(),
  ALERT_TELEGRAM_CHAT_ID: z.string().min(1).optional(),
});

const parsed = envSchema.safeParse(process.env);

if (!parsed.success) {
  // Процесс не стартует без валидной конфигурации (ADR 0023 §4)
  console.error(z.prettifyError(parsed.error));
  process.exit(1);
}

export const env = parsed.data;
```
Bun сам загружает `.env` из каталога `apps/api`, при `bun test` — `.env.test`. Имена переменных — [.env.example](../../.env.example), раздел бэкенда.

---

## 5. База данных

`src/db/client.ts`:
```ts
import { SQL } from 'bun';
import { drizzle } from 'drizzle-orm/bun-sql';

import { env } from '#src/config/env';
import * as schema from '#src/db/schema/index';

export const client = new SQL(env.DATABASE_URL);
export const db = drizzle({ client, schema, casing: 'snake_case' });
```

`drizzle.config.ts`:
```ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  dialect: 'postgresql',
  schema: './src/db/schema',
  out: './drizzle',
  casing: 'snake_case',
  dbCredentials: { url: process.env.DATABASE_URL ?? '' },
  strict: true,
  verbose: true,
});
```

`src/db/migrate.ts` — применение миграций на сервере без `drizzle-kit` ([ADR 0021](../decisions/backend/0021-database-and-migrations.md) §5):
```ts
import { migrate } from 'drizzle-orm/bun-sql/migrator';

import { client, db } from '#src/db/client';

await migrate(db, { migrationsFolder: './drizzle' });
await client.close();
```

---

## 6. Локальная разработка и тесты

`apps/api/compose.dev.yml` — локальная и тестовая базы (пароли только для локальной машины):
```yaml
services:
  db:
    image: postgres:18-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    ports:
      - '127.0.0.1:5432:5432'
    volumes:
      - db-dev:/var/lib/postgresql

  db-test:
    image: postgres:18-alpine
    environment:
      POSTGRES_DB: app_test
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    ports:
      - '127.0.0.1:5433:5432'
    tmpfs:
      - /var/lib/postgresql

volumes:
  db-dev:
```
Запуск: `docker compose -f compose.dev.yml up -d`, затем `bun run db:migrate` и `bun run dev`.

Тестовая база подключается через `apps/api/.env.test` (`DATABASE_URL=postgres://app:app@localhost:5433/app_test`). Миграции применяются перед прогоном тестов файлом предзагрузки:
```toml
# apps/api/bunfig.toml
[test]
preload = ["./src/test/setup.ts"]
```
```ts
// apps/api/src/test/setup.ts
import { migrate } from 'drizzle-orm/bun-sql/migrator';

import { db } from '#src/db/client';

await migrate(db, { migrationsFolder: './drizzle' });
```

**Проксирование `/api` в dev-сервере фронтенда** ([ADR 0020](../decisions/backend/0020-api-contract-and-errors.md) §1):
- Astro — в `astro.config.ts` ([frontend.md](frontend.md) §3.1): `vite: { server: { proxy: { '/api': 'http://localhost:3000' } } }`.
- Next.js — в `next.config.ts`: `rewrites` с `source: '/api/:path*'` на `http://localhost:3000/api/:path*`.

Серверный код фронтенда обращается к API по `API_INTERNAL_URL` (локально `http://localhost:3000`).

---

## 7. Продакшен: образ и Docker Compose

`.dockerignore` в корне репозитория (контекст сборки — весь монорепо):
```text
.git
**/node_modules
**/.env
**/.env.*
**/uploads
apps/web/*
!apps/web/package.json
```

`apps/api/Dockerfile`:
```dockerfile
FROM oven/bun:1
WORKDIR /app
ENV NODE_ENV=production

# Манифесты всех пакетов нужны для установки по общему bun.lock
COPY package.json bun.lock ./
COPY apps/api/package.json apps/api/
COPY apps/web/package.json apps/web/
COPY packages/shared/package.json packages/shared/
RUN bun install --frozen-lockfile --production

COPY packages/shared packages/shared
COPY apps/api apps/api

WORKDIR /app/apps/api
USER bun
EXPOSE 3000
CMD ["bun", "src/index.ts"]
```
Сборка в CI: `docker build -f apps/api/Dockerfile -t ghcr.io/<owner>/<repo>-api:<sha> .` ([ADR 0018](../decisions/common/0018-deployment-and-caching.md) §4).

`/srv/project/docker-compose.yml` на сервере:
```yaml
x-logging: &logging
  driver: journald

x-api: &api
  image: ghcr.io/<owner>/<repo>-api:${API_TAG}
  env_file: .env
  logging: *logging
  depends_on:
    db:
      condition: service_healthy

services:
  db:
    image: postgres:18-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      APP_DB_USER: ${APP_DB_USER}
      APP_DB_PASSWORD: ${APP_DB_PASSWORD}
    volumes:
      - db-data:/var/lib/postgresql
      - ./initdb:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U "$${POSTGRES_USER}" -d "$${POSTGRES_DB}"']
      interval: 5s
      timeout: 3s
      retries: 10
    logging: *logging

  api:
    <<: *api
    restart: unless-stopped
    ports:
      - '127.0.0.1:3000:3000'
    volumes:
      - ./uploads:/app/apps/api/uploads
    healthcheck:
      test: ['CMD', 'bun', '-e', "fetch('http://localhost:3000/api/health').then(r => process.exit(r.ok ? 0 : 1)).catch(() => process.exit(1))"]
      interval: 30s
      timeout: 5s
      retries: 3

  worker:
    <<: *api
    restart: unless-stopped
    command: ['bun', 'src/worker.ts']

  migrate:
    <<: *api
    command: ['bun', 'src/db/migrate.ts']
    profiles: ['migrate']

volumes:
  db-data:
```
- `/srv/project/.env` (права `600`) содержит переменные бэкенда, `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` (суперпользователь для обслуживания) и `APP_DB_USER`, `APP_DB_PASSWORD` (роль приложения). `DATABASE_URL` использует роль приложения и сервис `db`: `postgres://<APP_DB_USER>:<APP_DB_PASSWORD>@db:5432/<POSTGRES_DB>`. `UPLOADS_DIR=/app/apps/api/uploads`. Пароли генерируются командой `openssl rand -hex 32`.
- Роль приложения создается скриптом `/srv/project/initdb/01-app-user.sh` ([ADR 0021](../decisions/backend/0021-database-and-migrations.md) §1). Образ PostgreSQL выполняет его один раз, при инициализации пустого тома:
  ```bash
  #!/bin/sh
  set -e
  psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
    CREATE ROLE "$APP_DB_USER" LOGIN PASSWORD '$APP_DB_PASSWORD';
    ALTER DATABASE "$POSTGRES_DB" OWNER TO "$APP_DB_USER";
  EOSQL
  ```
  Владелец базы получает право создавать таблицы в схеме `public` (в PostgreSQL 15 и новее схемой владеет `pg_database_owner`), поэтому миграции выполняются ролью приложения.
- Каталог `/srv/project/uploads` с подкаталогами `public/` и `private/` создается до первого запуска; его владелец — пользователь `bun` из образа (UID: `docker compose run --rm api id -u`).
- Сервер авторизуется в GHCR один раз: `docker login ghcr.io` с токеном, у которого есть только право `read:packages`.
- Деплой: `export API_TAG=<sha>`, затем шаги [ADR 0018](../decisions/common/0018-deployment-and-caching.md) §5.

**Хранение логов 14 дней** ([ADR 0023](../decisions/backend/0023-security-logging-and-config.md) §5), `/etc/systemd/journald.conf`:
```ini
[Journal]
MaxRetentionSec=14day
SystemMaxUse=1G
```
Применение: `systemctl restart systemd-journald`. Просмотр: `docker compose logs -f api`.

---

## 8. Резервные копии

`/usr/local/bin/backup-project.sh` ([ADR 0021](../decisions/backend/0021-database-and-migrations.md) §6):
```bash
#!/usr/bin/env bash
set -euo pipefail

PROJECT_DIR=/srv/project
BACKUP_DIR=/var/backups/project
STAMP=$(date +%F)

mkdir -p "$BACKUP_DIR"
cd "$PROJECT_DIR"
docker compose exec -T db sh -c 'pg_dump -Fc -U "$POSTGRES_USER" "$POSTGRES_DB"' > "$BACKUP_DIR/db-$STAMP.dump"
tar -czf "$BACKUP_DIR/uploads-$STAMP.tar.gz" -C "$PROJECT_DIR" uploads
chmod 600 "$BACKUP_DIR"/*
find "$BACKUP_DIR" -type f -mtime +14 -delete
```
Расписание, `/etc/cron.d/backup-project`:
```text
30 3 * * * root /usr/local/bin/backup-project.sh
```
Ежемесячная проверка восстановления во временную базу:
```bash
docker compose exec -T db sh -c 'createdb -U "$POSTGRES_USER" restore_check'
docker compose exec -T db sh -c 'pg_restore -U "$POSTGRES_USER" -d restore_check' < /var/backups/project/db-<дата>.dump
docker compose exec -T db sh -c 'psql -U "$POSTGRES_USER" -d restore_check -c "select count(*) from \"user\""'
docker compose exec -T db sh -c 'dropdb -U "$POSTGRES_USER" restore_check'
```

---

## 9. Dependabot

`.github/dependabot.yml` ([ADR 0023](../decisions/backend/0023-security-logging-and-config.md) §8):
```yaml
version: 2
updates:
  - package-ecosystem: 'bun'
    directory: '/'
    schedule:
      interval: 'weekly'
  - package-ecosystem: 'docker'
    directory: '/apps/api'
    schedule:
      interval: 'weekly'
```
