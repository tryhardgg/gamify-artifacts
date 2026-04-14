# DB Schema для Gamify Artifacts

Этот документ описывает черновую схему БД под текущую доменную модель модульного монолита: Player, Economy, Inventory, Shop, Artifacts, Habits, SingularityIntegration.  
Цель: дать основу для создания таблиц в Supabase и дальнейшей эволюции схемы.

> **Это черновая схема.** Типы, enum-ы, индексы и RLS-политики будут уточняться по мере реализации Epic 1.  
> Все таблицы логически принадлежат конкретным доменным модулям. Модули не обращаются к чужим таблицам напрямую — только через публичные операции модуля-владельца.

---

## 0. Общие принципы

- **СУБД:** PostgreSQL (Supabase).
- **Один владелец ресурса:** баланс → Economy, предметы → Inventory, артефакты → Artifacts, привычки → Habits, внешние данные → SingularityIntegration.
- **Внешние ключи** — для явных связей между доменными сущностями.
- **`player_id`** — ключ для будущих RLS-политик (Row Level Security): каждый игрок видит только свои данные.
- **Денормализация** допустима для удобства запросов (например, `player_id` в `transactions`), но источник истины — FK-связь.
- **Enum-типы** в черновой схеме описаны текстом (`text`), но при реализации рекомендуется использовать PostgreSQL `ENUM` или `CHECK`-ограничения.

---

## 1. Player

### 1.1. Таблица `players`

Основная сущность игрока. Пока один игрок — автор, но схема многоразовая.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор игрока |
| `auth_user_id` | uuid, FK → auth.users.id | Связь с Supabase Auth (внешний пользователь) |
| `username` | text, unique, nullable | Отображаемое имя игрока |
| `avatar_url` | text, nullable | URL аватара, хранящегося в Supabase Storage |
| `settings` | jsonb, nullable | Пользовательские настройки: активные категории артефактов, фильтры задач Singularity, предпочтения магазина и т.п. |
| `created_at` | timestamptz, default now() | Дата и время создания записи игрока |
| `updated_at` | timestamptz, default now() | Дата и время последнего обновления записи игрока |

> **На будущее:** `settings` может вырасти в отдельную таблицу `player_settings`, если настроек станет много.

---

## 2. Economy

### 2.1. Таблица `wallets`

Один кошелёк на игрока.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор кошелька |
| `player_id` | uuid, FK → players.id, unique | Игрок-владелец кошелька (один кошелёк на игрока) |
| `balance` | bigint, default 0 | Текущий баланс в минимальных единицах. Инвариант: `balance >= 0`. |
| `currency` | text, default 'GAF' | Код валюты (Gamify Artifacts) |
| `created_at` | timestamptz, default now() | Дата и время создания кошелька |
| `updated_at` | timestamptz, default now() | Дата и время последнего изменения баланса |

### 2.2. Таблица `transactions`

История операций по кошельку — истинный источник данных по движению средств.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор транзакции |
| `wallet_id` | uuid, FK → wallets.id | Кошелёк, к которому относится транзакция |
| `player_id` | uuid, FK → players.id | Денормализовано для удобства запросов (источник истины — `wallets.player_id`) |
| `type` | text | Тип транзакции. В текущей схеме хранится как `text`, но на этапе миграций MVP необходимо добавить ограничение через `CHECK` или `ENUM`. |
| `amount` | bigint | Сумма со знаком: `+` для начислений, `−` для списаний |
| `source_module` | text | Модуль-инициатор: `artifacts`, `habits`, `shop`, `manual` (ручная корректировка), `system` (стартовый баланс, системные исправления) |
| `source_id` | uuid, nullable | Идентификатор сущности-источника (artifact_id, habit_log_id, purchase_id и т.п.) |
| `description` | text, nullable | Человекочитаемое описание. Примеры: `source_module = 'system'` + `description = 'Initial welcome bonus'`; `source_module = 'manual'` + `description = 'Ручная корректировка: +500 GAF'` |
| `created_at` | timestamptz, default now() | Дата и время совершения транзакции |

**Ограничения на `type` (MVP):**
Для поля `type` необходимо добавить ограничение допустимых значений:
- `reward` — награда за задачи/артефакты;
- `purchase` — покупка товара в магазине;
- `habit_reward` — награда за выполнение привычки;
- `artifact_reward` — награда, привязанная к артефакту;
- `bonus` — стартовые и разовые системные бонусы;
- `correction` — ручная корректировка баланса;
- `refund` — возврат средств после отмены покупки.

Миграции должны обеспечить `CHECK (type IN (...))` или PostgreSQL `ENUM` с этим списком.

**Инвариант:** `wallets.balance` = SUM(transactions.amount) для данного wallet_id. Модуль Economy гарантирует это при каждой операции.

---

## 3. Inventory

### 3.1. Таблица `items`

