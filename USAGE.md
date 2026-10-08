# Руководство по использованию шаблона (USAGE.md)

Инструкция для разработчиков и AI-агентов по старту, ведению и сдаче нового проекта на базе этого репозитория.

---

## 1. Концепция шаблона

`ai-template` — архитектурно-документационный стартовый комплект (meta-template). В нем нет `node_modules` и устаревших версий библиотек. Он предоставляет:
1. **Инженерные стандарты:** реестр архитектурных решений в `docs/decisions/` (перечень — [ARCHITECTURE.md](ARCHITECTURE.md)).
2. **Оперативные руководства:** дизайн-система, SEO, качество, типографика, регламент навыков.
3. **Руководства развертывания** ([docs/setup/](docs/setup/common.md): общие настройки, фронтенд, бэкенд) с эталонными конфигами под актуальные версии пакетов.

> **Быстрый старт:** создайте репозиторий кнопкой **Use this template** на GitHub (или скопируйте файлы шаблона в чистую папку), откройте чат с AI-агентом и отправьте готовый промпт из начала [PROMPT.md](PROMPT.md). Агент проведет допрос по продукту и заполнит паспорт проекта.

---

## 2. Пошаговый сценарий работы над новым проектом

```text
[Шаг 1: Паспорт продукта] ➔ [Шаг 2: Выбор стека] ➔ [Шаг 3: Инициализация] ➔ [Шаг 4: Разработка] ➔ [Шаг 5: Приемка]
       PRODUCT.md                 ADR 0001              docs/setup/               docs/pages/*.md        docs/quality.md
```

### Шаг 1: Паспорт продукта (`PRODUCT.md`)
Перед кодом заполняется [PRODUCT.md](PRODUCT.md) по итогам допроса ([PROMPT.md](PROMPT.md)): цели и метрика, аудитория, границы MVP, контент, страницы, формы, интеграции, аналитика и юридический контур, хостинг и домен, источник дизайна, сроки.

### Шаг 2: Выбор технологического профиля
По [ADR 0001](docs/decisions/frontend/0001-project-architecture.md):
- **Astro (SSG / Islands) — по умолчанию:** лендинги, корпоративные сайты, каталоги, блоги.
- **Next.js (App Router) — только при необходимости:** личные кабинеты с авторизацией, динамические панели, тяжелые серверные мутации.

Отдельный бэкенд (Bun, Elysia, Drizzle, PostgreSQL) подключается, только если он выбран в PRODUCT.md ([ADR 0019](docs/decisions/backend/0019-backend-architecture.md) §1).

### Шаг 3: Инициализация кодовой базы
1. [docs/setup/common.md](docs/setup/common.md): структура репозитория (один пакет или монорепо), `package.json`, Prettier, переменные окружения.
2. [docs/setup/frontend.md](docs/setup/frontend.md) для выбранного профиля: создание проекта и установка одобренного стека (раздел 1); эталонные конфиги (раздел 2): `tsconfig.json`, ESLint, Stylelint, `scripts`, тесты; конфиг фреймворка (раздел 3): `astro.config.ts`, `src/content.config.ts`; базовые стили в `src/shared/styles/` (раздел 4).
3. [docs/setup/backend.md](docs/setup/backend.md), если выбран бэкенд: `apps/api`, база данных, локальное окружение, Docker Compose.
4. Создание локального `.env` из `.env.example`.
5. Проверка: `bun run check`.

### Шаг 4: Верстка страниц
Паспорта страниц создаются и утверждаются сразу после `PRODUCT.md`, до инициализации ([PROMPT.md](PROMPT.md)): для каждой страницы копия [docs/pages/page-template.md](docs/pages/page-template.md) в `docs/pages/[name].md` с метаданными ([docs/seo.md](docs/seo.md)), секциями, типами компонентов и источниками данных. Верстка идет строго по утвержденным паспортам:
1. Новую страницу без паспорта не верстать: сначала паспорт.
2. Разрабатывайте компоненты по FSD-Lite ([ARCHITECTURE.md](ARCHITECTURE.md)).
3. Соблюдайте стандарты: разметка и a11y — [ADR 0002](docs/decisions/frontend/0002-html-standards.md); стили — [docs/design.md](docs/design.md) и [ADR 0003](docs/decisions/frontend/0003-scss-standards.md); типографика и тексты — [docs/content.md](docs/content.md); состояние — [ADR 0006](docs/decisions/frontend/0006-state-management.md).

### Шаг 5: Проверка качества и приемка
Перед сдачей задачи или релизом выполняется Definition of Done из [ADR 0014](docs/decisions/common/0014-testing-and-qa.md) §4:
1. `bun run check`, `bun run test`, `bun run test:e2e`, `bun run build`, `bun run validate:html`.
2. Чеклист геометрии и пороги Lighthouse — [docs/quality.md](docs/quality.md).
3. Проверка 5 состояний форм и мобильной верстки (360–390 px, без горизонтального скролла).
4. Пост-деплой проверка живого сайта в браузере (Google Rich Results Test для поддерживаемых типов, Schema.org Validator, W3C Nu HTML Checker) — [ADR 0013](docs/decisions/frontend/0013-seo-standards.md) §4.

---

## 3. Работа агента: навыки, задачи и отчет

- Перед стартом задачи агент объявляет применяемые навыки из [docs/skills.md](docs/skills.md) (или пишет «Скилы не требуются»); в итоговом отчете указывает «Использованные скилы».
- Агент оценивает уровень задачи (Low / Medium / High / XHigh); от уровня зависят порядок работы, карточка в `docs/tasks/` и исполнитель. Крупные задачи и целые этапы разбивает `planner`, подзадачи выполняют субагенты из `.claude/agents/` — [docs/agents.md](docs/agents.md).
- Регламент поведения агента — [AGENTS.md](AGENTS.md).

---

## 4. Фиксация прогресса и Git

- **Статус:** после каждого этапа обновляются [STATE.md](STATE.md) и [docs/tasks.md](docs/tasks.md).
- **Коммиты ([ADR 0011](docs/decisions/common/0011-git-workflow-and-commits.md)):** Conventional Commits на английском (`feat:`, `fix:`, `docs:`, `style:`, `refactor:`, `perf:`, `test:`, `chore:`). Пример: `feat(catalog): add category filter with url sync`.
- **Безопасность:** `git push --force` в `main` и деструктивные сбросы без бэкапа запрещены.
