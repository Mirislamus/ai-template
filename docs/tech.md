Failed to write init script: open C:\Users\Windows 10\AppData\Local\Packages\ohmyposh.cli_96v55e8n804z4\LocalCache\Local\oh-my-posh\init.814522496948324317.ps1: Access is denied.
Export-Clixml: Access to the path 'C:\Users\Windows 10\AppData\Roaming\powershell\Community\Terminal-Icons\devblackops_color.xml' is
denied.
Export-Clixml: Access to the path 'C:\Users\Windows
10\AppData\Roaming\powershell\Community\Terminal-Icons\devblackops_light_color.xml' is denied.
Export-Clixml: Access to the path 'C:\Users\Windows 10\AppData\Roaming\powershell\Community\Terminal-Icons\devblackops_icon.xml' is
denied.
Export-Clixml: Access to the path 'C:\Users\Windows 10\AppData\Roaming\powershell\Community\Terminal-Icons\prefs.xml' is denied.
# Технологический стек проекта

Документ фиксирует технологический стек, правила выбора инструментов и допустимые библиотеки для веб-разработки (лендинги, многостраничные сайты, веб-сервисы).

---

## 1. Базовые стандарты (Общие для всех проектов)

- **Язык:** TypeScript (строгий режим `strict: true`).
- **Пакетный менеджер:** Bun (установка пакетов, запуск скриптов).
- **Стилизация:** SCSS Modules + глобальные токены/переменные в `src/styles/` (или `src/shared/styles/`).
- **Классы по условию:** 
  - В Astro-разметке: нативная директива `class:list`.
  - В React-компонентах: легковесный `clsx`.
- **Иконки:** Lucide React (`lucide-react`) и оптимизированные локальные SVG-векторы с TheSVG.org.
- **Модальные окна и Lightbox (галерея):** 
  - Нативный HTML-элемент `<dialog>` с доступным поведением (ESC, блокировка скролла, фокус-трап).
  - Полноэкранный просмотр изображений (Lightbox): собственная модалка на базе `<dialog>` + Embla Carousel со стилями на SCSS Modules (без сторонних тяжелых библиотек). При острой необходимости жестов пинч-зума допустимо точечное подключение PhotoSwipe v5.
- **Всплывающие уведомления (Toast):** Sonner (`sonner`).
- **Сетевой клиент:** Ky (`ky`) — легковесная обертка над Fetch с retry, таймаутами и хуками.
- **Управление серверным состоянием:** TanStack Query (`@tanstack/react-query`) для кэширования и мутаций данных.
- **Слайдеры и карусели:** Embla Carousel (`embla-carousel-react`).
- **Маски ввода (телефон, суммы):** IMask (`imask` / `react-imask`).
- **Аккордеоны (FAQ) и селекты:** Строго нативные HTML-элементы:
  - FAQ / спойлеры: нативные `<details>` и `<summary>`.
  - Выпадающие списки: нативный `<select>` (без Radix UI и тяжелых headless-компонентов).
- **Работа с датами:** Нативные объекты `Date` и `Intl.DateTimeFormat` (без Moment.js / Day.js / date-fns).
- **Карты (локации, контакты):** Конструктор карт через `<iframe>` с отложенной/ленивой загрузкой (по умолчанию Яндекс.Карты, с возможностью замены на Google Maps / 2GIS).
- **Анимации и появление при скролле:** Нативный `IntersectionObserver` + CSS Transitions/Keyframes. Обязательный учет `prefers-reduced-motion`. Запрет тяжелых библиотек (GSAP, Framer Motion, AOS) без специального согласования.
- **Инструменты качества и форматирования:**
  - **EditorConfig:** `.editorconfig` (отступы 2 пробела, LF, utf-8, trim trailing whitespace).
  - **Форматирование:** Prettier (`.prettierrc` с плагином `prettier-plugin-astro`, singleQuote, printWidth: 120, trailingComma: all, semi: true, arrowParens: avoid).
  - **Линтинг кода:** ESLint (`eslint-plugin-astro`, `eslint-plugin-react-hooks`).
  - **Линтинг стилей:** Stylelint (`stylelint-config-standard-scss`).
  - **Проверка типов:** `tsc --noEmit` (или `astro check`).

---

## 2. Профиль Astro (Контентные сайты, лендинги, многостраничники)

Применяется, когда главный приоритет — скорость загрузки, высокая оценка Core Web Vitals (Lighthouse 100) и минимальный клиентский JavaScript.

- **Основной фреймворк:** Astro (генерация статики SSG).
- **Архитектура интерактива:** Astro Islands (острова архитектуры).
  - Статическая разметка, каркас, Hero, текстовые блоки — строго `.astro` (0 Кб JS в браузер).
  - Интерактивные модули (фильтры, корзина, сложные формы, слайдеры) — React через `@astrojs/react`.
- **Директивы гидратации:** `client:visible` (по умолчанию для блоков ниже первого экрана), `client:idle` (для фоновых виджетов), `client:load` (только для критического интерактива на первом экране).
- **Управление контентом:** Astro Content Collections (`src/content/`).
- **Проверка схем контента:** Встроенный Zod (`astro:content` / `astro/zod`).
- **Формы и валидация:** 
  - На клиенте (React Island): React Hook Form + Zod.
- **Глобальное состояние между островами:** Nano Stores (`nanostores` + `@nanostores/react`).
- **SEO и sitemap:** `@astrojs/sitemap`, генерация OpenGraph и JSON-LD разметки.

---

## 3. Профиль Next.js (Веб-сервисы, сложная динамика, личные кабинеты)

Применяется, когда требуются развитый серверный рендеринг (SSR), авторизация пользователей, API Route Handlers или сложные панели управления.

- **Основной фреймворк:** Next.js (App Router).
- **Рендеринг:** Server Components по умолчанию (RSC), Client Components (`'use client'`) только на листьях дерева интерактивности.
- **Формы и валидация:** React Hook Form + Zod (на клиенте) + валидация Zod в Server Actions / Route Handlers.
- **Глобальное состояние:** Zustand (только при необходимости межкомпонентного взаимодействия; отказ от нативного React Context для глобальных хранилищ).
- **Оптимизация ассетов:** Встроенные компоненты `next/image` и `next/font`.

---

## 4. Запрещенные к бесконтрольному использованию зависимости

Без прямого технического согласования **запрещено** подключать:
- **UI-фреймворки с тяжелыми рантаймами:** Tailwind CSS (если выбран SCSS Modules), Ant Design, MUI, Chakra UI.
- **Headless UI-библиотеки:** Radix UI, Headless UI (использовать нативные HTML-теги `<dialog>`, `<details>`, `<select>`).
- **Тяжелые слайдеры:** Swiper.
- **Тяжелые анимации:** Framer Motion, Motion, GSAP, AOS.
- **Тяжелые библиотеки дат и утилит:** Moment.js, Day.js, date-fns, Lodash.
- **Устаревшие HTTP-клиенты:** Axios (использовать нативный Fetch API).
- **Сложные стейт-менеджеры:** Redux, MobX.