Справочник типов предметов (что вообще может существовать в игре).

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор типа предмета |
| `code` | text, unique | Короткий машинно-читаемый код (`MATE_10G`, `COFFEE`, `GEM_BLUE`) |
| `name` | text | Человекочитаемое название предмета |
| `description` | text, nullable | Текстовое описание предмета, его назначения и свойств |
| `rarity` | text, nullable | Редкость предмета: `common`, `uncommon`, `rare`, `epic`, `legendary`, `mythic` |
| `metadata` | jsonb, nullable | Дополнительные свойства: категория, иконка, теги, физический/виртуальный и т.п. |
| `created_at` | timestamptz, default now() | Дата и время добавления типа предмета в справочник |
| `updated_at` | timestamptz, default now() | Дата и время последнего обновления записи типа предмета |

### 3.2. Таблица `inventory_items`

Конкретные количества предметов у игрока.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор записи инвентаря |
| `player_id` | uuid, FK → players.id | Игрок-владелец предметов |
| `item_id` | uuid, FK → items.id | Тип предмета из справочника |
| `quantity` | integer, default 0, check >= 0 | Количество единиц данного типа предмета у игрока |
| `created_at` | timestamptz, default now() | Дата и время первой записи предмета в инвентарь |
| `updated_at` | timestamptz, default now() | Дата и время последнего изменения количества |

**Уникальный индекс:** `(player_id, item_id)` — одна запись на комбинацию «игрок + тип предмета».

> **На будущее:** если понадобится история движения предметов (когда и откуда получен/списан), можно добавить таблицу `inventory_transactions` по аналогии с `transactions`.

---

## 4. Shop

### 4.1. Таблица `products`

Каталог доступных для покупки товаров.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор товара в каталоге |
| `code` | text, unique | Машинно-читаемый код продукта (может совпадать с `items.code` или быть отдельным) |
| `name` | text | Человекочитаемое название товара |
| `description` | text, nullable | Описание товара, отображаемое в магазине |
| `price` | bigint | Цена в минимальных единицах валюты |
| `currency` | text, default 'GAF' | Код валюты цены |
| `item_id` | uuid, FK → items.id, nullable | Тип предмета, который будет выдан при покупке (null = товар без предмета, например, услуга) |
| `item_quantity` | integer, default 1 | Сколько единиц предмета выдать при одной покупке |
| `stock_quantity` | integer, default 0 | Текущий остаток на складе (0 = нет в наличии, −1 = безлимит) |
| `stock_limit` | integer, nullable | Максимальный лимит склада (null = безлимит) |
| `category` | text, nullable | Категория товара для фильтрации в магазине |
| `is_active` | boolean, default true | Видим ли товар в каталоге (false = скрыт или снят с продажи) |
| `created_at` | timestamptz, default now() | Дата и время добавления товара в каталог |
| `updated_at` | timestamptz, default now() | Дата и время последнего обновления товара |

### 4.2. Таблица `purchases`

Факт покупки (успешной или отклонённой).

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор записи о покупке |
| `player_id` | uuid, FK → players.id | Игрок, совершивший покупку |
| `product_id` | uuid, FK → products.id | Товар, который был куплен |
| `transaction_id` | uuid, FK → transactions.id, nullable | Ссылка на транзакцию списания валюты (null если покупка отклонена) |
| `granted_item_id` | uuid, FK → items.id, nullable | Тип выданного предмета (денормализовано из `products.item_id`) |
| `granted_quantity` | integer, default 0 | Количество выданного предмета |
| `status` | text | Статус покупки: `pending` (ожидает), `completed` (успешна), `declined` (отклонена), `refunded` (возврат) |
| `decline_reason` | text, nullable | Причина отклонения (например, «недостаточно средств», «нет в наличии») |
| `created_at` | timestamptz, default now() | Дата и время совершения покупки |

---

## 5. Artifacts

### 5.1. Таблица `artifacts`

Основная сущность артефакта.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор артефакта |
| `player_id` | uuid, FK → players.id | Игрок-владелец артефакта |
| `title` | text | Краткое человекочитаемое имя артефакта |
| `description` | text, nullable | Развёрнутое описание артефакта, его цели и контекста |
| `category` | text | Категория артефакта: `work`, `health`, `sport`, `study`, `household`, а также специальная категория `VISION` для артефакта-жизненного видения |
| `metadata` | jsonb, not null, default '{}'::jsonb | Дополнительные данные артефакта; структура зависит от категории. Для `VISION` здесь хранится текст видения, 4 сферы с картинками и настройки screensaver (см. 5.1.1) |
| `lifecycle_status` | text | Статус жизненного цикла: `active` (в работе), `completed` (завершён), `overdue` (просрочен), `cancelled` (отменён), `archived` (в архиве) |
| `current_stage` | text | Текущая стадия из 4-стадийного цикла: `planning`, `gathering`, `creating`, `review` |
| `progress_total` | integer, default 0, check 0–100 | Общий процент выполнения артефакта (0–100%), отображается на Dashboard |
| `progress_today` | integer, default 0, check >= 0 | Процент прогресса, сделанный за текущий день (для визуализации ежедневной динамики) |
| `deadline` | timestamptz, nullable | Дедлайн завершения артефакта; по истечении статус меняется на `overdue` |
| `quality_score` | integer, nullable | Итоговое качество, рассчитанное на стадии review. Копируется из `artifact_evaluations.total_quality` при завершении оценки (денормализованный кэш для быстрого доступа). |
| `rarity` | integer, nullable | Редкость артефакта (0–5), рассчитанная на стадии review. Копируется из `artifact_evaluations.rarity` при завершении оценки (денормализованный кэш). При переоценке обновляется из последней записи `artifact_evaluations` (по `completed_at`). |
| `reward_calculated` | boolean, default false | Флаг: награда за артефакт уже начислена (true) или ещё нет (false) |
| `template_id` | uuid, nullable, FK → artifact_templates.id | Шаблон, по которому был автоматически создан артефакт |
| `created_at` | timestamptz, default now() | Дата и время создания артефакта |
| `updated_at` | timestamptz, default now() | Дата и время последнего обновления артефакта |

