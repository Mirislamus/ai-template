# 012 — Дополнения шаблона: CI, навыки, права, профиль Next.js

- **Уровень:** Medium
- **Статус:** готово
- **Источники:** аудит шаблона по запросу пользователя; ADR 0018 §4, ADR 0014 §4 и §6, ADR 0004 §13, ADR 0012 §3; [docs/skills.md](../skills.md); документация [Claude Code](https://code.claude.com/docs/en/permissions), [setup-bun](https://github.com/oven-sh/setup-bun), README источников навыков
- **Цель:** шаблон содержит эталонный workflow CI, источники навыков, права Claude Code, версию и историю изменений, шаблон PR, а профиль Next.js имеет эталонные конфиги наравне с Astro.
- **Обоснование оценки:** документация, эталонные конфиги и настройки, больше 5 файлов, есть решения пользователя ([docs/workflow.md](../workflow.md) §1).

## Решения пользователя

- Профиль Next.js доводится до уровня Astro: эталонные конфиги проверяются в песочнице.
- CHANGELOG.md с версией шаблона по SemVer; карточки задач шаблона остаются.
- Файл LICENSE не создается.
- Права в `.claude/settings.json`: запрет чтения файлов окружения, подтверждение Git-команд записи, разрешение проверок и Git-команд чтения.

## Критерии готовности

- [x] docs/setup содержит эталонный `.github/workflows/ci.yml` по шагам ADR 0018 §4, включая дополнения для бэкенда.
- [x] docs/skills.md §6 указывает источник и команду установки навыков.
- [x] `.claude/settings.json` содержит `permissions`.
- [x] CHANGELOG.md создан, шаг очистки переносит версию в STATE.md.
- [x] Есть `.github/pull_request_template.md`.
- [x] Профиль Next.js: `next.config.ts`, `env.ts`, `vitest.config.ts`, команда Playwright, `validate:html`, состав `eslint.config.mjs`.

## Подзадачи

| # | Подзадача | Уровень | Исполнитель | Зависит от | Статус |
|---|---|---|---|---|---|
| 1 | CI: docs/setup/common.md §6, docs/setup/backend.md §10 | Medium | основной агент (инфраструктура) | — | ☑ |
| 2 | Источники навыков: docs/skills.md §6 | Low | основной агент (документация) | — | ☑ |
| 3 | Права Claude Code, шаблон PR, CHANGELOG и ссылки на них | Medium | основной агент (документация) | — | ☑ |
| 4 | Профиль Next.js: docs/setup/frontend.md, docs/setup/backend.md, ADR 0004, ADR 0014, субагенты | Medium | основной агент (документация) | — | ☑ |

## Проверки

- Песочница вне проекта: `create-next-app` 16.4 по команде из docs/setup/frontend.md, эталонные конфиги Next.js, `eslint`, `tsc --noEmit`, `vitest run`, `next build`, `html-validate`, `next start --port 4321`.
- Разбор YAML эталонного workflow, проверка JSON настроек, проверка относительных ссылок во всех `.md`.

## Итог

- **CI:** эталонный workflow повторяет шаги ADR 0018 §4; версии — `actions/checkout@v7`, `oven-sh/setup-bun@v2`. В GitHub Actions workflow не запускался.
- **Навыки:** источники подтверждены по README для всех навыков, кроме `$request-refactor-plan` и `$resolving-merge-conflicts`: в mattpocock/skills их нет, в таблицу они не включены.
- **Права:** `deny` перечисляет файлы окружения поименно, поэтому `.env.example` остается доступным. Действие правил в сессии Claude Code не проверялось.
- **Next.js:** скрипт `validate:html` из профиля Astro в Next.js не работал — каталога `dist/` нет, а набор правил по умолчанию дает 78 ошибок на разметке React. Теперь проверяется `.next/server/app/` набором `html-validate:standard`. Составной `eslint.config.mjs`, `env.ts`, `vitest.config.ts` и `next.config.ts` проходят `eslint`, `tsc` и `bun fmt --check`; сборка и запуск на порту 4321 работают. Playwright и `build:analyze` в песочнице не запускались.
- **Не исправлялось:** конфиг Nginx и команды деплоя в ADR 0018 описаны только для Astro; ADR 0013 не описывает `metadata` Next.js.
