# AI-Native Web Template

Инженерный шаблон для создания быстрых веб-сайтов, лендингов и веб-сервисов с помощью AI-агентов и разработчиков. Приоритеты: производительность (пороги — [docs/quality.md](docs/quality.md)), строгая типобезопасность и модульная архитектура.

> **Как использовать шаблон:** создайте репозиторий кнопкой **Use this template** на GitHub (или скопируйте файлы в чистую папку) и отправьте AI-агенту промпт из [PROMPT.md](PROMPT.md). Пошаговая инструкция для разработчиков и AI-агентов — в [USAGE.md](USAGE.md).

---

## Технологический стек

- **Рантайм и пакетный менеджер:** [Bun](https://bun.sh) (`bun install`, `bun run`).
- **Язык:** TypeScript (`strict: true`, запрет `any` и `enum`).
- **Фреймворки (по профилям):**
  - **Astro (SSG / Islands):** по умолчанию для лендингов, каталогов и контентных сайтов.
  - **Next.js (App Router):** для веб-сервисов, личных кабинетов и сложной серверной динамики.
- **Интерактивные компоненты:** React 19 (`function` declaration, плоские импорты, `ref` без `forwardRef`).
- **Стилизация:** SCSS Modules + семантические токены в CSS-переменных (8pt Grid).
- **Сеть и состояние:** `ky`, `@tanstack/react-query`, `nanostores` (Astro), `zustand` (Next.js).
- **Формы и валидация:** `react-hook-form` + `zod` + `imask`.
- **Иконки и уведомления:** `lucide-react`, `sonner`.
- **Контроль качества:** Prettier, ESLint, Stylelint, Vitest, Playwright, `html-validate`.
- **Бэкенд (опционально):** Elysia, Drizzle ORM, PostgreSQL, Better Auth, Docker Compose; подключается по решению в PRODUCT.md ([ADR 0019](docs/decisions/backend/0019-backend-architecture.md)). Команды `apps/api` — [docs/setup/backend.md](docs/setup/backend.md) §3.

Полный перечень и правила выбора — [docs/tech.md](docs/tech.md).

---

## Быстрый старт

### Требования к окружению
- **Bun:** актуальная стабильная версия.
- **Node.js:** `>= 24.0.0` (LTS).

### Установка и запуск
Новый проект создается по руководствам [docs/setup/](docs/setup/common.md): общие настройки, фронтенд, бэкенд (если выбран). Дальше:
```bash
bun install
bun run dev
```
В монорепо с бэкендом dev-серверы запускаются из корня отдельными командами `bun run dev:web` и `bun run dev:api` ([docs/setup/common.md](docs/setup/common.md) §2).

Использование `npm`, `yarn` и `pnpm` запрещено: конфликт lock-файлов.

---

## Команды проекта

| Команда | Описание |
|---|---|
| `bun run dev` | Локальный сервер разработки |
| `bun run build` | Сборка в продакшен |
| `bun run preview` | Предпросмотр собранного проекта (Astro; в Next.js — `bun run start`) |
| `bun run check` | **Единый конвейер качества:** Prettier + ESLint + Stylelint + проверка типов |
| `bun run lint` | ESLint и Stylelint |
| `bun run format` | Форматирование Prettier |
| `bun run typecheck` | Проверка типов без генерации файлов |
| `bun run test` | Модульные тесты Vitest |
| `bun run test:e2e` | Сквозные smoke-тесты Playwright |
| `bun run validate:html` | Проверка валидности собранной разметки (`html-validate`) |
| `bun run build:analyze` | Сборка с отчетом о размерах бандла (`stats.html`) |

Эталонные `scripts`: фронтенд — [docs/setup/frontend.md](docs/setup/frontend.md) §2.4, бэкенд — [docs/setup/backend.md](docs/setup/backend.md) §3.

---

## Архитектура кодовой базы

Структура `src/` (FSD-Lite), зоны ответственности слоев и правило импортов — в [ARCHITECTURE.md](ARCHITECTURE.md).

---

## Документация и стандарты

- **Оперативные руководства (`docs/`):**
  - [docs/tech.md](docs/tech.md) — технологический стек и правила выбора библиотек.
  - [docs/setup/](docs/setup/common.md) — развертывание проекта и эталонные конфиги: [common.md](docs/setup/common.md), [frontend.md](docs/setup/frontend.md), [backend.md](docs/setup/backend.md).
  - [docs/skills.md](docs/skills.md) — регламент AI-навыков.
  - [docs/design.md](docs/design.md) — дизайн-система: сетка, брейкпоинты, токены, состояния, анти-шаблоны.
  - [docs/quality.md](docs/quality.md) — чеклист геометрии, пороги Lighthouse и Core Web Vitals, кроссбраузерность.
  - [docs/seo.md](docs/seo.md) — метаданные, иерархия заголовков, OpenGraph.
  - [docs/content.md](docs/content.md) — типографика, инфостиль, микрокопирайтинг, согласие на обработку данных.
  - [docs/pages/](docs/pages/) — паспорта страниц.
- **Реестр архитектурных решений:** [docs/decisions/](docs/decisions/), перечень — в [ARCHITECTURE.md](ARCHITECTURE.md).
