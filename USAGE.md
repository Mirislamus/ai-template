Failed to write init script: open C:\Users\Windows 10\AppData\Local\Packages\ohmyposh.cli_96v55e8n804z4\LocalCache\Local\oh-my-posh\init.814522496948324317.ps1: Access is denied.
Export-Clixml: Access to the path 'C:\Users\Windows
10\AppData\Roaming\powershell\Community\Terminal-Icons\devblackops_light_color.xml' is denied.
Export-Clixml: Access to the path 'C:\Users\Windows 10\AppData\Roaming\powershell\Community\Terminal-Icons\devblackops_color.xml' is
denied.
Export-Clixml: Access to the path 'C:\Users\Windows 10\AppData\Roaming\powershell\Community\Terminal-Icons\devblackops_icon.xml' is
denied.
Export-Clixml: Access to the path 'C:\Users\Windows 10\AppData\Roaming\powershell\Community\Terminal-Icons\prefs.xml' is denied.
# Руководство по использованию шаблона (USAGE.md)

Инструкция для разработчиков и AI-агентов по старту, ведению и сдаче нового проекта на базе данного репозитория.

---

## 1. Концепция шаблона

`ai-template` — это **архитектурно-документационный стартовый комплект (Meta-template)**. 
Он не содержит раздутой папки `node_modules` или устаревших версий библиотек. Вместо этого он предоставляет:
1. **Строгие инженерные стандарты** (18 готовых архитектурных решений в `docs/decisions/`).
2. **Оперативные руководства** (дизайн-система, SEO, качество, типографика).
3. **Пошаговое руководство развертывания** (`docs/setup.md`) с готовыми эталонными конфигурациями под самые свежие версии пакетов.

---

> **Быстрый старт нового проекта:** Скопируйте репозиторий в чистую папку, откройте чат с AI-агентом и отправьте ему готовый промпт из начала файла [PROMPT.md](PROMPT.md). Агент запустит обязательный допрос по продукту и сам заполнит паспорт проекта!

## 2. Пошаговый сценарий работы над новым проектом

Весь процесс разработки делится на 5 последовательных шагов:

```text
[Шаг 1: Паспорт продукта] ➔ [Шаг 2: Выбор стека] ➔ [Шаг 3: Инициализация] ➔ [Шаг 4: Разработка] ➔ [Шаг 5: Приемка]
       PRODUCT.md                 ADR 0001              docs/setup.md             docs/pages/*.md        docs/quality.md
```

---

### Шаг 1: Заполнение паспорта продукта (`PRODUCT.md`)

Перед написанием любой строчки кода заполняется файл [PRODUCT.md](PRODUCT.md):
- **Цели проекта:** бизнес-цель и пользовательская цель.
- **Целевая аудитория:** кто основной клиент, ключевые боли и ожидания.
- **Пользовательские сценарии (User Flow):** шаги пользователя от первого экрана до конверсии (заявка, покупка, расчет).
- **Критические требования и ограничения:** сроки, интеграции (CRM, Telegram, эквайринг), юридические нюансы.

---

### Шаг 2: Выбор технологического профиля

На основе требований из `PRODUCT.md` определяется стек согласно [ADR 0001](docs/decisions/0001-project-architecture.md):
- **Профиль Astro (SSG / Islands) — по умолчанию:**
  - Выбирается для лендингов, корпоративных сайтов, каталогов, блогов и промо-страниц.
  - Приоритет: 0 Кб JS на первом экране, 100 баллов в Lighthouse, чистый статический HTML.
- **Профиль Next.js (App Router / SSR):**
  - Выбирается только при наличии личных кабинетов с авторизацией, сессий, динамических панелей управления или тяжелых серверных мутаций данных.

---

### Шаг 3: Инициализация кодовой базы

Откройте [docs/setup.md](docs/setup.md) и выполните команды для выбранного профиля:
1. **Создание проекта через Bun:**
   - Для Astro: `bun create astro@latest . --template minimal --typescript strict --install --no-git`
   - Для Next.js: `bun create next-app@latest . --typescript --eslint --app --src-dir --use-bun`
2. **Установка одобренного стека библиотек (ADR 0010):**
   - Команды из раздела 1 документа `docs/setup.md`.
