# User Stories — Epic 0/1: OKR + VISION

User stories и use cases для MVP: личный OKR-трекер + VISION-экран.

---

## US-1: Просмотр OKR текущего квартала

**Как** игрок, **я хочу** видеть свои цели и ключевые результаты по 4 сферам, **чтобы** понимать, как продвигаюсь к целям квартала.

**Acceptance Criteria:**
- Given: у игрока есть Objective в выбранном квартале
- When: игрок открывает экран OKR
- Then: видит 4 блока сфер, в каждом — список Objective с прогресс-барами, бейджами статуса и количеством KR
- And: может свернуть/развернуть блок сферы кликом по заголовку
- And: может выбрать другой квартал через дропдаун

---

## US-2: Создание Objective

**Как** игрок, **я хочу** создать новую цель на квартал, **чтобы** зафиксировать направление, в котором хочу двигаться.

**Acceptance Criteria:**
- Given: игрок на экране OKR
- When: нажимает «Создать Objective» и заполняет форму (сфера, title, purpose, тип)
- Then: Objective создаётся со статусом `draft`
- And: прогресс = 0%, бейдж «Черновик»
- And: pin запрещён

---

## US-3: Создание Key Result

**Как** игрок, **я хочу** добавить измеримый результат к цели, **чтобы** иметь возможность отслеживать прогресс.

**Acceptance Criteria:**
- Given: Objective существует
- When: игрок добавляет KR (title, type, start_value, target_value)
- Then: KR создаётся с `weight = null`
- And: Objective остаётся в статусе `draft` (веса не распределены)

---

## US-4: Распределение весов KR

**Как** игрок, **я хочу** распределить 100% веса между KR цели, **чтобы** Objective начала считать прогресс.

**Acceptance Criteria:**
- Given: у Objective есть KR без весов
- When: игрок назначает веса всем KR (сумма = 100%, шаг 5%)
- Then: статус Objective меняется на `active`
- And: прогресс начинает считаться по формуле Σ(weight × progress_percent_kr / 100)
- And: pin становится доступен

---

## US-5: Обновление прогресса KR

**Как** игрок, **я хочу** обновить текущее значение KR, **чтобы** видеть, как продвигаюсь.

**Acceptance Criteria:**
- Given: KR в статусе `active`
- When: игрок вводит новое `current_value`
- Then: бэкенд пересчитывает `progress_percent`, `okr_score`, `status`
- And: прогресс Objective обновляется
- And: запись в `okr_kr_log` с `event_type = 'update_value'`

---

## US-6: Pin/Unpin Objective для screensaver

**Как** игрок, **я хочу** закрепить до 4 целей для отображения в screensaver, **чтобы** видеть их картинки и purpose во время бездействия.

**Acceptance Criteria:**
- Given: Objective в статусе `active` и имеет ≥ 1 картинку
- When: игрок нажимает pin
- Then: `is_pinned = true`, максимум 4 на игрока
- And: Objective появляется в screensaver
- When: игрок нажимает unpin
- Then: `is_pinned = false`, Objective убирается из screensaver

---

## US-7: VISION Screensaver

**Как** игрок, **я хочу** чтобы через 3 минуты бездействия включался screensaver с моим VISION, **чтобы** напоминать себе, ради чего я работаю над целями.

**Acceptance Criteria:**
- Given: приложение открыто, 3 минуты бездействия
- When: screensaver активируется
- Then: фон — 4 квадранта из картинок pinned Objective
- And: по центру — vision_text (полупрозрачный)
- And: раз в 3 минуты → purpose рандомной pinned Objective на 1 минуту
- When: игрок кликает в любом месте
- Then: screensaver закрывается, возврат на предыдущий экран

---

## US-8: Ретроспектива квартала

**Как** игрок, **я хочу** проанализировать итоги квартала, **чтобы** улучшить навык постановки целей.

**Acceptance Criteria:**
- Given: квартал завершён (статус `completed`)
- When: игрок открывает экран ретроспективы
- Then: видит сводку (всего/завершены/провалены, средний % Stretch)
- And: видит список Objective с финальным % и цветовой зоной
- And: может заполнить 3 поля рефлексии (what_went_well, what_didnt_work, what_to_change)
- And: видит сравнение с прошлым кварталом (тренды по сферам)

---

## US-9: Закрытие квартала

**Как** игрок, **я хочу** закрыть квартал, **чтобы** зафиксировать итоги и перейти к следующему.

**Acceptance Criteria:**
- Given: есть активные Objective в текущем квартале
- When: игрок нажимает «Закрыть квартал»
- Then: все `active` Objective → `completed` с фиксацией `progress_percent`
- And: `result_status` заполняется: `success` (>= 70%), `partial` (40–69%), `failed` (< 40%)
- And: pinned Objective сбрасывают `is_pinned = false`

---

## US-10: Перенос Objective в следующий квартал

**Как** игрок, **я хочу** перенести незавершённую цель в следующий квартал, **чтобы** продолжить работу без потери истории.

