# TutorLite — Легковесный календарь с уведомлениями в Telegram

> Практический проект: чистая архитектура веб-сервиса и глубокое освоение Docker (Multi-stage, Compose, сети, volumes, безопасность).

---

## 1. Суть проекта и границы (Scope)

Никаких громоздких CRM, биллинга, чатов и личных кабинетов учеников.  
Сервис решает ровно **две задачи**:
1. **Календарь преподавателя:** быстрый просмотр недели/дня, создание и перенос занятий в 2 клика.
2. **Telegram-уведомления репетитору:** утренняя сводка расписания на день и напоминание за 15–30 минут до урока.

---

## 2. Архитектура сервиса

Система состоит из 4 изолированных компонентов, каждый из которых упаковывается в свой Docker-контейнер:

```
                            [ Клиент / Браузер ]
                                     │
                                     │ HTTP :80
                                     ▼
                        ┌────────────────────────┐
                        │      nginx-proxy       │  (Reverse Proxy + статика)
                        └───────────┬────────────┘
                                    │
                                    │ внутренняя сеть Docker
                                    ▼
                        ┌────────────────────────┐
                        │      app-backend       │  (API + планировщик уведомлений)
                        └───────────┬────────────┘
                                    │
             ┌──────────────────────┴──────────────────────┐
             ▼                                             ▼
┌─────────────────────────┐                   ┌─────────────────────────┐
│        postgres         │                   │    Telegram Bot API     │
│  (БД в Docker Volume)   │                   │    (исходящие алерты)   │
└─────────────────────────┘                   └─────────────────────────┘
```

---

## 3. Схема данных (Минимальная и достаточная)

Всего две таблицы в PostgreSQL:

```sql
-- Таблица пользователей (репетиторов)
CREATE TABLE tutors (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    telegram_chat_id BIGINT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Таблица уроков
CREATE TABLE lessons (
    id SERIAL PRIMARY KEY,
    tutor_id INT REFERENCES tutors(id) ON DELETE CASCADE,
    student_name VARCHAR(100) NOT NULL,
    subject VARCHAR(100) NOT NULL,
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    meet_link TEXT,
    notes TEXT,
    notified_morning BOOLEAN DEFAULT FALSE,
    notified_before BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 4. План прокачки в Docker

Проект строится так, чтобы руками прощупать все ключевые концепции контейнеризации:

1. **Dockerfile для бэкенда (Multi-stage build):**
   - Этап `builder`: компиляция / установка зависимостей.
   - Этап `runner`: минималистичный образ (Alpine или Scratch), без лишнего мусора, итоговый вес контейнера минимален.
   - Запуск под непривилегированным пользователем (`non-root user`).
2. **Dockerfile для фронтенда и Nginx:**
   - Сборка легковесного SPA в статические файлы.
   - Раздача статики через Nginx + проксирование API-запросов на бэкенд.
3. **Docker Compose:**
   - Объединение всех сервисов в единый локальный контур (`docker compose up --build`).
   - Настройка сетей (`networks: app-net`) — изоляция базы данных от внешнего мира.
   - Настройка томов (`volumes: pgdata:/var/lib/postgresql/data`) — чтобы данные не стирались при перезапуске.
   - Healthcheck для Postgres (`pg_isready`), чтобы бэкенд ждал реальной готовности базы перед стартом.
   - Управление секретами через `.env`.
