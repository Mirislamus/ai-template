# ADR 0003: Стандарты стилизации (SCSS Modules и дизайн-токены)

## Контекст и цели

Стилизация проекта обязана быть модульной, предсказуемой, производительной и удобной для поддержки.
Цели стандарта:
1. **Изоляция:** исключить глобальные конфликты классов и перебивания селекторов (CSS Modules).
2. **Предсказуемость единиц:** строгий отказ от `rem` и `em` в пользу точного `px` и современных адаптивных единиц (`vw`, `vh`, `dvh`, `%`).
3. **Mobile-First адаптивность:** базовые стили пишутся для мобильных устройств, расширение идет вверх через Range Syntax (`@media (width >= ...)`).
4. **Устранение тач-артефактов:** предотвращение залипания `:hover` на смартфонах.
5. **Централизация дизайн-токенов:** единый источник правды для цветов, отступов, радиусов, теней, слоев и анимаций через CSS Custom Properties. Имена и смысл токенов — [docs/design.md](../../design.md), эталонный файл — [docs/setup/frontend.md](../../setup/frontend.md) §4.

---

## Архитектура файлов стилей (`src/shared/styles/`)

Глобальная база стилей состоит строго из трех файлов без лишних подпапок:

```text
src/shared/styles/
├── _tokens.scss  # Все дизайн-токены: переменные :root (цвета, шрифты, радиусы, отступы, z-index, анимации)
├── _tools.scss   # Sass-переменные брейкпоинтов и миксины (up, down, hover, reduced-motion, focus-ring и др.)
└── global.scss   # @layer reset/base, стили html/body, типографика, .container, .visually-hidden, .skip-link
```

Локальные стили компонентов хранятся рядом с компонентом в файле `[Component].module.scss`. Инструменты подключаются через алиас: `@use '@/shared/styles/tools' as *;`.

---

## Принятые стандарты и правила

### 1. Адаптивность: Mobile-First и Range Syntax

Стили пишутся по принципу прогрессивного улучшения:
- **Базовые стили компонента** создаются для мобильных экранов (нижняя граница поддержки — 360px).
- Расширение для больших экранов оформляется миксинами `up()` и `down()` из `_tools.scss`, которые раскрываются в нативный Range Syntax (`@media (width >= 768px)`). Прямые числовые значения в медиазапросах запрещены.

```scss
@use '@/shared/styles/tools' as *;

// Базовые стили для мобильных (от 0px)
.grid {
  display: flex;
  flex-direction: column;
  gap: var(--space-4);

  // Планшеты (от 768px)
  @include up(md) {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: var(--space-6);
  }

  // Ноутбуки (от 1024px)
  @include up(lg) {
    grid-template-columns: repeat(3, 1fr);
  }

  // Десктоп (от 1280px)
  @include up(xl) {
    grid-template-columns: repeat(4, 1fr);
    gap: var(--space-8);
  }
}
```

#### Шкала брейкпоинтов (единственная в проекте):
- `sm: 576px` — большие смартфоны (горизонтальная ориентация);
- `md: 768px` — планшеты (граница переключения мобильного меню на десктопную навигацию);
- `lg: 1024px` — ноутбуки и небольшие экраны;
- `xl: 1280px` — стандартный десктоп (максимальная ширина контейнера);
- `2xl: 1440px` — широкие мониторы.

Брейкпоинты объявляются Sass-переменными в `_tools.scss` (CSS-переменные внутри `@media` не работают). Миксин `down(md)` раскрывается в `@media (width < 768px)`, поэтому `up(md)` и `down(md)` никогда не пересекаются. В JS-атрибутах (`client:media`) используется тот же синтаксис: `(width < 768px)`.

---

### 2. Единицы измерения: px и адаптивные единицы

