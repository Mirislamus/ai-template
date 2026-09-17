Failed to write init script: open C:\Users\Windows 10\AppData\Local\Packages\ohmyposh.cli_96v55e8n804z4\LocalCache\Local\oh-my-posh\init.814522496948324317.ps1: Access is denied.
Export-Clixml: Access to the path 'C:\Users\Windows 10\AppData\Roaming\powershell\Community\Terminal-Icons\devblackops_color.xml' is
denied.
Export-Clixml: Access to the path 'C:\Users\Windows
10\AppData\Roaming\powershell\Community\Terminal-Icons\devblackops_light_color.xml' is denied.
Export-Clixml: Access to the path 'C:\Users\Windows 10\AppData\Roaming\powershell\Community\Terminal-Icons\devblackops_icon.xml' is
denied.
Export-Clixml: Access to the path 'C:\Users\Windows 10\AppData\Roaming\powershell\Community\Terminal-Icons\prefs.xml' is denied.
Failed to write init script: open C:\Users\Windows 10\AppData\Local\Packages\ohmyposh.cli_96v55e8n804z4\LocalCache\Local\oh-my-posh\init.814522496948324317.ps1: Access is denied.
Export-Clixml: Access to the path 'C:\Users\Windows
10\AppData\Roaming\powershell\Community\Terminal-Icons\devblackops_light_color.xml' is denied.
Export-Clixml: Access to the path 'C:\Users\Windows 10\AppData\Roaming\powershell\Community\Terminal-Icons\devblackops_color.xml' is
denied.
Export-Clixml: Access to the path 'C:\Users\Windows 10\AppData\Roaming\powershell\Community\Terminal-Icons\devblackops_icon.xml' is
denied.
Export-Clixml: Access to the path 'C:\Users\Windows 10\AppData\Roaming\powershell\Community\Terminal-Icons\prefs.xml' is denied.
# Архитектура системы

## Обзор и подход: Lite FSD

В проекте используется упрощенная и практичная версия Feature-Sliced Design (Lite FSD). Она исключает избыточные слои (`processes`, `entities`), оставляя только понятную и строгую иерархию для веб-сайтов и приложений.

## Структура слоев (сверху вниз)

```text
src/
├── app/        # Инициализация, роутинг, глобальные провайдеры и стили
├── pages/      # Страницы сайта (композиция виджетов и секций)
├── widgets/    # Крупные самостоятельные блоки и секции (Header, Footer, Hero, FAQ)
├── features/   # Интерактивные действия пользователя и формы (send-lead, switch-theme, callback)
└── shared/     # Переиспользуемый базис без привязки к конкретному контексту
    ├── ui/     # Атомарный UI-кит (кнопки, инпуты, карточки, модалки)
    ├── api/    # Базовые клиенты, эндпоинты, отправка запросов
    ├── lib/    # Утилиты, хелперы, кастомные хуки
    └── types/  # Общие типы данных
```

### Зоны ответственности слоев

1. **app:** точка входа приложения, конфигурация провайдеров, глобальные стили и шрифты.
2. **pages:** собирает страницу из готовых секций (`widgets`) и настраивает SEO/метатеги.
3. **widgets:** визуально цельные блоки страницы (секции лендинга, шапка, подвал). Виджет может использовать несколько `features` и элементы `shared`.
4. **features:** законченные пользовательские действия с бизнес-логикой (форма заявки, фильтрация, переключатель языка). Может переиспользоваться в разных виджетах.
5. **shared:** фундамент проекта, не содержащий бизнес-специфики конкретного блока.

## Главное правило импортов

**Импорты разрешены только строго сверху вниз:**
`app` → `pages` → `widgets` → `features` → `shared`.

- Нижний слой ничего не знает о верхнем.
- **Запрещены горизонтальные импорты:** модуль из `features` не может импортировать другой модуль из `features`. Модуль из `widgets` не импортирует соседний `widgets`. Если логика нужна обоим — она выносится в `shared`.

## Движение данных и формы

- **Формы (`features`):** содержат валидацию, локальное состояние полей, вызов API из `shared/api` и показ статусов (loading, success, error).
- **Состояние UI:** хранится максимально локально внутри конкретного компонента или фичи.
- **Глобальное состояние:** выносится на уровень `app` только при реальной межстраничной необходимости (корзина, сессия, глобальные уведомления).

## Технические стандарты и соглашения

- **Разметка (HTML):** семантическая верстка (`main`, `section`, `header`, `footer`), доступность (a11y).
- **Стилизация (CSS / SCSS):** изоляция стилей компонентов (CSS Modules / BEM), использование токенов и переменных из `shared`.
- **Код и логика (JS / TS / React):** строгая типизация, публичный API каждого слайса через `index.ts` (Public API).
- **Архитектурные решения:** нестандартные или стек-специфичные решения фиксируются в `docs/decisions/`.

## Архитектурные ограничения

- Не создавать лишние слои (`entities` подключать только при наличии сложной доменной модели с множеством методов).
- Не делать прямых API-запросов из слоев `widgets` или `pages` в обход `features` или `shared/api`.
- Сохранять модульность: удаление виджета или фичи не должно ломать остальной проект.

## Архитектурные решения (ADR)

Ссылки на принятые решения в каталоге `docs/decisions/`:
- `docs/decisions/0001-init.md` — Выбор Lite FSD и стартового стека.
- `docs/decisions/0002-html-standards.md` — Стандарты семантики, доступности и верстки HTML.
- `docs/decisions/0003-scss-standards.md` — Стандарты стилизации (SCSS Modules и дизайн-токены).