3. **Копирование эталонных конфигураций:**
   - `tsconfig.json` (строгий режим, алиас `@/*`, `noUncheckedIndexedAccess`).
   - `.prettierrc` (единые правила форматирования).
   - `eslint.config.mjs` (Flat Config со всеми запретами и правилами хуков).
   - `.stylelintrc.mjs` (сортировка свойств и правила SCSS Modules).
   - Секция `scripts` в `package.json` (команда `bun run check` обязательна).
4. **Создание базовых SCSS-токенов:**
   - Скопировать сниппеты `_spacing.scss`, `_z-index.scss`, `_typography.scss` в `src/shared/styles/`.
5. **Создание локального `.env`:**
   - Скопировать `.env.example` в `.env` и заполнить реальными ключами.

---

### Шаг 4: Проектирование и верстка страниц

Для каждой страницы сайта:
1. **Создание спецификации страницы:**
   - Скопировать шаблон [docs/pages/page-template.md](docs/pages/page-template.md) в `docs/pages/[name].md` (например, `docs/pages/home.md`, `docs/pages/catalog.md`).
   - Заполнить метаданные (`title`, `description`) по правилам [docs/seo.md](docs/seo.md).
   - Описать структуру секций сверху вниз, указав тип компонента (статичный `.astro` или интерактивный остров) и источники данных.
2. **Разработка компонентов по FSD-Lite:**
   - Базовый UI-кит (кнопки, инпуты, модалки) ➔ в `src/shared/ui/`.
   - Бизнес-логика и формы (валидация Zod, маски) ➔ в `src/features/`.
   - Смысловые секции ➔ в `src/widgets/`.
   - Каркас ➔ в `src/layouts/BaseLayout.astro`.
   - Маршрут ➔ в `src/pages/[route].astro`.
3. **Соблюдение стандартов в процессе кода:**
   - Разметка и a11y: по [ADR 0002](docs/decisions/0002-html-standards.md) (нативный `<dialog>`, `:focus-visible`, Skip Link).
   - Стили: по [docs/design.md](docs/design.md) (8pt Grid на пикселях, семантические токены, резиновые заголовки `clamp()`, `text-wrap: balance/pretty`).
   - Типографика: по [docs/content.md](docs/content.md) («елочки», длинное тире `—`, неразрывные пробелы `&nbsp;`, правовой дисклеймер под формами).
   - Состояние: по [ADR 0006](docs/decisions/0006-state-management.md) (Nano Stores `$`, URL как SSOT).

---

### Шаг 5: Проверка качества и приемка (Definition of Done)

Перед сдачей задачи или релизом в продакшен проект проверяется по чеклисту [docs/quality.md](docs/quality.md):
1. **Запуск автоматического конвейера:**
   ```bash
   bun run check
   ```
   - Ноль ошибок Prettier.
   - Ноль ошибок и предупреждений ESLint (`--max-warnings 0`).
   - Ноль ошибок Stylelint.
   - Ноль ошибок проверки типов (`typecheck`).
2. **Запуск тестов:**
   ```bash
   bun run test
   ```
   - Успешное прохождение unit-тестов схем Zod и утилит на Vitest.
3. **Аудит мобильной верстки:**
   - Проверка отсутствия горизонтального скролла на разрешениях 360–1920 px (запрещен костыль `overflow-x: hidden` на `body`).
   - Проверка 5 состояний форм (Idle, Submitting, Success, Error, Disabled).
4. **Аудит Google Lighthouse (Mobile):**
   - Performance ≥ 90 (для Astro 95–100).
   - Accessibility ≥ 95.
   - Best Practices = 100.
   - SEO = 100.
   - Сдвиг верстки `CLS = 0.00`.
5. **Пост-деплой проверка живого сайта в браузере (ADR 0013, 0014):**
   - Google Rich Results Test (зеленый статус для сниппетов).
   - Schema.org Validator (ноль ошибок JSON-LD).
   - W3C Nu HTML Checker (ноль синтаксических ошибок разметки).

---

## 3. Правила фиксации прогресса и работы с Git

- **Фиксация статуса:** после каждого этапа обновляются файлы [STATE.md](STATE.md) (текущая фаза, блокеры, следующий шаг) и [docs/tasks.md](docs/tasks.md) (отметка выполненных пунктов).
- **Сообщения коммитов (ADR 0011):** строго по стандарту Conventional Commits на английском языке (`feat:`, `fix:`, `docs:`, `style:`, `refactor:`, `perf:`, `chore:`).
  - *Пример:* `feat(catalog): add category filter with url sync`
- **Безопасность:** категорически запрещены `git push --force` в ветку `main` и деструктивные сбросы без бэкапа.



