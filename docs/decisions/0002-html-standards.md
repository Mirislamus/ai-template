# ADR 0002: Стандарты семантики, доступности и верстки HTML

## Контекст и цели

Чистая, семантическая и доступная разметка (HTML / JSX / Astro) напрямую влияет на:
1. **SEO и поисковую индексацию:** роботы оценивают дерево заголовков, микроразметку и смысловую структуру.
2. **Доступность (a11y):** сайт предсказуемо работает для пользователей со скринридерами и при навигации с клавиатуры.
3. **Стабильность верстки:** отсутствие Cumulative Layout Shift (CLS) при загрузке ассетов.
4. **Качество кода:** запрет «div-спагетти» и использование нативных возможностей браузера вместо тяжелых библиотек.
5. **Валидность:** отсутствие синтаксических ошибок спецификации W3C HTML5.

---

## Принятые стандарты

### 1. Каркас страницы и структура секций

- **Главные ориентиры страницы:**
  - Ровно один `<header>` — шапка сайта с навигацией `<nav>`.
  - Ровно один `<main id="main-content">` — основное содержимое страницы.
  - Ровно один `<footer>` — подвал сайта.
- **Иерархия заголовков:**
  - Ровно один `<h1>` на страницу (главное назначение страницы).
  - Заголовки строго следуют иерархии (`h1` → `h2` → `h3`). Запрещено пропускать уровни (например, ставить `h4` сразу после `h2`).
- **Смысловые секции (`<section>` / `<article>`):**
  - Каждый самостоятельный блок оборачивается в `<section>`.
  - **Запрещены безымянные секции:** каждая секция обязана содержать тег заголовка (`<h2>`–`<h6>`).
  - **Скрытие заголовков:** если по дизайну заголовок секции не предусмотрен визуально, он обязательно добавляется в разметку с классом `.visually-hidden` (скрыт визуально, но доступен роботам и скринридерам).

---

### 2. Кнопки против ссылок (`<a>` vs `<button>`)

Разделение строго по типу действия:
- **`<a href="...">` (Ссылка):** используется **только** для перехода по URL (страница, якорь `#contacts`, внешний ресурс, телефон `tel:`, почта `mailto:`).
  - Запрещены пустые ссылки `href="#"`, `href="javascript:void(0)"` и ссылки без `href`.
- **`<button>` (Кнопка):** используется для любого действия на странице без смены URL (открытие модалки, таба, корзины, сабмит формы).
  - Обязателен явный атрибут типа: `type="button"` (по умолчанию для действий) или `type="submit"` (для отправки форм).
- **Запрет кликабельных тегов:** категорически запрещено вешать обработчики клика (`onClick`) на `<div>`, `<span>`, `<p>` или иконки.

#### Внешние ссылки (`target="_blank"`)
- Обязателен атрибут безопасности: `rel="noopener noreferrer"`.
- Обязательно текстовое предупреждение для скринридеров о смене контекста (через `aria-label` или скрытый текст):
  ```html
  <a href="https://example.com" target="_blank" rel="noopener noreferrer" aria-label="Перейти на сайт партнера (откроется в новой вкладке)">
    Партнер
  </a>
  ```

---

### 3. Формы и поля ввода (Forms & Inputs a11y)

#### Атрибуты для `<input>`
Для мобильного UX, удобства автозаполнения и безопасности обязательны нативные атрибуты:
- **`autocomplete`:** обязателен для стандартных полей:
  - Имя: `autocomplete="name"`
  - Email: `autocomplete="email"`
  - Телефон: `autocomplete="tel"`
  - Адрес: `autocomplete="street-address"`
  - Пароль: `autocomplete="current-password"` или `autocomplete="new-password"`
- **`inputmode`:** вызов специализированной клавиатуры на смартфонах:
  - Телефон: `inputmode="tel"`
  - Коды, цифры, номера: `inputmode="numeric"`
  - Email: `inputmode="email"`
  - Дробные суммы: `inputmode="decimal"`
- **`enterkeyhint`:** индикация действия на мобильной клавиатуре:
  - Промежуточные поля: `enterkeyhint="next"`
  - Последнее поле перед отправкой: `enterkeyhint="send"` или `enterkeyhint="done"`
- **`spellcheck`:** отключать для телефонов, email, кодов: `spellcheck="false"`.

#### Связка с `<label>` и ошибки валидации
- Каждый `<input>`, `<textarea>`, `<select>` обязан иметь связанный `<label for="id">`.
  - Плейсхолдер (`placeholder`) служит только примером ввода и **не заменяет** лейбл.
  - Если по дизайну подписи нет, текст внутри `<label>` скрывается классом `.visually-hidden`.
