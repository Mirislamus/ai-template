# ADR 0017: Стандарты анимаций, микровзаимодействий и доступности

## Контекст и цели

Анимации оживляют интерфейс и направляют внимание пользователя, но неграмотная реализация приводит к лагам (падение FPS), перерасходу батареи мобильных устройств и дискомфорту людей с вестибулярными расстройствами.
Цели архитектурного стандарта:
1. **Стабильные 60/120 FPS:** анимируются только аппаратные GPU-свойства (`transform`, `opacity`), тяжелые пересчеты макета (Layout/Reflow) запрещены.
2. **Доступность:** обязательная поддержка `prefers-reduced-motion` на уровне CSS и JS.
3. **Легковесный Scroll Reveal без библиотек:** единый `IntersectionObserver` и CSS Transitions вместо AOS и GSAP.
4. **Защита от FOUC и пустых экранов:** контент по умолчанию полностью видим; скрытие перед анимацией включается только атрибутом состояния `[data-revealed="false"]`.

---

## Принятые стандарты

### 1. Аппаратное ускорение (GPU) и запрещенные свойства

- **Разрешено анимировать:**
  - `transform` (`translate`, `scale`, `rotate`) — положение, масштаб и угол.
  - `opacity` — появление и растворение элементов.
  - Эти свойства обрабатываются видеочипом на этапе композитинга без пересчета геометрии страницы.
- **Категорически запрещено анимировать:**
  - Размеры и отступы: `width`, `height`, `margin`, `padding`.
  - Позиционирование: `top`, `left`, `right`, `bottom`.
  - Тяжелые фильтры: `filter: blur()` на больших блоках.
- **Единственное исключение — раскрытие аккордеонов** через нативный CSS Grid (зафиксировано в [ADR 0003](0003-scss-standards.md) §15):
  ```scss
  .accordion-content {
    display: grid;
    grid-template-rows: 0fr;
    transition: grid-template-rows var(--motion-base) var(--ease-out);

    &.is-open {
      grid-template-rows: 1fr;
    }

    > .inner {
      overflow: hidden;
    }
  }
  ```
  Либо нативный HTML-элемент `<details>`.
- **Свойство `will-change`:** используется строго точечно и временно; вешать его на десятки статичных элементов запрещено.
- Длительности и кривая плавности берутся из токенов `--motion-fast`, `--motion-base`, `--motion-slow`, `--ease-out`.

---

### 2. Вестибулярный комфорт: `prefers-reduced-motion`

- **Глобальный сброс в `src/shared/styles/global.scss`** (исключение из запрета `!important`, [ADR 0003](0003-scss-standards.md) §7):
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

- **Атрибут `data-reveal` запрещен на элементах первого экрана** (Hero, шапка, LCP-изображение): скрытие блока с LCP-элементом ухудшает метрики. Reveal применяется только к секциям ниже первого экрана.
- **Изолированный атрибут состояния `data-revealed`:**
  - В разметке блок помечается атрибутом `data-reveal`:
    ```html
    <section data-reveal class="section">...</section>
    ```
  - В стилях элемент по умолчанию **видим**; скрытие применяется, только когда JS взял блок под наблюдение:
    ```scss
    [data-reveal][data-revealed='false'] {
      opacity: 0;
      transform: translateY(var(--space-6));
    }

    [data-reveal][data-revealed='true'] {
      opacity: 1;
      transform: translateY(0);
      transition:
        opacity var(--motion-slow) var(--ease-out),
        transform var(--motion-slow) var(--ease-out);
    }
    ```
  - `transition` задается у состояния `'true'`: браузер берет параметры перехода из нового состояния, поэтому появление анимируется, а начальное скрытие при старте скрипта происходит мгновенно, без мигания.
- **Единый `IntersectionObserver` на всю страницу** (`src/shared/lib/reveal/init-reveal.ts`, подключается в `BaseLayout`):
  ```ts
  const observer = new IntersectionObserver(
    entries => {
      for (const entry of entries) {
        if (!entry.isIntersecting || !(entry.target instanceof HTMLElement)) continue;
        entry.target.dataset.revealed = 'true';
        observer.unobserve(entry.target); // Однократное срабатывание
      }
    },
    { threshold: 0.1, rootMargin: '0px 0px -40px 0px' },
  );

  for (const element of document.querySelectorAll<HTMLElement>('[data-reveal]')) {
    element.dataset.revealed = 'false';
    observer.observe(element);
  }
  ```
- **Гарантия надежности:** если JS отключен или сломан, селектор `[data-revealed='false']` не появляется в DOM, и контент остается доступным.
