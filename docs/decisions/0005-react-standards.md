# ADR 0005: Стандарты разработки на React

## Контекст и цели

React используется для интерактивных модулей (Astro React Islands и Next.js App Router).
Цели стандартов:
1. **Чистота и предсказуемость рендеринга:** запрет скрытых сайд-эффектов, корректная гигиена эффектов и защита от гонок данных.
2. **Точная типобезопасность без неймспейса React:** плоский именованный импорт типов (`import type { ... } from 'react'`), запрет префиксов `React.`.
3. **Современные паттерны React 19:** отказ от устаревших `React.FC` и `forwardRef`, передача `ref` как обычного пропса компонента.
4. **Разделение ответственности:** тонкие UI-компоненты, изоляция бизнес-логики в кастомные хуки, отказ от React Context в пользу Nano Stores и Zustand.
5. **Эргономика стилей и производительность:** компактный импорт модулей стилей (`import s from './Component.module.scss'`), условные классы через `import cx from 'clsx'`, передача динамических стилей строго через CSS Custom Properties.
6. **Отказоустойчивость:** изоляция ошибок на уровне интерактивных островов через Error Boundaries.

---

## Принятые стандарты

### 1. Объявление компонентов и типизация пропсов

- **`function` declaration — стандарт объявления компонентов:**
  ```tsx
  import cx from 'clsx';
  import type { ReactNode } from 'react';
  import s from './Card.module.scss';

  type CardProps = {
    title: string;
    children: ReactNode;
    isHighlighted?: boolean;
  };

  export function Card({ title, children, isHighlighted = false }: CardProps) {
    return (
      <article className={cx(s.card, { [s.highlighted]: isHighlighted })}>
        <h3 className={s.title}>{title}</h3>
        <div className={s.content}>{children}</div>
      </article>
    );
  }
  ```
- **Тотальный запрет `React.FC` / `React.FunctionComponent`:**
  - Запрещено объявлять компоненты через `const Button: React.FC<Props> = ...`.
  - Тип пропсов аннотируется прямо в аргументе функции.
- **Расширение нативных HTML-атрибутов — через `interface`:**
  ```tsx
  import type { ButtonHTMLAttributes, Ref } from 'react';

  export interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
    ref?: Ref<HTMLButtonElement>;
    variant?: 'primary' | 'secondary' | 'ghost';
    isLoading?: boolean;
  }
  ```
- **Чистые пропсы без наследования DOM — через `type`:**
  ```ts
  export type ModalHeaderProps = {
    title: string;
    subtitle?: string;
    onClose: () => void;
  };
  ```

---

### 2. Плоский импорт типов из React

- **Запрет префикса неймспейса `React.`:**
  - Запрещено писать `React.ReactNode`, `React.MouseEvent`, `React.Ref`, `React.ChangeEvent`.
  - Все типы импортируются напрямую через плоский именованный `import type`:
    ```ts
    import type {
      ButtonHTMLAttributes,
      CSSProperties,
      ChangeEvent,
      FormEvent,
      KeyboardEvent,
      MouseEvent,
      ReactNode,
      Ref,
    } from 'react';
    ```

---

### 3. Передача `ref` и отказ от `forwardRef`

- **`ref` как обычный пропс (React 19 / Modern TS):**
  - Запрещено оборачивать компоненты в `forwardRef`.
  - Свойство `ref` объявляется как стандартный опциональный проп интерфейса:
    ```tsx
    import type { ButtonHTMLAttributes, Ref } from 'react';

    export interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
      ref?: Ref<HTMLButtonElement>;
    }

    export function Button({ ref, children, ...restProps }: ButtonProps) {
      return (
        <button ref={ref} {...restProps}>
          {children}
        </button>
      );
    }
    ```

---

### 4. Типизация событий и обработчиков

- **Строго `e.currentTarget` для свойств слушателя:**
  - Чтение полей, атрибутов, `dataset` и состояний выполняется строго из `e.currentTarget` (элемент, на котором висит обработчик).
  - `e.target` не используется для чтения свойств, так как клик может прийтись на дочерний элемент (например, тег `<path>` SVG-иконки).
- **Точные типы событий DOM:**
  ```tsx
  const handleClick = (event: MouseEvent<HTMLButtonElement>) => { ... };
  const handleChange = (event: ChangeEvent<HTMLInputElement>) => { ... };
  const handleSubmit = (event: FormEvent<HTMLFormElement>) => {
    event.preventDefault();
  };
  const handleKeyDown = (event: KeyboardEvent<HTMLInputElement>) => { ... };
  ```
- **Разделение нейминга:**
  - Внутри компонента: `handleClick`, `handleSubmit`, `handleFilterChange`.
  - В пропсах компонента: `onClick`, `onSubmit`, `onFilterChange`.

---

### 5. Управление состоянием (State Management)

