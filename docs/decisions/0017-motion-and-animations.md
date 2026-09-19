# ADR 0017: Стандарты анимаций, микровзаимодействий и доступности

## Контекст и цели

Анимации оживляют интерфейс и направляют внимание пользователя, но неграмотная реализация приводит к лагам (падение FPS), перерасходу батареи мобильных устройств и дискомфорту людей с вестибулярными расстройствами.
Цели архитектурного стандарта:
1. **Стабильные 60/120 FPS:** строгая анимация только аппаратных GPU-свойств (`transform`, `opacity`) и запрет тяжелых пересчетов макета (Layout/Reflow).
2. **Доступность (Accessibility):** обязательная поддержка медиа-запроса `prefers-reduced-motion` на уровне CSS и JS.
3. **Легковесный Scroll Reveal без библиотек:** отказ от тяжелых пакетов (AOS, GSAP) в пользу единого синглтона `IntersectionObserver` и CSS Transitions.
4. **Защита от FOUC и пустых экранов:** контент по умолчанию полностью видим; скрытие перед анимацией активируется только через атрибут состояния `[data-revealed="false"]`.

---

## Принятые стандарты

### 1. Аппаратное ускорение (GPU) и запрещенные свойства

- **Разрешено анимировать:**
  - `transform` (`translate`, `scale`, `rotate`) — изменение положения, масштаба и угла.
  - `opacity` — появление и растворение элементов.
  - *Почему:* эти свойства обрабатываются видеочипом на этапе композитинга без пересчета геометрии страницы.
- **Категорически запрещено анимировать в CSS и JS:**
  - Размеры и отступы: `width`, `height`, `margin`, `padding`.
  - Позиционирование: `top`, `left`, `right`, `bottom`.
  - Тяжелые фильтры: `filter: blur()` на больших блоках.
  - *Причина:* вызывают непрерывный перерасчет геометрии всего документа (Layout/Reflow), приводящий к резкому падению частоты кадров на мобильных устройствах.
- **Анимация высоты аккордеонов (Спойлеров):**
  - Реализуется через нативный CSS Grid:
    ```scss
    .accordion-content {
      display: grid;
      grid-template-rows: 0fr;
      transition: grid-template-rows 0.3s ease-out;

      &.is-open {
        grid-template-rows: 1fr;
      }

      > .inner {
        overflow: hidden;
      }
    }
    ```
  - Либо через нативный HTML-элемент `<details>`.
- **Свойство `will-change`:**
  - Используется строго точечно и временно. Запрещено вешать `will-change` глобально на десятки статичных элементов.

---

### 2. Вестибулярный комфорт: `prefers-reduced-motion`

- **Глобальный сброс в `src/styles/base.scss`:**
  - Обязательное отключение анимаций для пользователей с системным ограничением движения:
    ```scss
    @media (prefers-reduced-motion: reduce) {
      *,
      *::before,
      *::after {
        animation-duration: 0.01ms !important;
        animation-iteration-count: 1 !important;
        transition-duration: 0.01ms !important;
        scroll-behavior: auto !important;
      }
    }
    ```
- **Проверка в JavaScript перед запуском анимаций:**
  ```ts
  const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
  if (prefersReducedMotion) {
    element.dataset.revealed = 'true';
    return;
  }
  ```

---

### 3. Появление блоков при скролле (Scroll Reveal)

- **Изолированный дата-атрибут состояния `data-revealed`:**
  - В разметке блок помечается атрибутом `data-reveal`:
    ```html
    <section data-reveal class="section">...</section>
    ```
  - В стилях элемент по умолчанию **видим**. Скрытие применяется только в момент, когда JS взял блок под наблюдение:
    ```scss
    [data-reveal][data-revealed='false'] {
      opacity: 0;
      transform: translateY(24px);
      transition: opacity 0.5s ease, transform 0.5s cubic-bezier(0.16, 1, 0.3, 1);
    }

    [data-reveal][data-revealed='true'] {
      opacity: 1;
      transform: translateY(0);
    }
    ```
- **Единый синглтон `IntersectionObserver`:**
  - Создается ровно один экземпляр обсервера на всю страницу:
    ```ts
    const observer = new IntersectionObserver(
      entries => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            (entry.target as HTMLElement).dataset.revealed = 'true';
            observer.unobserve(entry.target); // Однократное срабатывание
          }
        });
      },
      { threshold: 0.1, rootMargin: '0px 0px -40px 0px' },
    );

    document.querySelectorAll('[data-reveal]').forEach(el => {
      (el as HTMLElement).dataset.revealed = 'false';
      observer.observe(el);
    });
    ```
- **Гарантия надежности:** если у пользователя отключен или сломан JS, селектор `[data-revealed='false']` никогда не появится в DOM, и контент останется на 100% доступным.

