```mermaid
graph TB
    %% ============================
    %% ВНЕШНИЕ КОМПОНЕНТЫ
    %% ============================
    CLIENT[Клиент<br/>NT_Planner Frontend]
    POSTGRES[(PostgreSQL 15+<br/>База данных NT_Planner)]
    
    %% ============================
    %% ОСНОВНОЙ ПОТОК ЗАПРОСОВ
    %% ============================
    FASTAPI[FastAPI<br/>HTTP endpoint<br/>uvicorn]
    ARIADNE[GraphQL сервер<br/>Ariadne]
    RESOLVER[Резолвер<br/>CRUD/кастомный]
    SQLALCHEMY[SQLAlchemy 2.x<br/>ORM + RLS/CLS фильтрация]
    
    %% Связи основного потока
    CLIENT -->|GraphQL запрос<br/>HTTP/WebSocket| FASTAPI
    FASTAPI -->|парсинг GraphQL| ARIADNE
    ARIADNE -->|вызов резолвера| RESOLVER
    RESOLVER -->|работа с данными<br/>RLS/CLS политики| SQLALCHEMY
    SQLALCHEMY -->|SQL запросы| POSTGRES
    
    %% ============================
    %% ДОПОЛНИТЕЛЬНЫЕ КОМПОНЕНТЫ
    %% ============================
    %% Redis кэш
    REDIS[(Redis<br/>Кэширование)]
    CACHE[Менеджер кэша<br/>TTL по таблицам]
    
    %% Webhook события
    WEBHOOK[Webhook обработчик<br/>Event Triggers]
    EXT_SERVICE[Внешний сервис]
    
    %% ============================
    %% МИГРАЦИИ ALEMBIC
    %% ============================
    MIGRATIONS[Alembic<br/>Миграции схемы БД]
    SCHEMA_GEN[Генератор схемы<br/>статические типы GraphQL]
    
    %% ============================
    %% ВЗАИМОДЕЙСТВИЯ
    %% ============================
    %% Кэширование
    RESOLVER -->|читает/пишет кэш| CACHE
    CACHE -->|кэширует результаты<br/>настраиваемый TTL| REDIS
    
    %% События (асинхронно)
    SQLALCHEMY -.->|триггер события<br/>INSERT/UPDATE/DELETE| WEBHOOK
    WEBHOOK -->|отправляет HTTP запрос<br/>с retry механизмом| EXT_SERVICE
    
    %% Миграции и схема
    FASTAPI -->|старт приложения<br/>apply migrations| MIGRATIONS
    MIGRATIONS -->|применяет миграции| POSTGRES
    
    SQLALCHEMY -->|модели таблиц| SCHEMA_GEN
    SCHEMA_GEN -->|статическая схема| ARIADNE
    
    %% ============================
    %% ВСПОМОГАТЕЛЬНЫЕ КОМПОНЕНТЫ
    %% ============================
    OIDC[OpenID Connect<br/>абстрактный провайдер]
    CONFIG[Конфигурация<br/>env + yaml]
    
    %% Вспомогательные связи
    OIDC -->|JWT токены<br/>аутентификация| RESOLVER
    CONFIG -->|настройки TTL| CACHE
    CONFIG -->|webhook URLs| WEBHOOK
    CONFIG -->|OIDC параметры| OIDC
    CONFIG -->|пути миграций| MIGRATIONS
    
    %% ============================
    %% СТИЛИЗАЦИЯ
    %% ============================
    classDef external fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef main fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef optional fill:#e8f5e8,stroke:#1b5e20,stroke-width:1px
    classDef migration fill:#e1f7d5,stroke:#33691e,stroke-width:2px
    classDef support fill:#fff3e0,stroke:#e65100,stroke-width:1px
    
    class CLIENT,POSTGRES,REDIS,EXT_SERVICE external
    class FASTAPI,ARIADNE,RESOLVER,SQLALCHEMY main
    class CACHE,WEBHOOK optional
    class MIGRATIONS,SCHEMA_GEN migration
    class OIDC,CONFIG support
```
# Описание архитектурной диаграммы

## Общий обзор
Диаграмма представляет архитектуру GraphQL бэкенда NT_Planner, заменяющего Hasura. Система построена по многоуровневой архитектуре с четким разделением ответственности компонентов.

## Классификация компонентов

### 1. Внешние компоненты (синий цвет)
- **Клиент (NT_Planner Frontend)** - интерфейс пользователя, инициирующий GraphQL запросы
- **PostgreSQL 15+** - основное хранилище данных с 23 таблицами согласно схеме
- **Redis** - система кэширования для повышения производительности
- **Внешний сервис** - для обработки событий через webhooks

### 2. Основной поток обработки запросов (фиолетовый цвет)
**Цепочка обработки GraphQL запроса:**
```
Клиент → FastAPI → Ariadne → Резолвер → SQLAlchemy → PostgreSQL
```