- **`em` и `rem` категорически запрещены** (контроль: Stylelint `unit-disallowed-list`): исключает накопление ошибок масштабирования шрифтов и непредсказуемые скачки размеров.
- **`px`:** основной стандарт для размеров шрифтов, отступов (padding, margin, gap), границ (border), радиусов скругления. Значения отступов берутся из токенов `var(--space-*)`.
- **Адаптивные единицы:**
  - `%` — для относительной ширины контейнеров и колонок.
  - `vw` / `vh` — для привязки к размеру экрана.
  - `dvh` (Dynamic Viewport Height) и `svh` — для высоты полноэкранных блоков на мобильных браузерах.
  - `cqw` / `cqh` — для Container Queries при изоляции карточек.
- Единицы времени (`ms`, `s`) и безразмерный `line-height` разрешены.

---

### 3. Именование классов в SCSS Modules

- Имя файла уже изолирует компонент. **БЭМ-префиксы внутри модулей запрещены.**
- Используются короткие, читаемые, плоские имена классов:
```scss
// Hero.module.scss
.root {
  padding-block: var(--space-16);
}

.title {
  font-size: var(--font-size-h1);
  font-weight: 700;
}

.button {
  display: inline-flex;

  &.active {
    background-color: var(--color-accent);
  }
}
```
В JSX: `className={s.title}`, `className={cx(s.button, isActive && s.active)}`.

---

### 4. Дизайн-токены: CSS Custom Properties

Все переменные объявляются нативными CSS Custom Properties в `src/shared/styles/_tokens.scss`:

- **Полный перечень и семантика токенов** — [docs/design.md](../../design.md): `--space-*`, `--color-*` (семантические роли), `--font-size-*`, `--line-height-*`, `--radius-*`, `--shadow-*`, `--z-*`, `--motion-*`, `--container-*`.
- **Цвета:** объявляются в HEX или RGB для прямого копирования из Figma. Прямые HEX-значения в модулях компонентов запрещены, используются только токены.
- **Прозрачность:** формируется на лету через `color-mix()`:
  ```scss
  background-color: color-mix(in srgb, var(--color-accent) 12%, transparent);
  ```
- **Sass-переменные (`$var`):** разрешены только для build-time значений (брейкпоинты для медиазапросов).

---

### 5. Обязательный миксин для :hover на смартфонах

Для исключения «залипания» hover-эффектов при тапе пальцем все наведения оборачиваются в единый миксин `@include hover`:

```scss
// src/shared/styles/_tools.scss
@mixin hover {
  @media (hover: hover) and (pointer: fine) {
    @content;
  }
}
```

Применение в компонентах:
```scss
.link {
  color: var(--color-text-primary);
  transition: color var(--motion-fast) var(--ease-out);

  @include hover {
    &:hover {
      color: var(--color-accent);
    }
  }

  &:active {
    color: var(--color-accent-hover);
  }
}
```

---

### 6. Уважение к доступности: prefers-reduced-motion

Все анимации, плавные переходы и скроллы обязаны отключаться для пользователей, чувствительных к движению. Глобальный сброс лежит в `global.scss` (ADR 0017 §2), а для точечных случаев есть миксин:

```scss
// src/shared/styles/_tools.scss
@mixin reduced-motion {
  @media (prefers-reduced-motion: reduce) {
    @content;
  }
}
```

---

### 7. Запрет !important

Использование `!important` категорически **запрещено**, кроме:
1. Утилитного класса `.visually-hidden` (гарантия скрытия от экрана).
2. Глобального сброса анимаций в блоке `prefers-reduced-motion` в `global.scss` (ADR 0017 §2).
3. Вынужденного перебивания жестких инлайн-стилей сторонних виджетов (карты, плееры).

---

### 8. Каскадные слои CSS (@layer)

В `src/shared/styles/global.scss` глобальные стили сброса и базовые стили организуются в слои:

```scss
@layer reset, base;

@layer reset {
  *,
  *::before,
  *::after {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }
}

@layer base {
  html {
    scrollbar-gutter: stable;
    background-color: var(--color-bg-page);
    color: var(--color-text-primary);
  }
}
```

