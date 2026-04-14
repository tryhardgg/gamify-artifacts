# OKR API Contracts

Минимальные контракты Edge Functions для работы с OKR. Все эндпоинты требуют аутентификации (Supabase Auth). `player_id` извлекается из контекста токена.

---

## 1. GET /okr/current

Получить список Objectives и KR для выбранного квартала.

**Параметры запроса:**

| Параметр | Тип | Обязательное | Описание |
|----------|-----|--------------|----------|
| `quarter_id` | uuid | Нет | ID квартала. По умолчанию — текущий квартал. |

**Ответ 200:**

```json
{
  "quarter": {
    "id": "uuid",
    "code": "2026-Q2",
    "start_date": "2026-04-01",
    "end_date": "2026-06-30"
  },
  "spheres": [
    {
      "code": "state",
      "title": "Тело и энергия",
      "progress_percent": 45,
      "objectives": [
        {
          "id": "uuid",
          "sphere": "state",
          "title": "...",
          "purpose": "...",
          "objective_type": "stretch",
          "status": "active",
          "result_status": null,
          "is_pinned": true,
          "progress_percent": 45,
          "kr_count": 4,
          "key_results": [
            {
              "id": "uuid",
              "title": "...",
              "type": "numeric",
              "unit": "hours",
              "weight": 25,
              "start_value": 5,
              "current_value": 6.5,
              "target_value": 8,
              "progress_percent": 50,
              "okr_score": 0.5,
              "status": "in_progress",
              "source": "manual",
              "last_updated_at": "2026-04-10T12:00:00Z"
            }
          ]
        }
      ]
    }
  ]
}
```

---

## 2. POST /okr/objectives

Создать новую Objective.

**Тело запроса:**

| Поле | Тип | Обязательное | Описание |
|------|-----|--------------|----------|
| `quarter_id` | uuid | Да | Квартал |
| `sphere` | text | Да | `state`, `finances`, `hobbies`, `relationships` |
| `title` | text | Да | Формулировка цели |
| `purpose` | text | Нет | Мотивационный текст «зачем» |
| `objective_type` | text | Нет | `normal` (default), `stretch`, `crazy` |
| `description` | text | Нет | Развёрнутое описание |

**Ответ 201:**

```json
{
  "id": "uuid",
  "quarter_id": "uuid",
  "sphere": "state",
  "title": "...",
  "purpose": "...",
  "objective_type": "stretch",
  "status": "draft",
  "is_pinned": false,
  "created_at": "2026-04-11T10:00:00Z"
}
```

**Ошибки:**
- `400` — невалидная сфера или тип
- `404` — квартал не найден

---

## 3. PATCH /okr/objectives/:id

Обновить Objective.

**Тело запроса (любые поля):**

| Поле | Тип | Описание |
|------|-----|----------|
| `title` | text | Новая формулировка |
| `purpose` | text | Новый «зачем» |
| `objective_type` | text | `normal`, `stretch`, `crazy` |
| `description` | text | Описание |
| `is_pinned` | boolean | Закрепить в screensaver |

**Ответ 200:** обновлённый объект Objective.

**Ошибки:**
- `400` — попытка pin-ить черновик или нет картинок
- `404` — Objective не найдена

---

## 4. DELETE /okr/objectives/:id

Удалить Objective. Все связанные KR удаляются (`ON DELETE CASCADE`).

**Ответ 204** — успешно удалено.

**Ошибки:**
- `404` — Objective не найдена

---

## 5. POST /okr/key-results

Создать KR для Objective.

**Тело запроса:**

| Поле | Тип | Обязательное | Описание |
|------|-----|--------------|----------|
| `objective_id` | uuid | Да | Родительская цель |
| `title` | text | Да | Формулировка KR |
| `type` | text | Да | `numeric`, `binary`, `qualitative` |
| `unit` | text | Нет | Единицы измерения |
| `start_value` | numeric | Да | Стартовое значение |
| `target_value` | numeric | Да | Целевое значение |
| `current_value` | numeric | Нет | Текущее значение (default = start_value) |
| `start_date` | date | Да | Дата начала |
| `end_date` | date | Да | Дата окончания |
| `weight` | integer | Нет | Вес KR (шаг 5%). Не обязателен при создании. |

**Ответ 201:**

```json
{
  "id": "uuid",
  "objective_id": "uuid",
  "title": "...",
  "type": "numeric",
  "unit": "hours",
  "weight": null,
  "start_value": 5,
  "current_value": 5,
  "target_value": 8,
  "progress_percent": 0,
  "okr_score": 0,
  "status": "start",
  "source": "manual",
  "last_updated_at": "2026-04-11T10:00:00Z"
}
```

**Ошибки:**
- `400` — невалидный тип KR
- `404` — Objective не найдена

---

## 6. PATCH /okr/key-results/:id

Обновить KR. Клиент отправляет только изменяемые поля. Сервер пересчитывает `progress_percent`, `okr_score`, `status`.

**Тело запроса (любые поля):**

