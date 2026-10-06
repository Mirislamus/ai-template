# Технологический стек проекта

Документ фиксирует технологический стек, правила выбора инструментов и допустимые библиотеки для веб-разработки (лендинги, многостраничные сайты, веб-сервисы). Полный список одобренных пакетов — [ADR 0010](decisions/common/0010-third-party-libraries-policy.md), команды установки — [docs/setup/](setup/common.md).

---

## 1. Базовые стандарты (общие для всех проектов)

- **Язык:** TypeScript (строгий режим `strict: true`).
- **Пакетный менеджер:** Bun (установка пакетов, запуск скриптов).
- **Стилизация:** SCSS Modules + глобальные токены и миксины в `src/shared/styles/` ([ADR 0003](decisions/frontend/0003-scss-standards.md)). Tailwind CSS запрещен.
- **Классы по условию:**
  - В Astro-разметке: нативная директива `class:list`.
  - В React-компонентах: легковесный `clsx`.
- **Иконки:** Lucide React (`lucide-react`) и оптимизированные (SVGO) локальные SVG-файлы; источник и лицензия каждого внешнего SVG фиксируются в описании изменения.
- **Модальные окна и Lightbox (галерея):**
  - Нативный HTML-элемент `<dialog>` с доступным поведением (ESC, блокировка скролла, фокус-трап).
  - Полноэкранный просмотр изображений (Lightbox): собственная модалка на базе `<dialog>` + Embla Carousel со стилями на SCSS Modules. При острой необходимости жестов пинч-зума допустимо точечное подключение PhotoSwipe v5.
- **Всплывающие уведомления (Toast):** Sonner (`sonner`).
- **Сетевой клиент:** Ky (`ky`) — легковесная обертка над Fetch с retry, таймаутами и хуками. Axios запрещен.
- **Управление серверным состоянием:** TanStack Query (`@tanstack/react-query`) для кеширования и мутаций данных.
- **Слайдеры и карусели:** Embla Carousel (`embla-carousel-react`).
- **Маски ввода (телефон, суммы):** IMask (`imask` / `react-imask`).
- **Аккордеоны (FAQ) и селекты:** строго нативные HTML-элементы:
  - FAQ и спойлеры: `<details>` и `<summary>`.
  - Выпадающие списки: нативный `<select>`.
- **Работа с датами:** нативные `Date` и `Intl.DateTimeFormat`.
- **Карты (локации, контакты):** конструктор карт через `<iframe>` с отложенной загрузкой (по умолчанию Яндекс.Карты, с возможностью замены на Google Maps или 2GIS).
- **Анимации и появление при скролле:** нативный `IntersectionObserver` + CSS Transitions/Keyframes; обязательный учет `prefers-reduced-motion` ([ADR 0017](decisions/frontend/0017-motion-and-animations.md)).
- **Инструменты качества и форматирования:**
  - **EditorConfig:** `.editorconfig` (2 пробела, LF, utf-8).
  - **Форматирование:** Prettier (`.prettierrc` в корне репозитория).
  - **Линтинг кода:** ESLint Flat Config (`typescript-eslint`, `eslint-plugin-astro`, `eslint-plugin-react-hooks`, `eslint-plugin-perfectionist`).
  - **Линтинг стилей:** Stylelint (`stylelint-config-standard-scss`, `stylelint-order`).
  - **Проверка типов:** `astro check` (Astro) или `tsc --noEmit` (Next.js).
  - **Тесты:** Vitest (unit), Playwright (E2E smoke), `html-validate` (валидность собранной разметки).

---

## 2. Профиль Astro (контентные сайты, лендинги, многостраничники)

Применяется, когда главный приоритет — скорость загрузки, Core Web Vitals (пороги — [quality.md](quality.md)) и минимальный клиентский JavaScript.

- **Основной фреймворк:** Astro (статическая генерация SSG).
- **Архитектура интерактива:** Astro Islands.
  - Статическая разметка, каркас, Hero, текстовые блоки — строго `.astro` (0 Кб JS в браузер).
  - Интерактивные модули (фильтры, корзина, сложные формы, слайдеры) — React через `@astrojs/react`.
