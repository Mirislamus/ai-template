# 010 — Форматирование через `bun fmt`

- **Уровень:** Medium
- **Статус:** готово
- **Источники:** запрос пользователя; [ADR 0009](../decisions/common/0009-tooling-and-linting.md) §1; [документация `bun fmt`](https://bun.com/docs/format)
- **Цель:** форматирование во всех профилях выполняет встроенный в Bun `bun fmt` с настройками из корневого `.prettierrc`; Prettier остается только для файлов `.astro`.
- **Обоснование оценки:** только документация, больше 5 файлов, есть решение пользователя ([docs/workflow.md](../workflow.md) §1).

## Решения пользователя

- `bun fmt` — единая команда форматирования во всех профилях (Astro, Next.js, `apps/api`).

## Критерии готовности

- [x] ADR 0009 §1 описывает `bun fmt`, `.prettierrc` как единственный источник настроек и Prettier только для `.astro`.
- [x] Скрипты `format` и `format:check` в docs/setup/frontend.md §2.4 и docs/setup/backend.md §3 вызывают `bun fmt`.
- [x] `prettier` убран из установки профиля Next.js и `apps/api`; в профиле Astro остается вместе с `prettier-plugin-astro`.
- [x] Упоминания Prettier как форматтера в README, docs/tech.md, docs/quality.md, ADR 0010, ADR 0011 и навигационных документах заменены.

## Подзадачи

| # | Подзадача | Уровень | Исполнитель | Зависит от | Статус |
|---|---|---|---|---|---|
| 1 | Песочница: сравнение `bun fmt` и Prettier на эталонном `.prettierrc` | Low | основной агент | — | ☑ |
| 2 | ADR 0009, 0010, 0011 | Medium | основной агент (документация) | 1 | ☑ |
| 3 | docs/setup: common, frontend, backend | Low | основной агент (документация) | 2 | ☑ |
| 4 | README, docs/tech.md, docs/quality.md, ARCHITECTURE.md, USAGE.md, TEMPLATE_STRUCTURE.md | Low | основной агент (документация) | 2 | ☑ |

## Проверки

- Песочница вне проекта (Bun 1.4.4, Prettier 3, `prettier-plugin-astro` 1.1.2), корневой `.prettierrc`, файлы `.ts`, `.tsx`, `.astro`, `.scss`, `.json`, `.md`.
- Проверка относительных ссылок в измененных `.md`.

## Итог

`bun fmt` с glob-шаблонами из скриптов дал результат, байт в байт совпадающий с `prettier --write`, во всех шести типах файлов; `--check` возвращает 1 при неотформатированных файлах и 0 после форматирования. Файлы `.astro` `bun fmt` передает установленному Prettier с плагином. Без установленного Prettier профиль Next.js (`.prettierrc` без `plugins`) и `apps/api` в монорепо (корневой `.prettierrc` с плагином Astro) форматируются так же, как Prettier.