| Поле | Тип | Описание |
|------|-----|----------|
| `current_value` | numeric | Новое текущее значение |
| `target_value` | numeric | Новое целевое значение |
| `weight` | integer | Новый вес (шаг 5%) |
| `title` | text | Новая формулировка |
| `end_date` | date | Новая дата окончания |
| `comment` | text | Комментарий |

**Ответ 200:** обновлённый объект KR с пересчитанными полями.

**Ошибки:**
- `400` — вес не кратен 5 или сумма весов ≠ 100%
- `404` — KR не найден

---

## 7. GET /okr/key-results/:id/history

Получить лог изменений KR.

**Параметры запроса:**

| Параметр | Тип | Обязательное | Описание |
|----------|-----|--------------|----------|
| `period` | text | Нет | `quarter`, `sprint`, или произвольный диапазон |
| `event_type` | text | Нет | Фильтр по типу: `create`, `update_value`, `update_weight`, `update_target`, `update_status`, `delete`, `comment` |

**Ответ 200:**

```json
[
  {
    "id": "uuid",
    "event_type": "update_weight",
    "old_value": { "weight": 25 },
    "new_value": { "weight": 30 },
    "note": null,
    "created_at": "2026-04-10T12:00:00Z",
    "created_by": "uuid"
  }
]
```

---

## 8. POST /okr/objectives/:id/images

Загрузить картинку для Objective.

**Тело запроса:** multipart/form-data с файлом изображения.

**Ответ 201:**

```json
{
  "id": "uuid",
  "storage_path": "objectives/<player_id>/<objective_id>/uuid.jpg",
  "sort_order": 0,
  "created_at": "2026-04-11T10:00:00Z"
}
```

**Ошибки:**
- `400` — неверный формат файла
- `404` — Objective не найдена

---

## 9. DELETE /okr/objectives/:id/images/:image_id

Удалить картинку Objective.

**Ответ 204** — успешно удалено.

**Ошибки:**
- `400` — попытка удалить последнюю картинку pinned Objective
- `404` — картинка не найдена

---

## 10. POST /okr/objectives/:id/carry-over

Перенести Objective в следующий квартал.

**Тело запроса:**

| Поле | Тип | Обязательное | Описание |
|------|-----|--------------|----------|
| `target_quarter_id` | uuid | Да | Квартал назначения |
| `copy_kr` | boolean | Нет | Скопировать KR (default: true) |
| `adjust_weights` | boolean | Нет | Скорректировать веса (default: false) |

**Ответ 201:**

```json
{
  "old_objective_id": "uuid",
  "new_objective_id": "uuid",
  "old_status": "carried_over",
  "new_status": "draft"
}
```

---

## 11. GET /okr/retrospective

Получить данные для экрана ретроспективы.

**Параметры запроса:**

| Параметр | Тип | Обязательное | Описание |
|----------|-----|--------------|----------|
| `quarter_id` | uuid | Да | Квартал для ретроспективы |

**Ответ 200:**

```json
{
  "quarter": { "id": "uuid", "code": "2026-Q1" },
  "summary": {
    "total_objectives": 5,
    "completed": 3,
    "failed": 1,
    "avg_stretch_percent": 68
  },
  "spheres_progress": [
    { "code": "state", "progress_percent": 72 },
    { "code": "finances", "progress_percent": 55 }
  ],
  "objectives": [
    {
      "id": "uuid",
      "title": "...",
      "sphere": "state",
      "objective_type": "stretch",
      "final_percent": 75,
      "result_status": "success"
    }
  ],
  "retrospective_notes": {
    "what_went_well": "...",
    "what_didnt_work": "...",
    "what_to_change": "..."
  },
  "comparison": {
    "prev_quarter_avg": 60,
    "current_quarter_avg": 68,
    "prev_failed_count": 2,
    "current_failed_count": 1,
    "sphere_trends": { "state": "up", "finances": "down" }
  }
}
```

---

## 12. POST /okr/retrospective/notes

Сохранить заметки рефлексии.

**Тело запроса:**

| Поле | Тип | Обязательное | Описание |
|------|-----|--------------|----------|
| `quarter_id` | uuid | Да | Квартал |
| `what_went_well` | text | Нет | Что получилось хорошо |
| `what_didnt_work` | text | Нет | Что не сработало |
| `what_to_change` | text | Нет | Что изменить |

**Ответ 201:** сохранённые заметки.

---

## 13. POST /okr/quarters/close

Закрыть квартал.

**Тело запроса:**

| Поле | Тип | Обязательное | Описание |
|------|-----|--------------|----------|
| `quarter_id` | uuid | Да | Квартал для закрытия |

**Ответ 200:**

```json
{
  "quarter_id": "uuid",
  "closed_objectives_count": 5,
  "result": "success"
}
```

**Побочные эффекты:**
- Все `active` Objective → `completed` с фиксацией `progress_percent` и `result_status`.
- Записи в `okr_kr_log` с `event_type = 'quarter_closed'`.

---

## Общие ошибки

| Статус | Описание |
|--------|----------|
| `401` | Неаутентифицирован |
| `403` | Нет доступа (чужой player_id) |
| `404` | Ресурс не найден |
| `409` | Конфликт (например, сумма весов ≠ 100%) |
| `500` | Внутренняя ошибка сервера |
