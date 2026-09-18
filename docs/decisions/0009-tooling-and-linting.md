# ADR 0009: Стандарты линтинга, форматирования и статического анализа

## Контекст и цели

Автоматизированный контроль качества кода предотвращает появление багов, гарантирует единообразный стиль и устраняет субъективные споры на код-ревью.
Цели архитектурного стандарта:
1. **Четкое разделение ответственности:** Prettier отвечает за форматирование (пробелы, переносы), ESLint — за логику и типы, Stylelint — за модульные стили.
2. **Современный ESLint Flat Config:** использование актуального формата `eslint.config.mjs` со строгой типизацией и правилами хуков.
3. **Порядок свойств и контроль специфичности в SCSS:** использование `stylelint-order` и запрет глубокой вложенности (максимум 3 уровня).
4. **Бескомпромиссная проверка типов:** обязательное прохождение `astro check` / `tsc --noEmit` перед любым коммитом.
5. **Единая точка входа качества:** команда `bun run check` с флагом `--max-warnings 0`.

---

## Принятые стандарты

### 1. Форматирование кода: Prettier

- **Правило невмешательства:** ESLint не проверяет правила форматирования. Все конфликты отключаются через `eslint-config-prettier`.
- **Единая конфигурация `.prettierrc`:**
  ```json
  {
    "printWidth": 120,
    "tabWidth": 2,
    "useTabs": false,
    "semi": true,
    "singleQuote": true,
    "trailingComma": "all",
    "bracketSpacing": true,
    "arrowParens": "avoid",
    "endOfLine": "lf",
    "plugins": ["prettier-plugin-astro"]
  }
  ```
- **Обязательный плагин `prettier-plugin-astro`:** обеспечивает корректное форматирование фронтматтера и разметки в файлах `.astro`.

---

### 2. Линтинг JavaScript и TypeScript: ESLint Flat Config

- **Формат конфигурации:** строго `eslint.config.mjs` (ESLint v9+).
- **Обязательный стек плагинов:**
  - `@typescript-eslint/eslint-plugin` + парсер `@typescript-eslint/parser`.
  - `eslint-plugin-astro` — для шаблонов и клиентских скриптов Astro.
  - `eslint-plugin-react-hooks` — строгий контроль правил хуков React.
  - `eslint-config-prettier` — отключение дублирующих правил стилизации.
- **Критические правила уровня `"error"`:**
  - `@typescript-eslint/no-explicit-any: "error"` — тотальный запрет типа `any`.
  - `@typescript-eslint/no-unused-vars: ["error", { "argsIgnorePattern": "^_", "varsIgnorePattern": "^_" }]` — запрет неиспользуемых переменных, кроме начинающихся с `_`.
  - `react-hooks/rules-of-hooks: "error"` — запрет вызова хуков внутри условий или циклов.
  - `react-hooks/exhaustive-deps: "error"` — запрет пропуска зависимостей в массивах хуков (защита от stale closures).
  - `no-console: ["warn", { "allow": ["warn", "error"] }]` — запрет случайных `console.log` в продакшене.

---

### 3. Линтинг стилей: Stylelint

- **Конфигурация `.stylelintrc.mjs`:**
  - Базовый набор правил: `stylelint-config-standard-scss`.
  - Плагин порядка свойств: `stylelint-order`.
- **Логический порядок CSS-свойств:**
  1. Позиционирование (`position`, `top`, `right`, `z-index`).
  2. Отображение и сетка (`display`, `flex`, `grid`, `gap`, `align-items`).
  3. Блочная модель (`width`, `height`, `margin`, `padding`, `box-sizing`).
  4. Типографика (`font-family`, `font-size`, `line-height`, `color`).
  5. Визуальное оформление (`background`, `border`, `border-radius`, `box-shadow`).
  6. Анимации и трансформации (`transform`, `transition`, `opacity`).
- **Критические ограничения:**
  - `selector-max-id: 0` — запрет селекторов по ID (`#`).
  - `selector-no-qualifying-type: true` — запрет селекторов с тегами (`div.card`, `button.primary`).
  - `max-nesting-depth: 3` — максимальная глубина вложенности SCSS не более 3 уровней.
  - `color-named: "never"` — запрет строковых названий цветов (`red`, `blue`).

---

### 4. Стандартизация команд проверки в package.json

В `package.json` проекта фиксируется единый конвейер статического анализа:

```json
{
  "scripts": {
    "lint:js": "eslint . --max-warnings 0",
    "lint:style": "stylelint \"src/**/*.{css,scss}\" --max-warnings 0",
    "lint": "bun run lint:js && bun run lint:style",
    "format": "prettier --write \"src/**/*.{ts,tsx,astro,scss,json,md}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx,astro,scss,json,md}\"",
    "typecheck": "astro check",
    "check": "bun run format:check && bun run lint && bun run typecheck"
  }
}
```

- **Флаг `--max-warnings 0`:** проект считается непрошедшим проверку, если линтер выдает хотя бы одно предупреждение.
- **Единая команда `bun run check`:** обязательный финальный шаг верификации перед коммитом и в CI/CD пайплайнах.