- **Стили CSS Modules не входят ни в один слой** (они попадают в общий каскад без слоя) и по правилам каскада всегда сильнее любых слоев. Поэтому слои используются только для сброса и базовых стилей: компонент всегда может их переопределить без роста специфичности.
- **Утилиты** (`.container`, `.visually-hidden`, `.skip-link`) объявляются в `global.scss` вне слоев:
```scss
.container {
  width: 100%;
  max-width: var(--container-max-width);
  margin-inline: auto;
  padding-inline: var(--container-gutter);
}
```
- **Запрещено** маскировать горизонтальный скролл через `overflow-x: hidden/clip` на `html` или `body` (docs/quality.md §1): переполнение устраняется в конкретном блоке.

---

### 9. Порядок сортировки CSS-свойств

Порядок свойств внутри селектора закреплен единым списком в конфиге Stylelint ([docs/setup/frontend.md](../../setup/frontend.md) §2.3, правило `order/properties-order`; описание — [ADR 0009](../common/0009-tooling-and-linting.md) §3). Группы: позиционирование, отображение и сетка, блочная модель, типографика, визуальное оформление, анимации и интерактив; затем вложенные состояния и медиазапросы.

Пример эталонного оформления правила:
```scss
@use '@/shared/styles/tools' as *;

.card {
  /* 1. Позиционирование */
  position: relative;
  z-index: var(--z-base);

  /* 2. Отображение и сетка */
  display: flex;
  flex-direction: column;
  gap: var(--space-4);

  /* 3. Блочная модель */
  width: 100%;
  padding: var(--space-6);

  /* 4. Типографика */
  color: var(--color-text-primary);

  /* 5. Визуальное оформление */
  background-color: var(--color-bg-surface);
  border: 1px solid var(--color-border-subtle);
  border-radius: var(--radius-md);

  /* 6. Анимации и интерактив */
  transition:
    border-color var(--motion-fast) var(--ease-out),
    box-shadow var(--motion-fast) var(--ease-out);

  /* 7. Состояния и адаптивность */
  @include hover {
    &:hover {
      border-color: var(--color-border-strong);
      box-shadow: var(--shadow-md);
    }
  }

  @include up(md) {
    padding: var(--space-8);
  }
}
```

---

### 10. Стандарты типографики и локальных шрифтов

- **Только локальные шрифты:** файлы хранятся в `public/fonts/` и подключаются по URL `/fonts/...`. Использование сторонних CDN (Google Fonts) через внешний `<link>` или CSS `@import` запрещено ради скорости и приватности.
- **Формат `.woff2`:** используется исключительно компактный формат.
- **Обязательный `font-display: swap`:** текст отображается системным шрифтом до окончания загрузки кастомного (устранение FOIT).
- **Ограничение начертаний:** подключаются только используемые веса (например, Regular 400 и SemiBold 600).

Пример подключения (`global.scss`):
```scss
@font-face {
  font-family: 'Inter';
  font-weight: 400;
  font-style: normal;
  font-display: swap;
  src: url('/fonts/inter-regular.woff2') format('woff2');
}

@font-face {
  font-family: 'Inter';
  font-weight: 600;
  font-style: normal;
  font-display: swap;
  src: url('/fonts/inter-semibold.woff2') format('woff2');
}
```

---

### 11. Флюидная адаптивная типографика (clamp)

Крупные заголовки настраиваются через нативную функцию `clamp(min, preferred, max)` в токенах `_tokens.scss`:

- **Минимальный порог (`px`):** гарантирует читаемость на узких экранах (от 360px).
- **Динамический множитель (`vw` + `px`):** плавное масштабирование без скачков.
- **Максимальный порог (`px`):** ограничивает рост шрифта на больших мониторах.
- **Базовый текст:** фиксируется на `16px` (Safari зумит инпуты меньше 16px).

Значения токенов `--font-size-h1`…`--font-size-xs` и `--line-height-*` — [docs/design.md](../../design.md) §4.

---

