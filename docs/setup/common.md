# Общая настройка проекта (docs/setup/common.md)

Пошаговая инструкция для разработчиков и AI-агентов: структура репозитория, общие конфиги и переменные окружения. Порядок развертывания: этот документ → [frontend.md](frontend.md) → [backend.md](backend.md) (только если бэкенд выбран в [PRODUCT.md](../../PRODUCT.md), [ADR 0019](../decisions/backend/0019-backend-architecture.md) §1). Документы `docs/setup/` — единственный источник эталонных конфигов; ADR описывают правила и ссылаются сюда.

Версии пакетов не фиксируются в документации: при старте проекта ставятся актуальные стабильные версии ([ADR 0010](../decisions/common/0010-third-party-libraries-policy.md) §1).

---

## 1. Структура репозитория

- **Без бэкенда:** фронтенд — единственный пакет в корне репозитория, инициализация по [frontend.md](frontend.md).
- **С бэкендом:** монорепо на Bun workspaces ([ADR 0019](../decisions/backend/0019-backend-architecture.md) §2):
  ```text
  package.json          # Корень: workspaces и общие скрипты
  bun.lock              # Единый lockfile монорепо
  apps/
  ├── web/              # Фронтенд, инициализация по frontend.md
  └── api/              # Бэкенд, инициализация по backend.md
  packages/
  └── shared/           # Общие Zod-схемы, типы и константы
  ```
  Файлы шаблона (`.gitignore`, `.gitattributes`, `.editorconfig`, `.prettierrc`) остаются в корне и действуют на весь репозиторий.

`packages/shared/package.json`:
```json
{
  "name": "shared",
  "private": true,
  "type": "module",
  "exports": { ".": "./src/index.ts" },
  "scripts": {
    "lint": "eslint . --max-warnings 0",
    "format": "bun fmt \"src/**/*.ts\" \"*.{ts,mjs,json}\"",
    "format:check": "bun fmt --check \"src/**/*.ts\" \"*.{ts,mjs,json}\"",
    "typecheck": "tsc --noEmit",
    "check": "bun run format:check && bun run lint && bun run typecheck"
  }
}
```
Пакет проверяется теми же правилами, что и `apps/api`, и входит в корневой `bun run check`:
```bash
cd packages/shared
bun add zod
# TypeScript 6: typescript-eslint не поддерживает TypeScript 7 (ADR 0010 §1)
bun add -d typescript@^6 eslint @eslint/js typescript-eslint eslint-plugin-perfectionist eslint-config-prettier
```
- `packages/shared/tsconfig.json` — копия `apps/api/tsconfig.json` ([backend.md](backend.md) §2) без `"types": ["bun"]` и с `"include": ["src"]`.
- `packages/shared/eslint.config.mjs` — копия `apps/api/eslint.config.mjs` ([backend.md](backend.md) §2) без `globalIgnores`, без блока-исключения для чтения окружения и с `internalPattern: []`: пакет не обращается к окружению ([ADR 0019](../decisions/backend/0019-backend-architecture.md) §2).

Зависимости приложений от пакетов монорепо: в `apps/web` — `"shared": "workspace:*"` и `"api": "workspace:*"` (только для импорта типа `App`, [ADR 0020](../decisions/backend/0020-api-contract-and-errors.md) §1), в `apps/api` — `"shared": "workspace:*"`.

---

## 2. `package.json`: пакетный менеджер и окружение (ADR 0012)

В `package.json` корня репозитория:
```jsonc
{
  "packageManager": "bun@<версия из bun --version>",
  "engines": {
    "node": ">=24.0.0",
    "bun": ">=<версия из bun --version>"
  }
}
```
Значения записываются фактическими версиями на момент инициализации.

В монорепо корневой `package.json` дополнительно объявляет workspaces и общие скрипты:
```json
{
  "name": "project",
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "dev:web": "bun run --filter web dev",
    "dev:api": "bun run --filter api dev",
    "check": "bun run --filter '*' check",
    "test": "bun run --filter '*' test",
    "build": "bun run --filter '*' build"
  }
}
```
- `bun run --filter` соблюдает порядок зависимостей между пакетами монорепо ([документация Bun](https://bun.com/docs/pm/filter)). Поэтому dev-серверы запускаются отдельными командами `dev:api` и `dev:web` в двух терминалах: общий `dev` ждал бы завершения dev-сервера API.
- Скрипты `check`, `test`, `build` объявляются в каждом приложении ([frontend.md](frontend.md) §2.4, [backend.md](backend.md) §3), `check` — также в `packages/shared` (§1). Пакет без нужного скрипта `bun run --filter` пропускает.

---

## 3. `.prettierrc`

Единственный источник — корневой [.prettierrc](../../.prettierrc), его читает форматтер `bun fmt`. Правила форматирования описаны в [ADR 0009](../decisions/common/0009-tooling-and-linting.md) §1. В монорепо форматирование всех пакетов идет по этому файлу.

---

## 4. Переменные окружения

- Шаблон — [.env.example](../../.env.example). Локальный `.env` копируется из него и не коммитится.
- **Без бэкенда** `.env` лежит в корне. **В монорепо** у каждого приложения свой файл: раздел фронтенда `.env.example` переносится в `apps/web/.env.example`, раздел бэкенда — в `apps/api/.env.example`; рабочие `.env` создаются рядом с ними.
- Фронтенд читает переменные только через `astro:env` или `src/shared/config/env.ts` ([ADR 0004](../decisions/common/0004-ts-js-standards.md) §13).
- Бэкенд читает переменные только через `src/config/env.ts` ([ADR 0023](../decisions/backend/0023-security-logging-and-config.md) §4, [backend.md](backend.md) §4).

---

## 5. Dependabot

`.github/dependabot.yml` ([ADR 0010](../decisions/common/0010-third-party-libraries-policy.md) §1):
```yaml
version: 2
updates:
  - package-ecosystem: 'bun'
    directory: '/'
    schedule:
      interval: 'weekly'
```
В проекте с бэкендом добавляется экосистема `docker` ([backend.md](backend.md) §9).