### 5.1.1. Специфика категории VISION

Артефакт с `category = 'VISION'` используется для визуализации идеального дня/жизни игрока. В отличие от обычных артефактов, VISION не проходит стандартный 4-стадийный цикл с начислением наград — его цель — хранить и отображать жизненное видение игрока.

**Основное содержимое VISION:**

- текстовое описание идеального дня/жизни;
- 4 сферы жизни, каждая с набором картинок (3–5 на сферу), вызывающих эмоциональный отклик;
- настройки screensaver (включён/выключен, таймаут бездействия).

**Структура `metadata` для VISION** (рекомендованный формат, может эволюционировать; ключи `vision_text`, `spheres` и `screensaver` считаются базовыми для клиента):

```json
{
  "vision_text": "Текстовое описание идеального дня/жизни игрока",
  "spheres": {
    "state": {
      "title": "Тело и энергия",
      "description": "Здоровье, питание, сон, восстановление, движение",
      "images": [
        { "id": "uuid-1", "path": "vision/<player_id>/state/uuid-1.jpg", "is_primary": true },
        { "id": "uuid-2", "path": "vision/<player_id>/state/uuid-2.jpg", "is_primary": false }
      ]
    },
    "finances": {
      "title": "Деньги и свобода",
      "description": "Финансы, доход, капитал, финансовая свобода",
      "images": []
    },
    "hobbies": {
      "title": "Хобби",
      "description": "Игры, творчество, сайд-проекты",
      "images": []
    },
    "relationships": {
      "title": "Любовь и близость",
      "description": "Отношения с партнёром, прогулки, свидания, совместное время",
      "images": []
    }
  },
  "screensaver": {
    "enabled": true,
    "idle_timeout_sec": 180
  }
}
```

**Пояснения к структуре:**

- Ключи `state`, `finances`, `hobbies`, `relationships` — внутренние коды сфер. Отображаемые названия берутся из поля `title` каждой сферы.
- Поле `images` — массив объектов, где каждый объект содержит:
  - `id` (uuid) — идентификатор изображения;
  - `path` (text) — путь к файлу в Supabase Storage (например, `vision/<player_id>/state/uuid-1.jpg`);
  - `is_primary` (boolean) — флаг основной картинки сферы.
  - В будущем может появиться дополнительный атрибут `type: "image" | "animation"` для поддержки анимаций.
- Поле `screensaver` содержит:
  - `enabled` (boolean) — включён ли режим screensaver;
  - `idle_timeout_sec` (integer) — таймаут бездействия в секундах, после которого запускается screensaver (по умолчанию 180 = 3 минуты).

> **Примечание:** VISION — это специализированный тип артефакта внутри существующего модуля Artifacts. Новый модуль для VISION не вводится. Всё реализуется через `category = 'VISION'` + `metadata` jsonb.

**Фиксированные значения полей для VISION:**

| Поле | Значение для VISION | Пояснение |
|------|---------------------|-----------|
| `lifecycle_status` | `'active'` или `'archived'` | VISION не имеет статусов `completed`, `overdue`, `cancelled` |
| `current_stage` | `null` | VISION не проходит 4-стадийный цикл |
| `progress_total` | `0` | Прогресс не рассчитывается |
| `progress_today` | `0` | Ежедневный прогресс не рассчитывается |
| `deadline` | `null` | VISION — долгоживущий артефакт без дедлайна |
| `reward_calculated` | `false` | Награды не начисляются |
| `quality_score` | `null` | Оценка не проводится |
| `rarity` | `null` | Редкость не рассчитывается |

### 5.2. Таблица `artifact_stages`

Детализация по стадиям: когда начата, когда завершена, данные стадии.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор записи стадии артефакта |
| `artifact_id` | uuid, FK → artifacts.id | Артефакт, к которому относится стадия |
| `stage` | text | Название стадии: `planning`, `gathering`, `creating`, `review` |
| `started_at` | timestamptz, nullable | Дата и время начала стадии (null если ещё не начата) |
| `completed_at` | timestamptz, nullable | Дата и время завершения стадии (null если ещё не завершена) |
| `data` | jsonb, nullable | Структурированные данные стадии: список шагов, ресурсы, заметки, чек-листы и т.п. |

**Уникальный индекс:** `(artifact_id, stage)` — одна запись на стадию артефакта.

### 5.3. Таблица `artifact_task_links`