- **Ленивая инициализация стейта (Lazy Initialization):**
  - Если начальное значение стейта требует вычислений, парсинга URL или чтения хранилища — строго использовать функцию:
    ```tsx
    const [filter, setFilter] = useState(() => getInitialFilter());
    ```
  - Запрещено передавать вызов функции напрямую (`useState(getInitialFilter())`), чтобы не нагружать ререндеры.
- **Функциональные обновления стейта:**
  - Если новое состояние рассчитывается на основе предыдущего, строго использовать коллбэк `setVal(prev => ...)`:
    ```tsx
    setCount(prevCount => prevCount + 1);
    setItems(prevItems => [...prevItems, newItem]);
    setIsActive(prev => !prev);
    ```
  - Прямой вызов `setVal(newVal)` допустим только для независимых внешних данных (`setValue(e.currentTarget.value)`).

---

### 6. Архитектура хуков, чистота компонентов и сайд-эффекты

- **Тонкие UI-компоненты:**
  - Компонент отвечает за рендер и разметку. Если компонент содержит более 2-3 хуков `useState`, таймеры или запросы, логика выносится в кастомный хук в той же папке (`use-lead-form.ts`).
- **Чистота тела компонента:**
  - Запрещены сайд-эффекты в теле рендера. Любые вызовы внешних API, мутации или таймеры допустимы только в `useEffect` или обработчиках событий.
- **Запрет `useEffect` для производного состояния:**
  - Вычисляемые значения рассчитываются на лету во время рендера, либо через `useMemo` при тяжелых операциях:
    ```tsx
    // ❌ Запрещено:
    const [isValid, setIsValid] = useState(false);
    useEffect(() => {
      setIsValid(Boolean(email && name));
    }, [email, name]);

    // ✅ Правильно:
    const isValid = Boolean(email && name);
    ```
- **Гигиена и очистка `useEffect`:**
  - **Обязательный cleanup:** возврат функции отписки для слушателей событий (`window.addEventListener`), таймеров и обсерверов.
  - **Запрет `async` функции в аргументе `useEffect`:** создавать внутреннюю асинхронную функцию внутри эффекта.
  - **Совместимость со Strict Mode:** эффект обязан без сайд-эффектов переживать двойное монтирование.
  - **Запрет копирования пропсов в стейт:** для сброса состояния формы/компонента при смене сущности использовать пропс `key` (`<UserEditForm key={user.id} />`).

---

### 7. Рендеринг коллекций, ключи и условный рендер

- **Строгие правила для пропса `key`:**
  - Строго уникальный стабильный идентификатор сущности (`key={item.id}`).
  - **Тотальный запрет `key={index}`** для любых динамических, фильтруемых или сортируемых списков.
  - Для статичных списков без ID — составной ключ из контента (`key={item.href}`).
  - Запрещено генерировать UUID (`crypto.randomUUID()`) прямо в JSX во время рендера.
- **Условный рендеринг:**
  - **Тотальный запрет неявного `count && <Component />`:** при `count === 0` React отрендерит цифру `0` в DOM.
  - Использовать строго тернарный оператор или явное булево приведение:
    ```tsx
    // ✅ Безопасно:
    {items.length > 0 ? <List items={items} /> : null}
    {Boolean(items.length) && <List items={items} />}
    ```
  - Ранний возврат (Early Return) в начале компонента для состояний экрана (загрузка, ошибка, пусто).

---

### 8. Прагматичная мемоизация (`useMemo`, `useCallback`, `React.memo`)

- **Запрет слепой мемоизации:** не оборачивать элементарные функции и простые арифметические операции.
- **`useMemo` обязателен только для:**
  1. Тяжелых вычислений на массивах от 100+ элементов.
  2. Стабилизации ссылок на объекты/массивы, если они входят в зависимости других хуков.
- **`useCallback` обязателен только если:**
  1. Функция передается в компонент, оптимизированный через `React.memo`.
  2. Функция входит в массив зависимостей `useEffect`.
- **`React.memo`:** применять точечно только к тяжелым листовым компонентам (графики, комплексные таблицы) с частыми ререндерами родителя.
- **React Compiler не включается** ни в одном профиле; мемоизация выполняется вручную по правилам выше. В отдельном Next.js-проекте с тяжелым интерактивом компилятор включается флагом только по решению владельца проекта.

---

### 9. Отказ от собственного React Context

- **Запрещено объявлять собственные контексты** (`createContext`, `useContext`; контроль: ESLint `no-restricted-syntax`):
  - Исключаются каскадные ререндеры и матрешки провайдеров (Provider Hell).
  - В архитектуре Astro Islands провайдеры контекста не проникают сквозь границы изолированных островов.
- **Провайдеры сторонних библиотек разрешены**, если библиотека их требует (`QueryClientProvider` из TanStack Query, `FormProvider` из React Hook Form). Они не создают собственного контекста приложения.
- **Паттерн для Astro:** каждый остров — отдельное React-дерево, поэтому остров, которому нужен TanStack Query, оборачивается в общий `QueryProvider`, использующий единый экземпляр `queryClient` (`src/shared/api/query-client.ts`, ADR 0008 §2). Кеш общий на всю страницу.
  ```tsx
  // src/shared/api/query-provider.tsx
  import { QueryClientProvider } from '@tanstack/react-query';
  import type { ReactNode } from 'react';

  import { queryClient } from './query-client';

  type QueryProviderProps = { children: ReactNode };

  export function QueryProvider({ children }: QueryProviderProps) {
    return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>;
  }
  ```
