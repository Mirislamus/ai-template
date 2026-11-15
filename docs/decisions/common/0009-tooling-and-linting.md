# ADR 0009: Стандарты линтинга, форматирования и статического анализа

## Контекст и цели

Автоматизированный контроль качества кода предотвращает баги, гарантирует единый стиль и устраняет субъективные споры на код-ревью.
Цели архитектурного стандарта:
1. **Четкое разделение ответственности:** `bun fmt` отвечает за форматирование, ESLint — за логику, типы и порядок импортов, Stylelint — за модульные стили.
2. **Современный ESLint Flat Config:** формат `eslint.config.mjs` с `defineConfig` из `eslint/config` и строгими правилами.
3. **Порядок свойств и контроль специфичности в SCSS:** `stylelint-order` и ограничение вложенности (максимум 3 уровня).
4. **Автоматический контроль правил ADR:** запреты `enum`, `any`, `!`, собственного React Context, `rem`/`em`, `transition: all` и произвольного `z-index` проверяются линтерами.
5. **Единая точка входа качества:** команда `bun run check` с флагом `--max-warnings 0`.

Все эталонные конфиги и команды находятся в [docs/setup/](../../setup/): общие — [common.md](../../setup/common.md), фронтенд — [frontend.md](../../setup/frontend.md) §2, бэкенд — [backend.md](../../setup/backend.md) §2–3; этот документ фиксирует правила.

---

## Принятые стандарты

### 1. Форматирование кода: `bun fmt`

- **Форматтер — встроенный в Bun [`bun fmt`](https://bun.com/docs/format):** отдельный пакет не нужен, результат совпадает с Prettier.
- **Правило невмешательства:** ESLint не проверяет правила форматирования. Все конфликты отключаются через `eslint-config-prettier`.
- **Единственный источник настроек — корневой [.prettierrc](../../../.prettierrc)** (`printWidth: 120`, `singleQuote`, `trailingComma: all`, `endOfLine: lf`, `arrowParens: avoid`). `bun fmt` читает его как конфиг Prettier; секция `[fmt]` в `bunfig.toml` настройки не дублирует.
- **Профиль Astro:** файлы `.astro` `bun fmt` передает установленному Prettier с плагином `prettier-plugin-astro`, поэтому оба пакета ставятся только в этом профиле. В профиле Next.js и в `apps/api` Prettier не устанавливается.
- **Сортировка импортов `bun fmt` (`sortImports`) не включается:** порядок импортов задает `perfectionist/sort-imports` (§2).

---

### 2. Линтинг JavaScript и TypeScript: ESLint Flat Config

- **Формат конфигурации:** `eslint.config.mjs`, сборка через `defineConfig` (`eslint/config`).
- **Обязательный стек плагинов:** `typescript-eslint`, `eslint-plugin-astro` (профиль Astro), `eslint-plugin-react-hooks`, `eslint-plugin-perfectionist`, `eslint-config-prettier`.
- **Критические правила уровня `error`:**
  - `@typescript-eslint/no-explicit-any` — запрет `any`.
  - `@typescript-eslint/no-non-null-assertion` — запрет `!` (ADR 0004 §10).
  - `@typescript-eslint/consistent-type-imports` — `import type` для типов (ADR 0004 §8).
  - `@typescript-eslint/no-floating-promises` — запрет «забытых» промисов (правила с типами включены для `.ts` и `.tsx`).
  - `@typescript-eslint/no-unused-vars` с исключением префикса `_`.
  - `react-hooks/rules-of-hooks`, `react-hooks/exhaustive-deps`.
  - `no-restricted-syntax` — запрет `enum` (ADR 0004 §4) и собственных `createContext`/`useContext` (ADR 0005 §9).
  - `no-restricted-properties` и `no-restricted-syntax` — запрет прямого чтения `process.env` и `import.meta.env` вне `env.ts` и конфигов сборки (ADR 0004 §13).
  - `perfectionist/sort-imports` — порядок групп импортов ADR 0004 §8; автоисправление `eslint --fix`.
- `no-console` на уровне `warn` (разрешены `warn` и `error`); из-за `--max-warnings 0` `console.log` блокирует проверку.

---

### 3. Линтинг стилей: Stylelint

- **Конфигурация `.stylelintrc.mjs`:** `stylelint-config-standard-scss` + плагин порядка свойств `stylelint-order`.
- **Порядок CSS-свойств (единый список, закреплен в конфиге):**
  1. Позиционирование (`position`, `inset*`, `top`…`left`, `z-index`).
  2. Отображение и сетка (`display`, `flex*`, `grid*`, `gap`, выравнивание `place-*`, `align-*`, `justify-*`, `order`).
  3. Блочная модель (`width`/`height` и их `min`/`max`, `aspect-ratio`, `margin*` и `padding*` вместе с логическими свойствами, `box-sizing`, `overflow*`).
  4. Типографика (`font*`, `line-height`, `letter-spacing`, `text-*`, `white-space`, `overflow-wrap`, `color`).
  5. Визуальное оформление (`background*`, `border*`, `box-shadow`, `outline*`, `opacity`).
  6. Анимации и интерактив (`transform`, `transition`, `animation`, `cursor`, `pointer-events`, `user-select`).
  - Свойства вне списка (`clip-path`, `isolation`, `content` и др.) идут в конце блока. Порядок исправляется автоматически командой `stylelint --fix`.
- **Критические ограничения:**
  - `selector-max-id: 0` — запрет селекторов по ID.
  - `selector-no-qualifying-type: true` — запрет `div.card`.
  - `max-nesting-depth: 3`.
  - `color-named: never` — запрет цветов по имени.
  - `unit-disallowed-list: ['rem', 'em']` — запрет `rem` и `em` (ADR 0003 §2).
  - `declaration-property-value-allowed-list` для `z-index` — только `var(--z-*)` или `auto` (ADR 0003 §13).
  - `declaration-property-value-disallowed-list` — запрет `transition: all` (ADR 0003 §15).

---

### 4. Стандартизация команд проверки

- Команды `lint`, `format`, `typecheck`, `check` описаны в [docs/setup/frontend.md](../../setup/frontend.md) §2.4 и [docs/setup/backend.md](../../setup/backend.md) §3 и являются единой точкой входа.
- **Флаг `--max-warnings 0`:** проект считается непрошедшим проверку при любом предупреждении линтера.
- **Единая команда `bun run check`** (форматирование, линтеры, типы) — финальный шаг перед коммитом и обязательный шаг Quality Gate в CI ([ADR 0018](0018-deployment-and-caching.md) §4).
