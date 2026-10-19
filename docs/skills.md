# Регламент использования AI-навыков (docs/skills.md)

Документ определяет одобренные AI-навыки (Skills): основные навыки жизненного цикла с условиями запуска (Core Pipeline), специализированные навыки по требованию (On-Demand), опциональные плагины Claude Code, правила совместимости с документацией проекта и протокол отчетности. Используются только навыки из этого списка.

---

## 1. Протокол использования навыков (Lifecycle & Reporting)

### Этап 1: Декларация на старте задачи (Before Work)
- В первом сообщении задачи агент **явно объявляет**, какие навыки применит и зачем:
  - *Пример:* «Будут использованы скилы: `$figma` (извлечение параметров макета), `$emil-design-eng` (полировка состояний кнопки), `$code-review` (проверка перед сдачей)».
- Если задача тривиальна (опечатка, правка документации) и навыки не нужны, агент пишет: *«Скилы не требуются»*.

### Этап 2: Фиксация в итоговом отчете (After Work)
- В итоговом отчете по [AGENTS.md](../AGENTS.md) агент добавляет пункт **«Использованные скилы»**: список фактически примененных навыков и конкретный результат каждого (например: *«`$emil-design-eng`: добавлен `transform: scale(0.98)` в состоянии `:active` кнопки»*).

---

## 2. Приоритет стандартов проекта (Strict Precedence)

[AGENTS.md](../AGENTS.md), [docs/design.md](design.md) и реестр [docs/decisions/](decisions/) имеют **абсолютный приоритет** над рекомендациями любого навыка:
- Tailwind CSS — запрещен (только SCSS Modules, [ADR 0003](decisions/frontend/0003-scss-standards.md)).
- Radix UI и Headless UI — запрещены (только нативные `<dialog>`, `<details>`, [ADR 0002](decisions/frontend/0002-html-standards.md) и [ADR 0010](decisions/common/0010-third-party-libraries-policy.md)).
- `rem`/`em` для отступов и шрифтов — запрещены (`px` и токены, [docs/design.md](design.md)).
- SWR — не используется: серверное состояние ведет TanStack Query ([ADR 0008](decisions/frontend/0008-forms-and-api.md)).
- Файл `DESIGN.md` не создается: единственный источник дизайн-правил — [docs/design.md](design.md).

### Дизайн-навыки: границы и приоритет

Правило касается `$impeccable`, `$emil-design-eng` и правил плагина `frontend-design` (сам плагин не используется, §5). Их стандартные правила вкуса, сетки и анимации в проекте не действуют: дизайн-правила задают [docs/design.md](design.md), ADR и [docs/quality.md](quality.md), а анти-шаблоны и порядок выбора стилистики — [docs/design.md](design.md) §9.

**Допустимое применение:**
- `$impeccable` — только проверка и доработка уже сверстанного интерфейса в рамках задачи: команды `audit`, `critique`, `polish`, `harden`, `clarify`, `layout`, `typeset`. Другие команды и сценарий новой работы (выбор направления, «кубики» `concept-seed`, страницы выбора, макеты-изображения) не запускаются.
- `$emil-design-eng` — ревью и доработка состояний элемента ([docs/design.md](design.md) §7) и переходов; значения длительностей, кривых и смещений берутся только из токенов.

**Не создаются:** `PRODUCT.md` и `DESIGN.md` от имени навыка, файлы `.impeccable/` (конфиг, макеты, скриншоты ревью, описания поверхностей), хуки детектора, HTML-комментарий «контракт направления» в разметке ([AGENTS.md](../AGENTS.md): лишние комментарии не оставляются), собственные субагенты навыка: субагенты — только из `.claude/agents/` по [docs/workflow.md](workflow.md).

**Правила навыков, которые заменяются правилами проекта:**