**Acceptance Criteria:**
- Given: Objective в статусе `completed` с result_status ≠ `success`
- When: игрок выбирает «Перенести в следующий квартал»
- Then: старая Objective → `carried_over`
- And: создаётся новая Objective с `parent_objective_id`
- And: KR копируются (опционально), веса можно скорректировать

---

## US-11: Просмотр достижений

**Как** игрок, **я хочу** видеть коллекцию своих ачивок, **чтобы** отслеживать прогресс геймификации.

**Acceptance Criteria:**
- Given: у игрока есть полученные ачивки
- When: игрок открывает экран «Достижения»
- Then: видит сетку иконок с фильтрами по типу (OKR, ретроспектива, консистентность)
- And: полученные ачивки яркие, не полученные — серые
- And: при получении — всплывающий тост

---

## US-12: История изменений KR

**Как** игрок, **я хочу** видеть историю изменений KR, **чтобы** анализировать, как менялись мои планы.

**Acceptance Criteria:**
- Given: KR имеет записи в `okr_kr_log`
- When: игрок открывает историю KR
- Then: видит список событий с типом, старым/новым значением, датой
- And: может фильтровать по типу события и периоду

---

## Use Case: Создание Objective с KR

**Актор:** Игрок  
**Предусловия:** Игрок аутентифицирован, квартал выбран.

**Основной поток:**
1. Игрок нажимает «Создать Objective» на экране OKR.
2. Заполняет форму: сфера, title, purpose, тип (normal/stretch/crazy).
3. Система создаёт Objective со статусом `draft`.
4. Игрок добавляет 3–4 KR (title, type, start_value, target_value).
5. Система создаёт KR с `weight = null`.
6. Игрок распределяет веса (сумма = 100%, шаг 5%).
7. Система меняет статус Objective на `active`.
8. Objective появляется в списке OKR с прогресс-баром.

**Постусловия:** Objective в статусе `active`, KR с весами, прогресс считается.

---

## Use Case: Закрытие квартала

**Актор:** Игрок  
**Предусловия:** Квартал активен, есть Objective в статусе `active`.

**Основной поток:**
1. Игрок нажимает «Закрыть квартал».
2. Система подтверждает действие (модалка).
3. Все `active` Objective → `completed`.
4. Для каждой Objective: `result_status` = `success`/`partial`/`failed` по %.
5. Pinned Objective сбрасывают `is_pinned = false`.
6. Система предлагает открыть ретроспективу.

**Постусловия:** Квартал закрыт, Objective в истории, ретроспектива доступна.

---

## Use Case: VISION Screensaver

**Актор:** Система (автоматически)  
**Предусловия:** Приложение открыто, 3 минуты бездействия, есть ≥ 1 pinned Objective.

**Основной поток:**
1. Таймер бездействия достигает 180 секунд.
2. Система показывает screensaver:
   - Фон: 4 квадранта из картинок pinned Objective.
   - Центр: vision_text (полупрозрачный).
3. Через 3 минуты → purpose рандомной pinned Objective на 1 минуту.
4. Возврат к основному режиму.
5. Цикл повторяется.

**Альтернативный поток:**
- Игрок кликает в любом месте → screensaver закрывается, возврат на предыдущий экран.

**Постусловия:** Screensaver деактивирован, таймер сброшен.

---

## Связь User Stories ↔ API эндпоинты

| US | Краткое название | Связанные API эндпоинты |
|----|------------------|------------------------|
| US-1 | Просмотр OKR квартала | `GET /okr/current`, `GET /okr/retrospective` |
| US-2 | Создание Objective | `POST /okr/objectives` |
| US-3 | Создание Key Result | `POST /okr/key-results` |
| US-4 | Распределение весов KR | `PATCH /okr/key-results/:id` (многократный вызов для каждого KR, пока сумма ≠ 100%) |
| US-5 | Обновление прогресса KR | `PATCH /okr/key-results/:id`, `GET /okr/key-results/:id/history` |
| US-6 | Pin/Unpin Objective | `PATCH /okr/objectives/:id`, `POST /okr/objectives/:id/images` (если нет картинок) |
| US-7 | VISION Screensaver | `GET /artifacts/vision` (VISION metadata), `GET /okr/current?is_pinned=true` (pinned Objective + картинки) |
| US-8 | Ретроспектива квартала | `GET /okr/retrospective`, `POST /okr/retrospective/notes` |
| US-9 | Закрытие квартала | `POST /okr/quarters/close` |
| US-10 | Перенос Objective | `POST /okr/objectives/:id/carry-over` |
| US-11 | Просмотр достижений | `GET /okr/achievements` (справочник + полученные) |
| US-12 | История изменений KR | `GET /okr/key-results/:id/history` |

> **Примечание:** US-4 (распределение весов) реализуется через серию `PATCH /okr/key-results/:id` — каждый вызов обновляет вес одного KR. Бэкенд после каждого PATCH проверяет, что сумма весов Objective = 100%, и при совпадении меняет статус Objective на `active`. Альтернатива (на будущее): батч-эндпоинт `PATCH /okr/objectives/:id/key-results/weights`.
