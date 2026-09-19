# Структура шаблона проекта

Шаблон предназначен для разработки лендингов, многостраничных сайтов и веб-сервисов с помощью AI-агентов и разработчиков. Документация разделена на постоянные правила, описание продукта, устройство кодовой базы и текущее состояние работ.

## Стартовый комплект

```text
PROMPT.md               # Промпты запуска и дерево обязательного допроса по продукту
TEMPLATE_STRUCTURE.md   # Эталонная структура файлов репозитория
USAGE.md                # Пошаговое руководство по использованию шаблона

AGENTS.md               # Регламент взаимодействия и поведения AI-агентов
PRODUCT.md              # Паспорт и бизнес-требования к продукту
ARCHITECTURE.md         # Архитектурная схема FSD-Lite и реестр ADR
README.md               # Главная инструкция по командам и окружению
STATE.md                # Текущий статус готовности проекта
.env.example            # Шаблон переменных окружения

docs/
├── tech.md             # Технологический стек и одобренные библиотеки
├── setup.md            # Пошаговое развертывание проекта через Bun с конфигами
├── skills.md           # Регламент AI-навыков (Core Pipeline vs On-Demand)
├── design.md           # Инженерная дизайн-система (8pt Grid, токены, clamp)
├── quality.md          # Чеклист качества, Lighthouse (90+/100), кроссбраузерность
├── seo.md              # Оперативные SEO-стандарты страниц, OpenGraph
├── content.md          # Экранная типографика, инфостиль и правовой дисклеймер
├── tasks.md            # Журнал задач и бэклог реализации
├── pages/              # Спецификации страниц
│   ├── page-template.md# Эталонный шаблон паспорта страницы
│   ├── 404.md          # Спецификация страницы ошибки 404
│   └── privacy-policy.md # Спецификация страницы политики конфиденциальности
└── decisions/          # Реестр из 18 архитектурных решений (ADR 0001–0018)
    ├── 0001-project-architecture.md
    ├── 0002-html-standards.md
    ├── 0003-scss-standards.md
    ├── 0004-ts-js-standards.md
    ├── 0005-react-standards.md
    ├── 0006-state-management.md
    ├── 0007-assets-and-media.md
    ├── 0008-forms-and-api.md
    ├── 0009-tooling-and-linting.md
    ├── 0010-third-party-libraries-policy.md
    ├── 0011-git-workflow-and-commits.md
    ├── 0012-build-and-package-manager.md
    ├── 0013-seo-standards.md
    ├── 0014-testing-and-qa.md
    ├── 0015-analytics-and-tracking.md
    ├── 0016-i18n-standards.md
    ├── 0017-motion-and-animations.md
    └── 0018-deployment-and-caching.md
```

`PROMPT.md`, `TEMPLATE_STRUCTURE.md` и `USAGE.md` — служебные файлы стартового комплекта. Остальные файлы относятся к конкретному разрабатываемому проекту.