- **Альтернативы для собственных данных:**
  - Межкомпонентный и межостровной глобальный стейт — **Nano Stores** (`$cart`, `$auth`).
  - Сложные веб-приложения на Next.js — **Zustand** с точечными селекторами.
  - Локальное состояние виджетов — явные пропсы или нативные HTML-элементы (`<details>`, `<dialog>`).

---

### 10. Формы и валидация

- **React Hook Form + Zod — стандарт для форм:**
  - Для любых форм от 2 полей используется `react-hook-form` в связке с `@hookform/resolvers/zod`.
  - Запрещено плодить множество ручных `useState` на каждое поле ввода.
  - Схемы валидации выносятся в отдельные файлы (`lead-form.schema.ts`).
- **Защита от двойной отправки (Double Submit):**
  - Кнопка подтверждения формы обязана блокироваться во время запроса: `disabled={isSubmitting}`.
- **Одиночные контролы:**
  - Для изолированных полей (поисковая строка, переключатель) допускается простой локальный `useState`.

---

### 11. Асинхронные запросы и отмена

- **Сетевые запросы выполняются только через хуки TanStack Query** ([ADR 0008](0008-forms-and-api.md) §2). Query передает `signal` в `queryFn`, поэтому запрос отменяется при смене ключа или размонтировании, и гонки данных (Race Conditions) исключены:
  ```tsx
  export function useProducts(filters: ProductFilters) {
    return useQuery({
      queryKey: productKeys.list(filters),
      queryFn: ({ signal }) => fetchProducts(filters, signal),
    });
  }
  ```
- **Запрещено** писать ручной `useEffect` + `fetch` + `useState` для загрузки данных.
- **`AbortController` вручную** нужен только в эффектах вне TanStack Query (например, разовая подписка на поток): контроллер создается внутри эффекта, а `controller.abort()` вызывается в функции очистки.

---

### 12. Контроль `children` и полиморфизм

- **Строгое назначение `children`:**
  - Пропс `children?: ReactNode` добавляется строго в контейнеры, карточки, модалки и обертки.
  - Текстовые компоненты и кнопки, где дизайн требует строго строку, должны требовать `label: string` или `text: string`.
- **Полиморфные компоненты (Discriminated Unions):**
  - Кнопки-ссылки и вариативные элементы реализуются через объединение непересекающихся типов:
    ```tsx
    type ButtonAsButton = {
      as?: 'button';
      href?: never;
      onClick?: () => void;
    };

    type ButtonAsLink = {
      as: 'link';
      href: string;
      onClick?: never;
    };

    export type ButtonProps = (ButtonAsButton | ButtonAsLink) & {
      variant?: 'primary' | 'secondary';
      children: ReactNode;
    };
    ```

---

### 13. Стилизация, SCSS Modules и `clsx`

- **Компактные стандарты импорта:**
  - Алиас для `clsx`: строго `import cx from 'clsx';`.
  - Алиас для SCSS Modules: строго `import s from './Component.module.scss';`.
  - Запрещено склеивать классы шаблонными строками с пробелами (`${s.btn} ${s.active}`).
- **Динамические стили через CSS Custom Properties:**
  - Запрещено вычислять стили инлайново в атрибуте `style`, например `width` от значения прогресса.
  - Динамика передается строго через CSS-переменные. Тип `CSSProperties` расширяется один раз на проект, поэтому приведение `as` не нужно:
    ```ts
    // src/shared/types/css-properties.d.ts
    import 'react';

    declare module 'react' {
      interface CSSProperties {
        [key: `--${string}`]: string | number | undefined;
      }
    }
    ```
    ```tsx
    <div style={{ '--progress': `${progress}%` }} className={s.bar} />
    ```

---

### 14. Файловая структура и изоляция ошибок

- **Именование файлов:**
  - Файл компонента: `Button.tsx` (`PascalCase`).
  - Папка компонента: `src/shared/ui/button/` (`kebab-case`).
  - Стили компонента: `Button.module.scss` в той же папке.
  - Типы (если вынесены): `Button.types.ts`.
  - Кастомный хук компонента: `use-button.ts`.
- **Изоляция ошибок через Error Boundaries:**
  - Каждый независимый интерактивный остров оборачивается в `ErrorBoundary` с локальным UI-фоллбэком и логированием ошибки.
  - Ошибка в одном виджете не должна приводить к белому экрану всего приложения.
- **Фасад модуля:**
  - Экспорт компонента строго именованный через `index.ts` модуля:
    ```ts
    export { Button } from './Button';
    export type { ButtonProps } from './Button.types';
    ```