- Текст ошибки связывается с полем через `aria-describedby="error-id"`, а на невалидное поле выставляется `aria-invalid="true"`.

```html
<div class="field">
  <label for="user-phone">Телефон для связи</label>
  <input 
    id="user-phone" 
    name="phone" 
    type="tel" 
    inputmode="tel" 
    autocomplete="tel" 
    enterkeyhint="next" 
    required 
    aria-describedby="phone-error" 
    aria-invalid="true" 
    placeholder="+7 (___) ___-__-__" 
  />
  <span id="phone-error" role="alert" class="field-error">Введите корректный номер телефона</span>
</div>
```

#### Многострочный ввод (`<textarea>`)
- Обязательны атрибуты `rows`, явные ограничения длины `maxlength`.
- Запрет горизонтального ресайза через CSS: `resize: vertical;` или `resize: none;`.

#### Выпадающие списки (`<select>`)
- Обязателен дефолтный скрытый/неактивный placeholder-option с пустым значением:
```html
<select id="service-select" name="service" required>
  <option value="" disabled selected hidden>Выберите интересующую услугу...</option>
  <option value="stand">Аренда стенда</option>
  <option value="furniture">Аренда мебели</option>
</select>
```

#### Группы чекбоксов и радиокнопок (`<fieldset>` и `<legend>`)
- Одиночный чекбокс:
```html
<label class="checkbox">
  <input type="checkbox" name="policy" required />
  <span>Согласен с политикой конфиденциальности</span>
</label>
```
- Группа опций: **обязательно** оборачивается в `<fieldset>` с подписью группы в `<legend>`:
```html
<fieldset class="radio-group">
  <legend class="radio-group__title">Способ доставки</legend>
  <label class="radio-group__item">
    <input type="radio" name="delivery" value="courier" checked />
    <span>Курьерская доставка</span>
  </label>
  <label class="radio-group__item">
    <input type="radio" name="delivery" value="pickup" />
    <span>Самовывоз со склада</span>
  </label>
</fieldset>
```

#### Защита от спама (Honeypot)
Скрытое поле для ботов скрывается через CSS-свойства:
```html
<div class="visually-hidden" aria-hidden="true">
  <label for="company-site-hp">Не заполняйте это поле, если вы человек</label>
  <input type="text" id="company-site-hp" name="site_url_hp" tabindex="-1" autocomplete="off" />
</div>
```

---

### 4. Кнопки-иконки и декоративные элементы

- **Кнопка без видимого текста (крестик, бургер, поиск):**
  - Обязателен осмысленный атрибут `aria-label="Описание действия"` на самом теге `<button>`.
  - Вложенный SVG-вектор/иконка обязана иметь `aria-hidden="true"`, чтобы скринридер не читал сырой SVG-код:
  ```html
  <button type="button" aria-label="Закрыть модальное окно" class="close-btn">
    <svg aria-hidden="true" width="24" height="24">...</svg>
  </button>
  ```

---

### 5. Изображения и предотвращение сдвигов (CLS)

- **Атрибут `alt`:**
  - Смысловые изображения: краткий, информативный текст, описывающий суть изображения.
  - Декоративные изображения (паттерны, абстрактные фоны): явный пустой атрибут `alt=""`.
- **Предотвращение CLS:**
  - Все изображения обязаны иметь явные атрибуты `width` и `height`, либо фиксироваться через CSS `aspect-ratio`.
  - В Astro используется компонент `<Image />` из `astro:assets`, в Next.js — `next/image`.

---

### 6. Списки, описания и цитаты

- **`<ul>` (Ненумерованный список):** списки преимуществ, ссылок навигации, карточек. Дочерними элементами могут быть **только** `<li>`.
- **`<ol>` (Нумерованный список):** шаги процессов, этапы оформления заказа, таймлайны («Шаг 1», «Шаг 2»).
- **`<dl>`, `<dt>`, `<dd>` (Список пар ключ-значение):** характеристики товаров (габариты, вес, материал) и контакты (Телефон/Адрес/Часы работы):
```html
<dl class="specs-list">
  <dt>Материал каркаса:</dt>
  <dd>Анодированный алюминий</dd>
  <dt>Максимальная нагрузка:</dt>
  <dd>120 кг</dd>
</dl>
```
- **`<blockquote>`:** отзывы клиентов и цитаты. Автор цитаты оформляется тегом `<cite>`.