Связь артефакта с задачами Singularity, которые внесли вклад в прогресс.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор связи артефакта с задачей |
| `artifact_id` | uuid, FK → artifacts.id | Артефакт, в прогресс которого внесла вклад задача |
| `singularity_task_id` | uuid, FK → singularity_tasks.id | Задача из Singularity, связанная с артефактом |
| `link_type` | text, nullable | Тип связи: `contributes` (внесла вклад), `blocks` (блокирует), `related` (относится) |
| `created_at` | timestamptz, default now() | Дата и время создания связи |

**Уникальный индекс:** `(artifact_id, singularity_task_id)`.

### 5.4. Таблица `artifact_evaluations`

Оценка артефакта по критериям на стадии review.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор оценки артефакта |
| `artifact_id` | uuid, FK → artifacts.id | Артефакт, который был оценён |
| `criteria` | jsonb | Оценки по универсальным и специфическим критериям (набор зависит от категории артефакта) |
| `total_quality` | integer | Итоговое качество, вычисленное по критериям |
| `rarity` | integer | Вычисленная редкость артефакта (0–5) на основе качества |
| `completed_at` | timestamptz, default now() | Дата и время проведения оценки |

### 5.5. Таблица `artifact_rewards`

Награда за завершение артефакта — что именно было начислено.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор записи награды за артефакт |
| `artifact_id` | uuid, FK → artifacts.id | Артефакт, за завершение которого выдана награда |
| `currency_amount` | bigint, default 0 | Сумма начисленной валюты за артефакт |
| `currency_transaction_id` | uuid, FK → transactions.id, nullable | Ссылка на транзакцию начисления валюты в Economy |
| `granted_item_id` | uuid, FK → items.id, nullable | Тип предмета, выданного как часть награды |
| `granted_quantity` | integer, default 0 | Количество выданного предмета |
| `created_at` | timestamptz, default now() | Дата и время начисления награды |

### 5.6. Таблица `artifact_templates` (опционально для MVP)

Шаблоны артефактов: правила генерации, какие задачи Singularity порождают артефакт, какие награды он даёт.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор шаблона артефакта |
| `name` | text | Человекочитаемое название шаблона |
| `category` | text | Категория артефактов, к которой применяется шаблон |
| `rule` | jsonb | Правило маппинга задач Singularity на этот шаблон. Минимальный контракт: `{"tag_any": [...], "tag_all": [...], "exclude_tags": [...]}`. Логика: (1) exclude_tags блокирует шаблон; (2) tag_all — все теги должны присутствовать; (3) tag_any — хотя бы один тег должен присутствовать. Опциональное поле `max_per_task` (integer, default 1) — макс. артефактов на одну задачу. Структура поля валидируется на уровне приложения; БД хранит значение как `jsonb` без встроенных ограничений на структуру. |
| `base_reward_currency` | bigint, default 0 | Базовая награда в валюте при завершении артефакта, созданного по шаблону |
| `base_reward_item_id` | uuid, FK → items.id, nullable | Базовый предмет в награде за артефакт по шаблону |
| `base_reward_quantity` | integer, default 0 | Базовое количество предмета в награде |
| `is_active` | boolean, default true | Активен ли шаблон для автогенерации артефактов |
| `created_at` | timestamptz, default now() | Дата и время создания шаблона |

> **На будущее:** шаблоны могут стать основой для автоматической генерации артефактов из задач Singularity.

**Пример значения `rule`:**

```json
{
  "tag_any": ["okr:health"],
  "tag_all": [],
  "exclude_tags": ["no-artifact"],
  "max_per_task": 1
}
```

Где:
- `tag_any` — массив строк: задача должна содержать хотя бы один из указанных тегов.
- `tag_all` — массив строк: задача должна содержать все указанные теги.
- `exclude_tags` — массив строк: если задача содержит хотя бы один из этих тегов, шаблон не применяется.
- `max_per_task` — число: максимальное количество артефактов, которое данный шаблон может создать для одной задачи (по умолчанию 1).

---

## 6. Habits

### 6.1. Таблица `habits`

Карточка привычки.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор привычки |
| `player_id` | uuid, FK → players.id | Игрок-владелец привычки |
| `name` | text | Человекочитаемое название привычки (например, «Зарядка», «Медитация») |
| `description` | text, nullable | Развёрнутое описание привычки, её цели и контекста |
| `type` | text | Тип привычки: `discrete` (точечная, выполняется в окне времени) или `abstain` (воздержание на весь день) |
| `target_time_from` | time, nullable | Начало окна выполнения для `discrete` привычек (например, 8:00) |
| `target_time_to` | time, nullable | Конец окна выполнения для `discrete` привычек (например, 9:00) |
| `target_duration_minutes` | integer, nullable | Целевая длительность выполнения на текущем уровне (5, 10, 15 мин и т.п.) |
| `level` | integer, default 1 | Текущий уровень привычки, растущий с накоплением XP |
| `xp` | numeric, default 0 | Накопленный опыт (1.0 = полный день, 0.5 = минимальное выполнение) |
| `xp_to_next_level` | numeric, nullable | Порог XP для перехода на следующий уровень (может считаться динамически) |
| `streak_current` | integer, default 0 | Текущий стрик — дней подряд с хотя бы минимальным выполнением |
| `streak_best` | integer, default 0 | Лучший стрик за всё время |
| `short_term_buffs` | jsonb, nullable | Краткосрочные бафы на текущий день: бодрость, настроение, снижение напряжения и т.п. |
| `long_term_buffs` | jsonb, nullable | Долгосрочные кумулятивные бафы: здоровье, осанка, координация и т.п. |
| `reward_currency` | bigint, default 0 | Моментальная награда в валюте за полное выполнение за день |
| `reward_item_id` | uuid, FK → items.id, nullable | Тип предмета, выдаваемого за полное выполнение за день |
| `reward_quantity` | integer, default 0 | Количество предмета за полное выполнение за день |
| `is_active` | boolean, default true | Активна ли привычка для отслеживания |
| `created_at` | timestamptz, default now() | Дата и время создания привычки |
| `updated_at` | timestamptz, default now() | Дата и время последнего обновления привычки |