- **Директивы гидратации:** `client:visible` (по умолчанию для блоков ниже первого экрана), `client:idle` (фоновые виджеты), `client:load` (только критический интерактив на первом экране).
- **Управление контентом:** Astro Content Collections (`src/content/`, схемы в `src/content.config.ts`), Zod из `astro/zod`.
- **Серверные эндпоинты (формы):** `src/pages/api/*.ts` с `prerender = false` и адаптер под хостинг (`@astrojs/node`, `@astrojs/cloudflare`, `@astrojs/vercel`, `@astrojs/netlify`), [ADR 0001](decisions/frontend/0001-project-architecture.md) §7.
- **Формы и валидация:** React Hook Form + Zod (React Island).
- **Глобальное состояние между островами:** Nano Stores (`nanostores` + `@nanostores/react`).
- **SEO и sitemap:** `@astrojs/sitemap`, OpenGraph и JSON-LD ([ADR 0013](decisions/frontend/0013-seo-standards.md)).

---

## 3. Профиль Next.js (веб-сервисы, сложная динамика, личные кабинеты)

Применяется, когда требуются развитый серверный рендеринг (SSR), авторизация, API Route Handlers или сложные панели управления.

- **Основной фреймворк:** Next.js (App Router).
- **Рендеринг:** Server Components по умолчанию, Client Components (`'use client'`) только на листьях дерева интерактивности.
- **Формы и валидация:** React Hook Form + Zod (клиент) + валидация Zod в Server Actions и Route Handlers.
- **Глобальное состояние:** Zustand (только клиентское состояние; собственный React Context запрещен).
- **Оптимизация ассетов:** `next/image` и `next/font`.

---

## 4. Бэкенд (опционально: своя БД, авторизация, админка, фоновые задачи)

Подключается только по решению в [PRODUCT.md](../PRODUCT.md) ([ADR 0019](decisions/backend/0019-backend-architecture.md) §1); проект становится монорепо на Bun workspaces.

- **Рантайм и фреймворк:** Bun + Elysia, TypeScript strict.
- **Клиент API во фронтенде:** Eden Treaty (`@elysia/eden`) внутри TanStack Query; `ky` — только для внешних API ([ADR 0020](decisions/backend/0020-api-contract-and-errors.md)).
- **Валидация:** Zod (общие схемы — в `packages/shared`).
- **База данных:** PostgreSQL 18 + Drizzle ORM (драйвер Bun SQL), миграции drizzle-kit ([ADR 0021](decisions/backend/0021-database-and-migrations.md)).
- **Авторизация:** Better Auth: сессии в httpOnly-cookie, роли, 2FA ([ADR 0022](decisions/backend/0022-auth-and-access.md)).
- **Логирование:** pino; алерты в Telegram ([ADR 0023](decisions/backend/0023-security-logging-and-config.md)).
- **Тесты:** `bun test` с реальной PostgreSQL ([ADR 0014](decisions/common/0014-testing-and-qa.md) §6).
- **Деплой:** Docker Compose на собственном сервере, образ в GHCR ([ADR 0018](decisions/common/0018-deployment-and-caching.md) §5).

---

## 5. Запрещенные к бесконтрольному использованию зависимости

Без прямого согласования **запрещено** подключать:
- **UI-фреймворки:** Tailwind CSS, Ant Design, MUI, Chakra UI.
- **Headless UI-библиотеки:** Radix UI, Headless UI (используются нативные теги `<dialog>`, `<details>`, `<select>`).
- **Тяжелые слайдеры:** Swiper.
- **Тяжелые анимации:** Framer Motion, Motion, GSAP, AOS.
- **Библиотеки дат и утилит:** Moment.js, Day.js, date-fns, Lodash.
- **HTTP-клиенты:** Axios (используется `ky`).
- **Сложные стейт-менеджеры:** Redux, MobX.