---

### 7. Таблицы данных (`<table>`)

Таблицы обязаны быть семантичными и доступными:
- Обязательный заголовок `<caption>` (при необходимости с классом `.visually-hidden`).
- Разделение на `<thead>`, `<tbody>` и `<tfoot>`.
- Заголовки строго в `<th>` с указанием области действия: `scope="col"` или `scope="row"`.
- Обертка с возможностью горизонтального скролла с клавиатуры:

```html
<div class="table-wrapper" tabindex="0" role="region" aria-label="Сравнение тарифов аренды">
  <table class="data-table">
    <caption>Тарифные планы на выставочные стенды</caption>
    <thead>
      <tr>
        <th scope="col">Тариф</th>
        <th scope="col">Площадь</th>
        <th scope="col">Цена</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <th scope="row">Стандарт</th>
        <td>до 15 м²</td>
        <td>от 50 000 ₽</td>
      </tr>
    </tbody>
  </table>
</div>
```

---

### 8. Шаблоны нативных интерактивных компонентов

#### Модальное окно (`<dialog>`)
```html
<dialog id="lead-modal" aria-labelledby="lead-modal-title" class="modal">
  <div class="modal__content">
    <div class="modal__header">
      <h2 id="lead-modal-title">Оставить заявку</h2>
      <button type="button" aria-label="Закрыть окно" class="modal__close">
        <svg aria-hidden="true">...</svg>
      </button>
    </div>
    <div class="modal__body">
      <!-- Форма или контент -->
    </div>
  </div>
</dialog>
```

#### Спойлер / FAQ (`<details>` + `<summary>`)
```html
<details class="accordion-item">
  <summary class="accordion-item__summary">
    <span>Как происходит оплата?</span>
    <svg aria-hidden="true" class="accordion-item__icon">...</svg>
  </summary>
  <div class="accordion-item__content">
    <p>Оплата производится по безналичному расчету или картой после подтверждения заказа.</p>
  </div>
</details>
```

#### Табы / Переключатели вкладок (WAI-ARIA Tabs Pattern)
```html
<div class="tabs">
  <div role="tablist" aria-label="Категории услуг" class="tabs__list">
    <button 
      type="button" 
      role="tab" 
      id="tab-1" 
      aria-selected="true" 
      aria-controls="tabpanel-1" 
      tabindex="0"
      class="tabs__btn active"
    >
      Услуга 1
    </button>
    <button 
      type="button" 
      role="tab" 
      id="tab-2" 
      aria-selected="false" 
      aria-controls="tabpanel-2" 
      tabindex="-1"
      class="tabs__btn"
    >
      Услуга 2
    </button>
  </div>

  <div 
    role="tabpanel" 
    id="tabpanel-1" 
    aria-labelledby="tab-1" 
    tabindex="0" 
    class="tabs__panel active"
  >
    <p>Содержимое первой вкладки...</p>
  </div>

  <div 
    role="tabpanel" 
    id="tabpanel-2" 
    aria-labelledby="tab-2" 
    tabindex="0" 
    hidden 
    class="tabs__panel"
  >
    <p>Содержимое второй вкладки...</p>
  </div>
</div>
```

---

### 9. Проверка качества HTML: Валидатор W3C (validator.w3.org)

Соблюдение стандартов W3C является **обязательным критерием приёмки верстки**.