### 6.2. Таблица `habit_logs`

Факт выполнения привычки за конкретный день.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор записи выполнения привычки |
| `habit_id` | uuid, FK → habits.id | Привычка, к которой относится лог |
| `player_id` | uuid, FK → players.id | Денормализовано для удобства запросов (источник истины — `habits.player_id`) |
| `date` | date | Календарный день, к которому относится выполнение |
| `status` | text | Статус выполнения: `full` (полное), `partial` (минимальное), `missed` (пропуск) |
| `xp_earned` | numeric, default 0 | Количество XP, начисленное за этот день |
| `streak_after` | integer | Значение стрика после учёта этого дня |
| `notes` | text, nullable | Опциональные комментарии игрока к выполнению |
| `created_at` | timestamptz, default now() | Дата и время создания записи лога |

**Уникальный индекс:** `(habit_id, date)` — одна запись на привычку в день.

### 6.3. Таблица `habit_rewards` (опционально)

Связь выполнения привычки с наградами в Economy/Inventory.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор записи награды за привычку |
| `habit_log_id` | uuid, FK → habit_logs.id | Лог выполнения привычки, за который выдана награда |
| `transaction_id` | uuid, FK → transactions.id, nullable | Ссылка на транзакцию начисления валюты в Economy |
| `granted_item_id` | uuid, FK → items.id, nullable | Тип предмета, выданного как часть награды |
| `granted_quantity` | integer, default 0 | Количество выданного предмета |
| `created_at` | timestamptz, default now() | Дата и время начисления награды |

---

## 7. SingularityIntegration

### 7.1. Таблица `singularity_tasks`

Таски, полученные из внешнего таск-менеджера.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор записи задачи из Singularity |
| `player_id` | uuid, FK → players.id | Игрок, которому принадлежит задача |
| `external_id` | text | Идентификатор задачи во внешней системе Singularity |
| `title` | text | Название задачи из Singularity |
| `description` | text, nullable | Описание задачи из Singularity |
| `status` | text | Состояние задачи из Singularity: `open`, `completed`, `archived` и т.п. |
| `tags` | text[], nullable | Массив тегов задачи для фильтрации и маппинга на категории артефактов |
| `raw_data` | jsonb, nullable | Полный raw-ответ от API Singularity (для отладки и дополнительных полей) |
| `completed_at` | timestamptz, nullable | Дата и время выполнения задачи (по данным Singularity) |
| `created_at` | timestamptz, default now() | Дата и время первой синхронизации задачи |
| `updated_at` | timestamptz, default now() | Дата и время последнего обновления статуса задачи |

**Уникальный индекс:** `(player_id, external_id)`.

### 7.2. Таблица `task_mappings`

Правила маппинга: какие задачи Singularity порождают артефакты, какие теги соответствуют каким категориям.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор правила маппинга |
| `player_id` | uuid, FK → players.id | Игрок, для которого действует правило |
| `tag_filter` | text[], nullable | Теги Singularity, по которым фильтровать задачи (null = все задачи) |
| `artifact_category` | text | Категория артефакта, в которую маппятся отфильтрованные задачи |
| `artifact_template_id` | uuid, FK → artifact_templates.id, nullable | Шаблон артефакта для автоматической генерации (null = без автогенерации) |
| `is_active` | boolean, default true | Активно ли правило маппинга |
| `created_at` | timestamptz, default now() | Дата и время создания правила |
| `updated_at` | timestamptz, default now() | Дата и время последнего обновления правила |

> **Примечание:** `task_mappings` логически относится к SingularityIntegration (доставка данных), но ссылается на `artifact_templates` (домен Artifacts). Это допустимая связь: SingularityIntegration знает о шаблонах, но не принимает решений о наградах.

### 7.3. Таблица `sync_logs`

Журнал синхронизаций с Singularity.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор записи журнала синхронизации |
| `player_id` | uuid, FK → players.id | Игрок, для которого проводилась синхронизация |
| `started_at` | timestamptz | Дата и время начала синхронизации с Singularity |
| `finished_at` | timestamptz, nullable | Дата и время завершения синхронизации |
| `tasks_fetched` | integer, default 0 | Количество задач, полученных от Singularity за эту синхронизацию |
| `tasks_processed` | integer, default 0 | Количество задач, успешно обработанных системой |
| `errors` | jsonb, nullable | Массив объектов с ошибками, возникшими во время синхронизации |
| `status` | text | Статус синхронизации: `running` (в процессе), `success` (успешно), `partial` (частично), `failed` (ошибка) |