| Правило навыка | Навык | Правило проекта |
|---|---|---|
| Смелое направление и стилистика на выбор агента | `$impeccable`, `frontend-design` | Стилистику задают макет, брендбук или план по [docs/design.md](design.md) §9 после подтверждения пользователя |
| Размеры шрифта в `rem`, межбуквенный интервал в `em` | `$impeccable` | Только `px` и токены, `rem` и `em` запрещены Stylelint ([ADR 0003](decisions/frontend/0003-scss-standards.md) §2) |
| Шкала отступов с шагом 4 | `$impeccable` | 8pt-сетка `--space-*`, полушаг 4px — для микро-отступов ([docs/design.md](design.md) §1) |
| Палитры в OKLCH, светлая или темная тема на выбор | `$impeccable`, `frontend-design` | HEX или RGB в семантических слотах; светлая тема в `:root`, темная — в `[data-theme='dark']` ([docs/design.md](design.md) §3, [ADR 0003](decisions/frontend/0003-scss-standards.md) §4) |
| Отказ от Inter и новый шрифт в каждом проекте | `$impeccable`, `frontend-design` | Шрифты — по плану [docs/design.md](design.md) §9 с самохостингом ([ADR 0007](decisions/frontend/0007-assets-and-media.md) §3); Inter в эталонных токенах — значение до утверждения плана |
| Шкала шрифтов и длина строки по собственным правилам | `$impeccable`, `frontend-design` | Токены `--font-size-*` и строка абзаца 65–75 символов ([docs/design.md](design.md) §4) |
| Радиус карточек 12–16px | `$impeccable` | Шкала `--radius-*` по уровню элемента ([docs/design.md](design.md) §5) |
| Зоны нажатия не меньше 44×44px | `$impeccable` | Естественные внутренние отступы и зазор `--space-2` ([docs/design.md](design.md) §6) |
| Анимация `blur`, `backdrop-filter`, `clip-path`, `mask` | `$impeccable`, `$emil-design-eng` | Безопасный список свойств и исключение аккордеона ([ADR 0003](decisions/frontend/0003-scss-standards.md) §15, [ADR 0017](decisions/frontend/0017-motion-and-animations.md) §1) |
| Собственные кривые и длительности (`cubic-bezier(0.23, 1, 0.32, 1)` и др.) | `$emil-design-eng`, `$impeccable` | Токены `--motion-*` и `--ease-out`; новое значение — новый токен с согласия пользователя |
| `scale(0.97)` в состоянии `:active` | `$emil-design-eng` | `scale(0.98)` или `translateY(1px)` ([docs/design.md](design.md) §7) |
| Пружины и `useSpring` из Motion (Framer Motion) | `$emil-design-eng` | Motion запрещен ([ADR 0010](decisions/common/0010-third-party-libraries-policy.md) §2): CSS-переходы и WAAPI |
| Эффектная загрузка страницы, WebGL и canvas ради выразительности | `$impeccable`, `frontend-design` | Движение объясняет изменение состояния ([ADR 0017](decisions/frontend/0017-motion-and-animations.md)); бюджет JS — [ADR 0012](decisions/common/0012-build-and-package-manager.md) §2 |
| Оценки навыка (баллы `audit`, `critique`) как критерий готовности | `$impeccable` | Приемка — Definition of Done ([ADR 0014](decisions/common/0014-testing-and-qa.md) §4) и [docs/quality.md](quality.md) |

**Работа с замечаниями навыка:**
- Замечание исправляется, только если его подтверждает правило проекта; в отчете указывается ссылка на это правило.
- Замечание, противоречащее правилу проекта, отклоняется со ссылкой на правило.
- Замечание без опоры на правило проекта молча не применяется: оно попадает в отчет как рекомендация навыка, решение принимает пользователь.

---

## 3. Основные навыки жизненного цикла (Core Pipeline)

| Навык | Когда запускать | Что делает в проекте |
|---|---|---|
| `$grilling` | Старт проекта, крупная фича, архитектурная развилка | Допрос по одному вопросу с рекомендацией; результат фиксируется в PRODUCT.md ([PROMPT.md](../PROMPT.md)) |
| `$figma` | Только при наличии макета Figma | Токены, ассеты и параметры макета для переноса в SCSS Modules |
| `$impeccable` | После верстки секции или страницы: аудит и полировка | UX-аудит, иерархия, состояния интерфейса, устранение визуального шума — только командами и по правилам §2 |
| `$emil-design-eng` | Интерактивные элементы и микровзаимодействия | Hover/active отклики, оптическое выравнивание, переходы — значения только из токенов (§2) |
| `$vercel-react-best-practices` | Только при написании или ревью React-кода | Аудит производительности: ререндеры, хуки, утечки; правила только для Next.js не применяются в Astro |
| `$agent-browser` | После UI-изменений и при финальной приемке | Проверка в реальном браузере: консоль без ошибок, отсутствие горизонтального скролла на 360px, формы |
| `$code-review` | Перед сдачей задачи | Ревью по двум осям: стандарты репозитория и соответствие спецификации |

### Совместимость навыков с файлами проекта

Навыки ищут файлы в собственной структуре; в этом репозитории они направляются так:

| Навык | Что он ожидает | Что использовать в проекте |
|---|---|---|
| `$impeccable` | `PRODUCT.md`, `DESIGN.md`, `.impeccable/`; команды `init`, `document`, `doctor` | Только команды из §2; PRODUCT.md заполняется только через допрос по [PROMPT.md](../PROMPT.md) и не перезаписывается навыком; `DESIGN.md` не создается, источник — [docs/design.md](design.md); каталог `.impeccable/` игнорируется Git |
| `$emil-design-eng` | Motion (Framer Motion), собственные кривые и длительности | CSS-переходы и WAAPI, токены `--motion-*` и `--ease-out` (§2) |
| `$code-review` | Issue tracker (`docs/agents/issue-tracker.md`), `CODING_STANDARDS.md` | Спецификация: `docs/pages/*.md` + PRODUCT.md + [docs/tasks.md](tasks.md); стандарты: AGENTS.md + [docs/decisions/](decisions/) + [docs/design.md](design.md) |
| `$tdd`, `$diagnosing-bugs` | `CONTEXT.md`, `docs/adr/` | `CONTEXT.md` заменяют [ARCHITECTURE.md](../ARCHITECTURE.md) и [PRODUCT.md](../PRODUCT.md); ADR — [docs/decisions/](decisions/) |
| `$request-refactor-plan` | Создает GitHub issue | План записывается в [docs/tasks.md](tasks.md); внешние записи (issue) только с разрешения пользователя по AGENTS.md |
| `$vercel-react-best-practices` | SWR, Next.js-паттерны | SWR заменяется TanStack Query; серверные паттерны Next.js не применяются в профиле Astro |
| `$agent-browser` | Глобальная установка CLI | Устанавливается один раз вне проекта; в зависимости репозитория не добавляется |

---

## 4. Специализированные навыки по требованию (On-Demand)

Подключаются, только когда возникает соответствующая задача:

- **`$imagegen`** — растровые иллюстрации, текстуры и заглушки при отсутствии готового фотобанка (с последующей оптимизацией в AVIF/WebP по [ADR 0007](decisions/frontend/0007-assets-and-media.md)).
- **`$research`** — исследование внешней документации и спецификаций незнакомых API (эквайринг, CRM, Telegram Bot API).
- **`$tdd`** — разработка через тестирование для сложной бизнес-логики (схемы Zod, калькуляторы цен, финансовые расчеты) на Vitest.
- **`$diagnosing-bugs`** — структурированная локализация трудноуловимых багов, утечек памяти и просадок FPS.
- **`$resolving-merge-conflicts`** — пошаговое разрешение конфликтов слияния по [ADR 0011](decisions/common/0011-git-workflow-and-commits.md).
- **`$request-refactor-plan`** — безопасный пошаговый план рефакторинга крупного модуля на серию атомарных коммитов.
- **`$codebase-design`** — проектирование чистых интерфейсов нестандартных модулей по FSD-Lite.

---

## 5. Плагины Claude Code (опционально)

Работают только в Claude Code, ставятся один раз в окружение пользователя, а не в репозиторий, и не заменяют навыки из §3–4.

| Плагин | Установка | Что делает в проекте |
|---|---|---|
| `security-guidance` | `/plugin install security-guidance@claude-plugins-official` | Не блокирующие предупреждения о небезопасном коде: regex-шаблоны при `Edit`/`Write` (например, сырой `innerHTML`, захардкоженные секреты), LLM-ревью diff в конце хода и ревью при `git commit`. Поддерживает правила [ADR 0008](decisions/frontend/0008-forms-and-api.md) о выводе сырого HTML и запрет секретов из AGENTS.md, но не заменяет `$code-review` |

Требования и ограничения `security-guidance` ([README плагина](https://github.com/anthropics/claude-code/tree/main/plugins/security-guidance)):
- Claude Code CLI ≥ 2.1.144 и Python 3.8+ в `PATH`.
- Ревью отправляет на API модели пути и фрагменты измененных файлов и расходует токены.
- Слои отключаются переменными окружения: `ENABLE_STOP_REVIEW=0`, `ENABLE_COMMIT_REVIEW=0`, `ENABLE_PATTERN_RULES=0`, весь плагин — `SECURITY_GUIDANCE_DISABLE=1`.

Другие плагины не используются. `superpowers`, `frontend-design` и `playwright` (MCP) дублируют навыки §3 или противоречат правилам проекта; правила `frontend-design` действуют только в редакции [docs/design.md](design.md) §9 (§2).