**Компоненты:**
- **FastAPI с uvicorn** - HTTP-сервер, принимающий GraphQL запросы
- **Ariadne** - GraphQL сервер, парсящий и валидирующий запросы
- **Резолвер** - обработчик запросов (автогенерируемые CRUD + кастомные)
- **SQLAlchemy 2.x** - ORM с реализацией RLS/CLS политик

### 3. Дополнительные компоненты (зеленый цвет)
- **Менеджер кэша** - управление TTL кэширования по таблицам
- **Webhook обработчик** - аналог Hasura Event Triggers с механизмом повторных попыток

### 4. Компоненты миграций и схемы (темно-зеленый цвет)
- **Alembic** - система миграций схемы БД
- **Генератор схемы** - создание статических GraphQL типов из моделей SQLAlchemy

### 5. Вспомогательные компоненты (оранжевый цвет)
- **OpenID Connect** - абстрактный провайдер аутентификации
- **Конфигурация** - централизованное управление настройками (env + yaml)

## Ключевые взаимодействия

### Основной поток данных
1. **Запрос:** Клиент → HTTP/WebSocket → FastAPI
2. **Парсинг:** FastAPI → GraphQL запрос → Ariadne
3. **Обработка:** Ariadne → вызов резолвера → Резолвер
4. **Авторизация:** OIDC → SQLAlchemy (RLS/CLS)
5. **Выполнение:** SQLAlchemy → SQL запросы → PostgreSQL

### Кэширование (опционально)
- Резолверы используют менеджер кэша для чтения/записи результатов
- Настройка TTL осуществляется через конфигурацию
- Redis используется как бэкенд кэширования

### Система событий (асинхронно)
- SQLAlchemy детектирует изменения данных (INSERT/UPDATE/DELETE)
- Триггерится Webhook обработчик
- Отправка HTTP запросов с механизмом retry во внешние сервисы

### Инициализация и поддержка
- **При старте:** FastAPI запускает Alembic для применения миграций
- **Генерация схемы:** SQLAlchemy модели → статическая GraphQL схема → Ariadne
- **Конфигурация:** управляет всеми настраиваемыми параметрами системы

## Особенности реализации

### RLS/CLS политики
- Наследуются из Hasura metadata.json
- Применяются на уровне SQLAlchemy перед формированием SQL
- Интегрируются с OpenID Connect для определения прав пользователя

### Расширяемость
- Поддержка кастомных резолверов через декораторы
- Плагинная система для webhook обработчиков
- Абстрактный OIDC провайдер для гибкой интеграции

### Производительность
- Многоуровневое кэширование с настраиваемым TTL
- Статическая генерация GraphQL схемы при старте
- Оптимизация запросов через SQLAlchemy

## Технологический стек
- **Python 3.14** (в случае нестабильности перейду на 3.11)
- **FastAPI + Ariadne** для GraphQL API
- **SQLAlchemy 2.x + Alembic** для работы с БД и миграций
- **Redis** для кэширования
- **PostgreSQL 15+** как основная БД

Система обеспечивает полную совместимость с существующим Hasura API NT_Planner при сохранении возможности горизонтального масштабирования.

```mermaid
sequenceDiagram
    participant C as Клиент
    participant F as FastAPI
    participant A as Ariadne
    participant R as Резолвер
    participant S as SQLAlchemy
    participant P as PostgreSQL
    participant CM as Менеджер кэша
    participant RD as Redis

    C->>F: POST /graphql<br/>X-Hasura-User-Id: ...<br/>X-Hasura-Role: ...
    Note over F: 1. Парсинг запроса
    F->>F: Извлечение заголовков<br/>user_id, role
    F->>F: Создание контекста запроса
    
    Note over F: 2. Валидация GraphQL
    F->>A: GraphQL запрос + контекст
    A->>A: Парсинг и валидация схемы
    A->>R: Вызов резолвера planned_tasks
    
    Note over R: 3. Проверка кэша
    R->>CM: build_cache_key(где: author_id={_eq:...})
    CM->>RD: GET planned_tasks:select:hash123
    RD-->>CM: null (кэш miss)
    CM-->>R: Нет в кэше
    
    Note over R: 4. Запрос к БД с RLS
    R->>S: query(PlannedTask).filter(...)
    S->>S: Применение RLS политик<br/>для роли "сетевой_инженер"
    S->>S: Фильтр: author_id == user_id
    S->>S: Применение CLS (скрытие столбцов)
    S->>P: SELECT * FROM planned_tasks<br/>WHERE author_id = '...'
    P-->>S: Данные (3 записи)
    
    Note over R: 5. Кэширование результатов
    S-->>R: Результаты
    R->>CM: set(planned_tasks:select:hash123, данные, TTL=300)
    CM->>RD: SETEX planned_tasks:select:hash123 300 ...
    
    Note over R: 6. Обработка вложенных запросов
    R->>S: Запрос planned_task_works для каждой задачи
    S->>P: SELECT * FROM planned_task_works WHERE planned_task_id IN (...)
    P-->>S: Результаты
    S-->>R: Вложенные данные
    
    R-->>A: Форматированные данные
    A-->>F: GraphQL ответ
    F-->>C: HTTP 200 JSON
```