### 7.4. Таблица `singularity_habits` (опционально, на будущее)

Если часть привычек живёт в Singularity Habit Tracker.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор записи привычки из Singularity |
| `player_id` | uuid, FK → players.id | Игрок, которому принадлежит привычка |
| `external_id` | text | Идентификатор привычки во внешней системе Singularity Habit Tracker |
| `name` | text | Название привычки из Singularity |
| `raw_data` | jsonb, nullable | Полный raw-ответ от API Singularity (для отладки и дополнительных полей) |
| `created_at` | timestamptz, default now() | Дата и время первой синхронизации привычки |
| `updated_at` | timestamptz, default now() | Дата и время последнего обновления данных привычки |

---

## 8. OKR (Личные цели)

Личные OKR — отдельный слой над VISION, не артефакты и не часть интеграции с Singularity. В MVP все значения по KR задаются вручную через интерфейс приложения.

### 8.1. Таблица `okr_quarters`

Кварталы, автоматически генерируемые системой.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор квартала |
| `code` | text, unique | Код квартала: `2026-Q1`, `2026-Q2`, `2026-Q3`, `2026-Q4` |
| `start_date` | date | Дата начала квартала |
| `end_date` | date | Дата окончания квартала |
| `created_at` | timestamptz, default now() | Дата и время создания записи |

**Автоматическая генерация:** система создаёт 4 квартала на каждый год:
- Q1: 01.01–31.03
- Q2: 01.04–30.06
- Q3: 01.07–30.09
- Q4: 01.10–31.12

### 8.2. Таблица `okr_objectives`

Цели игрока на период в конкретной сфере VISION.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор цели |
| `player_id` | uuid, FK → players.id | Игрок‑владелец цели |
| `quarter_id` | uuid, FK → okr_quarters.id | Квартал, к которому привязана цель |
| `sphere` | text | Сфера VISION: `state` (Тело и энергия), `finances` (Деньги и свобода), `hobbies` (Хобби), `relationships` (Любовь и близость) |
| `title` | text | Краткая формулировка цели (Objective) |
| `purpose` | text, nullable | Мотивационный текст «зачем» эта цель. Показывается в screensaver как цитата. |
| `objective_type` | text, default 'normal' | Тип цели: `normal` (обычная), `stretch` (амбициозная), `crazy` (экстремальная). Выбирается при создании, не вычисляется автоматически. |
| `description` | text, nullable | Развёрнутое описание/контекст цели |
| `weight` | integer, default 1 | Вес цели внутри сферы (для будущей взвешенной агрегации прогресса по сфере) |
| `status` | text, default 'draft' | Статус цели: `draft` (черновик), `active` (рабочая), `completed` (квартал завершён), `carried_over` (перенесена в следующий квартал), `archived` (в истории) |
| `result_status` | text, nullable | Статус результата при закрытии квартала: `success` (>= 70%), `partial` (40–69%), `failed` (< 40%). Заполняется при переходе в `completed`. |
| `is_pinned` | boolean, default false | Признак закрепления для VISION-screensaver. Максимум 4 pinned на игрока. Pin запрещён, если `status = 'draft'`. |
| `parent_objective_id` | uuid, FK → okr_objectives.id, nullable | Ссылка на предыдущий экземпляр цели при переносе между кварталами (carried_over). |
| `created_at` | timestamptz, default now() | Дата и время создания цели |
| `updated_at` | timestamptz, default now() | Дата и время последнего изменения цели |

**Примечания:**

- В MVP прогресс цели не кэшируется в этой таблице, а вычисляется по связанным KR.
- **Переходы статусов:** `draft` → `active` автоматически, когда сумма весов KR = 100%. `active` → `completed` при закрытии квартала. `completed` → `archived` вручную или автоматически.
- **Pin-валидация:** `is_pinned = true` разрешено только если `status != 'draft'` и у Objective есть минимум 1 картинка в `objective_images`. Максимум 4 pinned на игрока — проверка на уровне бизнес-логики.
- Возможен более чем один Objective на сферу и период, но рекомендуется ограничивать их количество бизнес‑логикой (например, 1–3 цели на сферу за период).

### 8.3. Таблица `objective_images`

Картинки, привязанные к Objective. Хранятся в Supabase Storage.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор картинки |
| `objective_id` | uuid, FK → okr_objectives.id, ON DELETE CASCADE | Objective, к которому относится картинка |
| `player_id` | uuid, FK → players.id | Игрок‑владелец (денормализовано для удобства выборок) |
| `storage_path` | text | Путь к файлу в Supabase Storage (например, `objectives/<player_id>/<objective_id>/uuid.jpg`) |
| `sort_order` | integer, default 0 | Порядок сортировки в галерее |
| `created_at` | timestamptz, default now() | Дата и время загрузки |

**Инвариант:** если `objective.is_pinned = true`, то у Objective должна быть минимум 1 запись в `objective_images`.

### 8.4. Таблица `okr_key_results`