- **Ручная верификация:** Любая готовая страница проверяется через [W3C Nu HTML Checker (validator.w3.org/nu)](https://validator.w3.org/nu/).
- **Автоматическая верификация:** Запуск HTML-валидатора в CI или локально через скрипты сборщика.
- **Критерий приемки:** **0 ошибок (0 errors)** в отчете валидатора.
  - Категорически запрещены: незакрытые теги, вложенные интерактивные элементы (`<a>` внутри `<a>`, `<button>` внутри `<a>`), дублирующиеся `id`, пропущенные обязательные атрибуты (`alt`, `for`, `type`), недопустимые дочерние элементы списков.

---

### 10. Базовый утилитный класс для скрытия текста (.visually-hidden)

Каждый проект обязан иметь в глобальных стилях эталонный класс доступности:
```scss
.visually-hidden {
  position: absolute !important;
  width: 1px !important;
  height: 1px !important;
  padding: 0 !important;
  margin: -1px !important;
  overflow: hidden !important;
  clip: rect(0, 0, 0, 0) !important;
  white-space: nowrap !important;
  border: 0 !important;
}
```

---

### 11. Оптимизация медиаресурсов (Картинки и Видео)

#### Картинки (`<img>` / `<picture>`)
- **Стратегия загрузки:**
  - Картинка первого экрана (Hero / LCP): `loading="eager" fetchpriority="high" decoding="async"`.
  - Все картинки ниже первого экрана: `loading="lazy" decoding="async"`.
- **Форматы и `<picture>`:** если разметка пишется без сборщиков ассетов, используется `<picture>` с современными форматами AVIF и WebP:
```html
<picture>
  <source srcset="image.avif" type="image/avif" />
  <source srcset="image.webp" type="image/webp" />
  <img src="image.jpg" alt="Описание изображения" width="800" height="600" loading="lazy" decoding="async" />
</picture>
```

#### Видео (`<video>`)
- **Фоновое автовоспроизводимое видео (Hero-видео):**
  - Обязательна строгая связка атрибутов: `autoplay loop muted playsinline preload="metadata" poster="poster.webp"`.
  - Без `muted` и `playsinline` автоплей блокируется мобильными браузерами (iOS Safari).
- **Пользовательское видео (отзывы, демонстрации):**
  - Связка: `controls playsinline preload="metadata" poster="poster.webp"`.

---

### 12. Семантика дат, времени и контактов

#### Даты и время (`<time>`)
Любые даты, время работы, сроки акций оборачиваются в `<time>` с атрибутом `datetime` в стандарте ISO 8601:
```html
<!-- Дата публикации или события -->
<time datetime="2026-10-15">15 октября 2026</time>

<!-- Время суток -->
<time datetime="18:00">до 18:00</time>

<!-- Период времени -->
<time datetime="P3D">3 дня</time>
```

#### Контакты организации (`<address>`)
Контактная информация автора или компании в шапке/подвале/на странице контактов оборачивается в `<address>`:
```html
<address class="contacts-block">
  <a href="tel:+79991234567" class="contacts-block__link">+7 (999) 123-45-67</a>
  <a href="mailto:info@example.com" class="contacts-block__link">info@example.com</a>
  <span>г. Москва, ул. Примерная, д. 10</span>
</address>
```

#### Стандарты оформления ссылок контактов:
- **Телефон:** строго `href="tel:+79991234567"` (в `href` — международный формат с `+` и без пробелов/скобок, в тексте ссылки — читаемый формат).
- **Email:** строго `href="mailto:info@example.com"`.
- **Telegram:** `href="https://t.me/username"` с `target="_blank" rel="noopener noreferrer"`.
- **WhatsApp:** `href="https://wa.me/79991234567"` (номер в международном формате без знака `+`).

---

### 13. Доступный переход к контенту («Skip Link»)

Для соответствия стандарту доступности WCAG первой строкой внутри `<body>` обязательна скрытая ссылка для пропуска повторяющейся навигации при работе с клавиатуры:
```html
<a href="#main-content" class="skip-link">Перейти к основному контенту</a>
```

Стили для ссылки (скрыта вне экрана, всплывает при фокусе):
```scss
.skip-link {
  position: absolute;
  top: -999px;
  left: 16px;
  z-index: 10000;
  padding: 12px 20px;
  background-color: var(--color-primary, #000);
  color: #fff;
  border-radius: 4px;
  font-weight: 600;
  text-decoration: none;

  &:focus {
    top: 16px;
  }
}
```

---

### 14. Стандарт структуры документа и секции `<head>`

Каждая страница обязана содержать эталонный валидный каркас документа без дублирования тегов:

```html
<!DOCTYPE html>
<html lang="ru" dir="ltr">
  <head>
    <!-- Базовые метатеги -->
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="theme-color" content="#ffffff" />
    <meta name="format-detection" content="telephone=no" />

    <!-- SEO метатеги -->
    <title>Заголовок страницы — Название бренда</title>
    <meta name="description" content="Емкое описание страницы длиной 140–160 символов." />
    <meta name="robots" content="index, follow" />
    <link rel="canonical" href="https://example.com/current-page/" />

    <!-- Минимальный современный набор фавиконок -->
    <link rel="icon" href="/favicon.ico" sizes="32x32" />
    <link rel="icon" href="/favicon.svg" type="image/svg+xml" />
    <link rel="apple-touch-icon" href="/apple-touch-icon.png" />
    <link rel="manifest" href="/site.webmanifest" />

    <!-- Open Graph / Социальные сети -->
    <meta property="og:type" content="website" />
    <meta property="og:title" content="Заголовок страницы — Название бренда" />
    <meta property="og:description" content="Емкое описание страницы." />
    <meta property="og:url" content="https://example.com/current-page/" />
    <meta property="og:image" content="https://example.com/og/cover.jpg" />
    <meta property="og:image:width" content="1200" />
    <meta property="og:image:height" content="630" />
    <meta name="twitter:card" content="summary_large_image" />
  </head>
  <body>
    <!-- Доступный переход к контенту -->
    <a href="#main-content" class="skip-link">Перейти к основному контенту</a>

    <!-- Шапка сайта -->
    <header class="header">
      <nav class="nav" aria-label="Основная навигация">
        <!-- Логотип, ссылки, контакты -->
      </nav>
    </header>

    <!-- Основное содержимое страницы -->
    <main id="main-content">
      <!-- Смысловые секции -->
    </main>

    <!-- Подвал сайта -->
    <footer class="footer">
      <!-- Дополнительная навигация, реквизиты, копирайт -->
    </footer>
  </body>
</html>
```

#### Строгие правила для секции `<head>`:
- **`lang` обязателен:** тег `<html>` обязан иметь атрибут языка (например, `ru`, `en`, `uz`).
- **Запрет блокировки зума:** в `viewport` категорически запрещено использовать `user-scalable=no`, `maximum-scale=1.0` (грубое нарушение доступности WCAG, блокирующее масштабирование для слабовидящих людей).
- **Каноникал:** тег `<link rel="canonical">` обязан указывать абсолютный URL с протоколом `https://` и согласованным завершающим слешем (trailing slash).
- **`format-detection`:** мета-тег `<meta name="format-detection" content="telephone=no" />` предотвращает неконтролируемое автооборачивание произвольных цифр в ссылки на смартфонах (звонки оформляются строго через явный тег `<a href="tel:...">`).

---

### 6. Архитектура модальных окон и диалогов (<dialog>)

- **Использование нативного `<dialog>` вместо самодельных `<div>`:**
  - Открытие выполняется строго через метод `.showModal()` (а не `.show()`). Это автоматически переносит диалог в верхний системный слой браузера (Top Layer), делает остальную страницу неактивной (`inert`), обеспечивает фокус-трап и нативное закрытие по клавише `ESC`.
- **Закрытие по клику на фон (Backdrop Click):**
  - Реализуется через проверку попадания клика в координаты контента:
    ```ts
    dialog.addEventListener('click', event => {
      const rect = dialog.getBoundingClientRect();
      const isInside =
        event.clientX >= rect.left &&
        event.clientX <= rect.right &&
        event.clientY >= rect.top &&
        event.clientY <= rect.bottom;
      if (!isInside) dialog.close();
    });
    ```
- **Блокировка скролла без сдвига макета (Zero Layout Shift):**
  - В глобальных стилях для `<html>` задается `scrollbar-gutter: stable;`.
  - При открытии окна на `<body>` вешается класс блокировки скролла (`overflow: hidden;`), при этом страница не прыгает вправо на ширину исчезнувшей полосы прокрутки.
- **Возврат фокуса на элемент-триггер:**
  - При закрытии диалога фокус автоматически возвращается на кнопку, вызвавшую открытие окна.
- **Мобильная шторка (Bottom Sheet):**
  - На мобильных устройствах (`max-width: 768px`) `<dialog>` адаптируется стилями как шторка, прижатая к нижнему краю экрана со скруглением верхних углов.

---

### 7. Доступность клавиатурного фокуса (:focus-visible) и Skip Link

- **Обязательный Skip Link (Ссылка пропуска навигации):**
  - Самый первый интерактивный элемент внутри `<body>`:
    ```html
    <a href="#main-content" class="skip-link">Перейти к основному контенту</a>
    ```
  - Визуально скрыт по умолчанию, плавно выезжает сверху при первом нажатии клавиши `Tab`. Позволяет пользователям с клавиатуры мгновенно перейти к чтению содержимого, минуя повторяющиеся ссылки шапки.
- **Категорический запрет `outline: none` без замены:**
  - Снятие обводки фокуса у интерактивных элементов без альтернативного оформления запрещено (нарушение стандарта WCAG 2.4.7).
- **Стандарт оформления фокуса:**
  - Использовать селектор `:focus-visible` (срабатывает только при навигации с клавиатуры, не затрагивая клики мыши):
    ```scss
    :focus-visible {
      outline: 2px solid var(--color-primary, #3b82f6);
      outline-offset: 2px;
    }
    ```