### 12. Правило выноса стилей в миксины (_tools.scss)

Любой повторяющийся состав CSS-деклараций (от 3–4 строк), встречающийся в двух или более компонентах, обязан выноситься в миксин в `src/shared/styles/_tools.scss`.

Обязательный базовый пул переиспользуемых миксинов:

```scss
// 1. Доступное кольцо фокуса клавиатуры
@mixin focus-ring {
  outline: 2px solid var(--color-accent);
  outline-offset: 2px;
}

// 2. Обрезка текста с многоточием на N строк
@mixin line-clamp($lines: 1) {
  display: -webkit-box;
  -webkit-line-clamp: $lines;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

// 3. Сброс дефолтных стилей кнопки
@mixin button-reset {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  border: none;
  background: none;
  font: inherit;
  color: inherit;
  cursor: pointer;
}

// 4. Тонкий кастомный скроллбар
@mixin custom-scrollbar {
  scrollbar-width: thin;
  scrollbar-color: var(--color-border-subtle) transparent;

  &::-webkit-scrollbar {
    width: 6px;
    height: 6px;
  }

  &::-webkit-scrollbar-track {
    background: transparent;
  }

  &::-webkit-scrollbar-thumb {
    background-color: var(--color-border-subtle);
    border-radius: var(--radius-full);
  }
}
```

Миксины `up()`, `down()`, `hover` и `reduced-motion` описаны в §1, §5 и §6.

---

### 13. Системная иерархия Z-Index

- **Единая шкала (в `_tokens.scss`, других шкал в проекте нет):**
```scss
:root {
  --z-base: 1;             /* Базовый приподнятый контент */
  --z-dropdown: 100;       /* Выпадающие списки, подсказки ввода */
  --z-sticky: 200;         /* Прилипающая шапка */
  --z-drawer: 300;         /* Мобильное меню-шторка */
  --z-modal-backdrop: 400; /* Оверлей под модальным окном */
  --z-modal: 500;          /* Модальные окна */
  --z-popover: 600;        /* Поповеры и контекстные меню */
  --z-toast: 700;          /* Уведомления (Sonner) */
  --z-tooltip: 800;        /* Тултипы и Skip Link поверх всего */
}
```
- **Категорический запрет произвольных чисел в `z-index`:** значение задается строго через `var(--z-*)` (контроль: Stylelint `declaration-property-value-allowed-list`).
- **Изоляция контекста наложения:** для сложных независимых виджетов использовать `isolation: isolate;`, чтобы внутренние слои не конфликтовали с глобальным деревом.

---

### 14. Шкала радиусов и теней

Вместо произвольных значений используются системные токены `--radius-sm/md/lg/xl/full` и `--shadow-sm/md/lg` (мягкие тени с альфа-каналом 4–8%). Значения — [docs/design.md](../../design.md) §5.

---

### 15. Производительность анимаций и переходов

Для исключения лагов и просадки FPS на мобильных устройствах действуют строгие правила:

1. **Запрет `transition: all`:** анимируются только явно перечисленные свойства.
2. **Запрет анимации геометрии:** запрещено анимировать `width`, `height`, `top`, `left`, `margin`, `padding` (вызывают reflow). Положение и масштаб меняются через `transform: translate(...) / scale(...)`. **Единственное исключение:** раскрытие аккордеонов через `grid-template-rows: 0fr → 1fr` (ADR 0017 §1).
3. **Безопасный список для анимаций:** `transform`, `opacity`, `color`, `background-color`, `border-color`, `box-shadow`.
4. **Единые тайминги и функция плавности (токены):**
```scss
:root {
  --motion-fast: 150ms;                        /* Ховеры, активные состояния кнопок */
  --motion-base: 250ms;                        /* Дропдауны, табы, спойлеры */
  --motion-slow: 350ms;                        /* Модальные окна, крупные шторки */
  --ease-out: cubic-bezier(0.22, 1, 0.36, 1);  /* Естественное физическое затухание */
}
```