Измеримые результаты (Key Results) для каждой цели.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор ключевого результата |
| `objective_id` | uuid, FK → okr_objectives.id | Цель, к которой относится KR. Связь настроена как `ON DELETE CASCADE`: при удалении цели все связанные KR автоматически удаляются. |
| `player_id` | uuid, FK → players.id | Игрок‑владелец KR (денормализовано для удобства выборок) |
| `sphere` | text | Сфера VISION, дублирующая `okr_objectives.sphere` (источник истины — цель) |
| `title` | text | Формулировка KR (измеримый результат) |
| `type` | text, default 'numeric' | Тип KR: `numeric` (числовой), `binary` (сделано/нет), `qualitative` (оценка 0–1 по ощущениям) |
| `unit` | text, nullable | Единицы измерения: `%`, `steps`, `RUB`, `kg`, `hours_per_week`, `times_per_week` и т.п. |
| `weight` | integer, nullable | Вес KR в прогрессе Objective (шаг 5%, сумма всех KR Objective = 100%). Nullable при создании — Objective в состоянии «черновик», пока веса не распределены. |
| `start_date` | date | Дата начала отслеживания KR |
| `end_date` | date | Дата окончания (дедлайн) KR |
| `start_value` | numeric | Стартовое значение метрики |
| `current_value` | numeric | Текущее значение метрики (обновляется вручную через UI) |
| `target_value` | numeric | Целевое значение метрики к `end_date` |
| `progress_percent` | numeric, default 0 | Процент выполнения KR: 0–100 (кэшируется из `current_value` и `target_value`) |
| `okr_score` | numeric, default 0 | Оценка KR по шкале OKR: 0–1 (например, 0.3 / 0.7 / 1.0) |
| `status` | text, default 'start' | Статус по порогам: `start` (0–30%), `in_progress` (30–70%), `near_goal` (70–99%), `done` (>=100%) |
| `comment` | text, nullable | Свободный комментарий/заметки по KR |
| `source` | text, default 'manual' | Источник данных для KR: `manual` (обновляется вручную пользователем), `singularity` (обновляется автоматически из данных Singularity, Epic 4+). В Epic 1–2 все KR фактически `manual` |
| `last_updated_at` | timestamptz, default now() | Дата и время последнего обновления KR |

**Примечания:**

- Поле «осталось дней» (`days_left`) **не хранится** в БД, а вычисляется на лету как `end_date - current_date` и может возвращаться в API‑ответах для UI.
- Расчёт `progress_percent`, `okr_score` и производного `status` выполняется бизнес‑логикой (или DB‑функцией); таблица хранит кэш.
- **Формула прогресса (numeric):** `progress_percent = ((current_value - start_value) / NULLIF(target_value - start_value, 0)) × 100`, ограничено 0–100.
- **OKR‑оценка:** `okr_score = progress_percent / 100`, ограничено 0–1.
- **Статус по порогам:** `start` (0–30%), `in_progress` (30–70%), `near_goal` (70–99%), `done` (>=100%).
- **Взвешенный прогресс Objective:** `progress_objective = Σ(weight_kr × progress_percent_kr / 100)`. Пока `weight` не распределены (сумма ≠ 100%), Objective = «черновик», прогресс = 0.
- **CHECK-ограничение:** `CHECK (weight IS NULL OR (weight >= 0 AND weight <= 100 AND weight % 5 = 0))`.

### 8.5. Таблица `okr_kr_log`

Единый лог событий по KR для ретроспективы и аудита.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор события |
| `kr_id` | uuid, FK → okr_key_results.id | KR, к которому относится событие |
| `objective_id` | uuid, FK → okr_objectives.id | Цель, к которой относится KR (денормализовано) |
| `player_id` | uuid, FK → players.id | Игрок‑владелец (денормализовано) |
| `event_type` | text | Тип события: `create`, `update_value`, `update_target`, `update_weight`, `update_status`, `delete`, `comment`, `quarter_closed` |
| `old_value` | jsonb, nullable | Старое значение (например, `{"weight": 25, "target_value": 100}`) |
| `new_value` | jsonb, nullable | Новое значение (например, `{"weight": 30, "target_value": 120}`) |
| `note` | text, nullable | Текстовый комментарий пользователя (например, выводы по итогам спринта) |
| `created_at` | timestamptz, default now() | Дата и время события |
| `created_by` | uuid, FK → players.id | Игрок, совершивший изменение (для MVP = `player_id`) |

**Примечания:**

- Запись создаётся при каждом изменении значимых параметров KR: `current_value`, `target_value`, `weight`, `status`, а также при создании/удалении KR и добавлении комментария.
- В UI лог фильтруется по периоду, типу события, конкретному KR.

### 8.6. Таблица `retrospective_notes`

Заметки рефлексии по итогам квартала.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор записи |
| `quarter_id` | uuid, FK → okr_quarters.id | Квартал, к которому относится ретроспектива |
| `player_id` | uuid, FK → players.id | Игрок‑владелец |
| `what_went_well` | text, nullable | «Что получилось хорошо в этом квартале?» |
| `what_didnt_work` | text, nullable | «Что не сработало?» |
| `what_to_change` | text, nullable | «Что изменить в следующем квартале?» |
| `created_at` | timestamptz, default now() | Дата и время создания |
| `updated_at` | timestamptz, default now() | Дата и время последнего обновления |

**Уникальный индекс:** `(player_id, quarter_id)` — одна запись ретроспективы на квартал.

### 8.7. Таблица `achievements`

Справочник косметических ачивок.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор ачивки |
| `code` | text, unique | Машинно-читаемый код (`stretch_quarter`, `kr_closed`, `retrospective_master`) |
| `name` | text | Человекочитаемое название |
| `description` | text | Описание условия получения |
| `icon` | text | Путь к иконке в Supabase Storage |
| `type` | text | Тип: `okr_objective`, `okr_kr`, `okr_consistency`, `okr_retrospective` |
| `rarity` | text, default 'common' | Редкость: `common`, `uncommon`, `rare` |
| `is_active` | boolean, default true | Активна ли ачивка для выдачи |
| `created_at` | timestamptz, default now() | Дата и время создания |

### 8.8. Таблица `user_achievements`

Связь игрока с полученными ачивками.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid, PK | Уникальный идентификатор записи |
| `player_id` | uuid, FK → players.id | Игрок, получивший ачивку |
| `achievement_id` | uuid, FK → achievements.id | Шаблон ачивки |
| `earned_at` | timestamptz, default now() | Дата и время получения |
| `context` | jsonb, nullable | Контекст: ссылка на Objective/KR/quarter, за что выдана (например, `{"objective_id": "uuid", "quarter_id": "uuid"}`) |

**Уникальный индекс:** `(player_id, achievement_id)` — одна ачивка на игрока (не выдаётся повторно).

---

## 9. Сводная диаграмма связей

```
players
  ├── wallets (1:1)
  │     └── transactions (1:N)
  ├── inventory_items (1:N) ──→ items
  ├── purchases (1:N) ──→ products ──→ items
  ├── artifacts (1:N)
  │     ├── artifact_stages (1:N)
  │     ├── artifact_task_links (1:N) ──→ singularity_tasks
  │     ├── artifact_evaluations (1:1)
  │     ├── artifact_rewards (1:1) ──→ transactions, items
  │     └── artifact_templates (N:1, nullable)
  ├── habits (1:N)
  │     ├── habit_logs (1:N)
  │     └── habit_rewards (1:N) ──→ transactions, items
  ├── okr_quarters (1:N)
  │     └── okr_objectives (1:N)
  │           ├── okr_key_results (1:N)
  │           ├── objective_images (1:N)
  │           └── okr_kr_log (1:N)
  ├── retrospective_notes (1:N) ──→ okr_quarters
  ├── user_achievements (1:N) ──→ achievements
  └── singularity_tasks (1:N)
        └── artifact_task_links (1:N)
  └── task_mappings (1:N) ──→ artifact_templates
  └── sync_logs (1:N)
```

> **Примечание:** таблица `artifacts` содержит гибкое поле `metadata` jsonb, которое позволяет расширять поведение артефактов без изменения основной структуры таблицы. В частности, для категории `VISION` в `metadata` хранится текст видения, 4 сферы с картинками и настройки screensaver. Это даёт возможность добавлять новые специализированные типы артефактов (с собственной структурой данных) без добавления новых таблиц или колонок.

---

## 9. Дальнейшая эволюция схемы

- Этот документ — отправная точка для создания таблиц в Supabase.
- По мере реализации Epic 1:
  - поля могут уточняться (типы, enum-ы, индексы, CHECK-ограничения);
  - могут появляться дополнительные таблицы (категории артефактов, навыки, связи бафов);
  - будут добавлены RLS-политики для изоляции данных по `player_id`.
- Любое изменение схемы должно:
  - обсуждаться в задачах;
  - отражаться в этом документе;
  - проводиться через миграции Supabase (SQL или миграционные файлы).

---

## 10. Инварианты и гарантии

| Инвариант | Модуль-гарант |
|-----------|---------------|
| `wallets.balance >= 0` | Economy |
| `wallets.balance = SUM(transactions.amount)` | Economy |
| `inventory_items.quantity >= 0` | Inventory |
| Один `(habit_id, date)` → одна запись в `habit_logs` | Habits |
| Один `(artifact_id, stage)` → одна запись в `artifact_stages` | Artifacts |
| `artifact_rewards.currency_transaction_id` → валидная транзакция в Economy | Artifacts + Economy |
| `purchases.status = 'completed'` → `transaction_id` не null | Shop + Economy |
| `singularity_tasks (player_id, external_id)` — уникальны | SingularityIntegration |
| Один активный VISION на игрока: `UNIQUE (player_id) WHERE category = 'VISION' AND lifecycle_status = 'active'` | Artifacts |
| Один KR всегда принадлежит одному Objective | OKR |
| `okr_key_results.sphere` = `okr_objectives.sphere` (источник истины — цель) | OKR |
| `okr_key_results.weight` сумма = 100% для активного Objective | OKR |
| `objective.is_pinned = true` → минимум 1 картинка в `objective_images` | OKR |
| Максимум 4 pinned Objective на игрока | OKR (бизнес-логика) |
| Один активный VISION на игрока: `UNIQUE (player_id) WHERE category = 'VISION' AND lifecycle_status = 'active'` | Artifacts |
| `retrospective_notes (player_id, quarter_id)` — уникальны | OKR |
| `user_achievements (player_id, achievement_id)` — уникальны | OKR |
