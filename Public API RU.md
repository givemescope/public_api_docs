# Public API getmatch

Документ для пользователей Public API getmatch.

## 1. Базовая информация

- Базовый URL Public API: `https://getmatch.ru/api/integrations/v1`
- Формат данных: `application/json`

## 2. Аутентификация

Public API использует OAuth2 Bearer Token.

### 2.1. Что нужно для подключения

- `client_id`
- `client_secret`
- `manage_token` (для настройки redirect_uri)
- `redirect_uri`


### 2.2. Заголовок авторизации

Во все запросы к Public API (за исключением работы с OAuth) передавайте:

```
Authorization: Bearer <access_token>
```

### 2.3. Настройка redirect_uri

Для настройки redirect-ссылок используется эндпоинт `PUT https://getmatch.ru/api/oauth/clients/{client_id}`. 
В заголовке авторизации необходимо указать manage_token.

Пример запроса:
```bash
curl -X PUT 'https://getmatch.ru/api/oauth/clients/{client_id}' -H 'Content-Type: application/json' -H 'Authorization: Bearer {manage_token}' --data-raw '{"redirect_uris": ["<http://localhost:8080>"]}'
```

Для получения списка текущих redirect_uris используйте метод GET:
```bash
curl -X GET 'https://getmatch.ru/api/oauth/clients/{client_id}' -H 'Content-Type: application/json' -H 'Authorization: Bearer {manage_token}'
```


### 2.4. Получение и обновление токена (OAuth2)

Используются OAuth-эндпоинты:

- `POST https://getmatch.ru/api/oauth/token` - обменять `authorization_code` на `access_token` и `refresh_token`
- `POST https://getmatch.ru/api/oauth/refresh` - обновить `access_token` по `refresh_token`

Для получения authorization_code пользователь должен выполнить вход в getmatch по ссылке: https://getmatch.ru/employer/oauth?redirect_uri={redirect_uri}&client_id={client_id}. 
После успешного входа пользователь будет перенаправлен на указанный redirect_uri с дополнительным параметром code={authorization_code}.

Пример запроса на получение access_token:
```bash
curl -X POST 'https://getmatch.ru/api/oauth/token' -H 'Content-Type: application/json' --data-raw '{"grant_type": "authorization_code", "client_id": "{client_id}", "client_secret": "{client_secret}", "code": "{authorization_code}"}'
```

Пример запроса на обновление access_token:
```bash
curl -X POST 'https://getmatch.ru/api/oauth/refresh' -H 'Content-Type: application/json' --data-raw '{"grant_type": "refresh_token", "refresh_token": "{refresh_token}"}'
```

Пример ответа на получение или обновление access_token:
```json
{"access_token": "{access_token}", "token_type": "bearer", "expires_at": "2026-01-01T09:30:00.012345", "refresh_token": "{refresh_token}"}
```


## 3. Где смотреть полную спецификацию (Swagger / OpenAPI)

- Swagger UI: [https://getmatch.ru/api/integrations/docs](https://getmatch.ru/api/integrations/docs)
- OpenAPI JSON: [https://getmatch.ru/api/integrations/openapi.json](https://getmatch.ru/api/integrations/openapi.json)


## 4. Доступные эндпоинты

Ниже перечислены эндпоинты Public API (`/api/integrations/v1/...`).

### 4.1. Общая информация

1. `GET /me`
Назначение: получить информацию о текущем авторизованном рекрутере и его компании.

### 4.2. Лимиты API

1. `GET /limits/get`
Назначение: получить текущие и максимальные лимиты компании:
- `daily` - дневной лимит запросов.
- `contacts` - лимит на раскрытие контактов.

### 4.3. Вакансии

1. `GET /vacancies/`
Назначение: получить список активных вакансий авторизованной компании.

2. `GET /employers/{company_id}/vacancies/active?page=<int>&per_page=<int>`
Назначение: получить активные вакансии компании с пагинацией. Вдохновлено HH API.
Параметры:
- `company_id` - ID компании (можно получить в /me).
- `page` - номер страницы, начиная с `0`.
- `per_page` - размер страницы (`1..200`).

3. `POST /employers/{company_id}/vacancies/{vacancy_id}/archive`
Назначение: снять вакансию с публикации.
Параметры:
- `company_id` - ID компании (можно получить в `/me`).
- `vacancy_id` - hash ID вакансии из `GET /vacancies/` или `GET /employers/{company_id}/vacancies/active`.
Успешный ответ: `{}`.

### 4.4. Черновики вакансий

1. `POST /employers/{company_id}/vacancies/drafts`  
Назначение: создать черновик вакансии и отправить его на валидацию.

2. `GET /employers/{company_id}/vacancies/drafts`  
Назначение: получить список черновиков текущего рекрутера (отсортированы по `updated_at DESC`).  
Важно: черновики в статусе `accepted` в этот список не попадают.

3. `GET /employers/{company_id}/vacancies/drafts/{draft_id}`  
Назначение: получить один черновик по ID.

4. `PATCH /employers/{company_id}/vacancies/drafts/{draft_id}`  
Назначение: частично обновить черновик и повторно отправить его на валидацию.

5. `POST /employers/{company_id}/vacancies/drafts/{draft_id}/publish`  
Назначение: поставить публикацию валидного черновика в очередь.  
Успешный ответ: `{"status": "queued"}`.

6. `DELETE /employers/{company_id}/vacancies/drafts/{draft_id}`  
Назначение: удалить черновик (если он еще не связан с финальной вакансией).

#### 4.4.1. Поля payload для create/update

Для `POST` обязательны:
- `position` - название вакансии.
- `salary_display_from`, `salary_display_to` - границы вилки.
- `salary_taxes` - `net` или `gross`.
- `salary_is_total` - совокупная ли компенсация.
- `type` - `only_web` или `web_and_tg`.

Опциональные поля:
- `location_requirements` - массив объектов вида `{"location_raw": "<текст локации>"}`.
- `salary_currency` - `rub` / `usd` / `eur` (также принимаются `₽`, `$`, `€`).
- `salary_hidden`, `cover_letter_required`, `cover_letter_placeholder`.
- `language`, `description`, `required_years_of_experience`.
- `location_validation`, `auto_prolong`.

Для `PATCH` передается только `payload` с изменяемыми полями (частичное обновление).

#### 4.4.2. Жизненный цикл черновика

- `new` - технический начальный статус.
- `filling` - система автодополняет поля.
- `validating` - выполняется проверка обязательных полей и контента.
- `rejected` - есть ошибки валидации (подробности в `errors`).
- `validated` - черновик готов к публикации.
- `publishing` - публикация поставлена в очередь.
- `accepted` - создана финальная вакансия, в `vacancy_id` записан ID вакансии.

Правила:
- редактирование (`PATCH`) разрешено только для `new`, `rejected`, `validated`;
- публикация (`/publish`) разрешена только для `validated` и при `vacancy_id = null`;
- удаление запрещено, если статус `accepted` или уже есть `vacancy_id`.

#### 4.4.3. Ошибки валидации

В `errors` возвращается массив объектов:
- `code` - тип ошибки (`required_field`, `max_length`, `content_policy`, `invalid_format`, `bad_words`, `other`);
- `justification` - человекочитаемое объяснение;
- `field_name` - поле с ошибкой (например, `description`, `location_requirements[0].format`).

### 4.5. Отклики

1. `GET /negotiations?vacancy_id=<vacancy_id>`
Назначение: получить коллекции откликов по вакансии (счетчики по статусам).

2. `GET /negotiations/{collection_name}?vacancy_id=<vacancy_id>&page=<int>&per_page=<int>`
Назначение: получить список откликов в выбранной коллекции с пагинацией.
Поддерживаемые `collection_name`:
- `all_applications`
- `in_progress_applications`
- `rejected_applications`
- `hired_applications`
- `pending_applications`

3. `GET /applications/{candidate_id}`
Назначение: получить резюме/профиль кандидата из отклика.
Параметры query:
- `open_contacts` - опциональный boolean-флаг, по умолчанию `false`.
Правила:
- по умолчанию отклик возвращается в закрытом виде: `contact` пустой, `last_name` и `birth_date` равны `null`;
- если передан `open_contacts=true`, API попытается раскрыть контакты и тогда запрос может повлиять на лимиты раскрытия контактов.

4. `POST /applications/{candidate_id}`
Назначение: рассмотреть отклик (одобрить или отклонить).
Payload:
- `resolution` - обязательное поле: `approve` или `reject`.
- `reason` - опциональный текст причины отказа.
- `forward_reason` - опциональный флаг, пересылать ли `reason` кандидату.
Ответ:
- `id` - hash_id отклика.
- `state.id` - итоговый статус (`pending`, `in_progress`, `rejected`, `hired`).
- `state.name` - человекочитаемое название статуса.
- `updated_at` - время последнего изменения статуса.

5. `PUT /applications/{candidate_id}`
Назначение: обновить клиентский статус отклика.
Payload:
- `client_status` - одно из значений: `pending`, `in_progress`, `rejected`, `hired`.
Ответ:
- `id` - hash_id отклика.
- `state.id` - итоговый статус (`pending`, `in_progress`, `rejected`, `hired`).
- `state.name` - человекочитаемое название статуса.
- `updated_at` - время последнего изменения статуса.

Правила для управления откликами:
- `candidate_id` — это hash_id отклика (берется из `GET /negotiations/...` или `GET /applications/{candidate_id}`);
- перевод в `hired` считается финальным: вернуть отклик из `hired` в другой статус нельзя;
- сброс в `pending` запрещен, если кандидат уже получил уведомление о решении;
- изменение статуса доступно только рекрутеру-владельцу вакансии (или администратору компании);

### 4.6. Профили кандидатов

1. `GET /profiles/get_profile/a/{hash_id}`
Назначение: получить профиль кандидата по ID отклика (application).

2. `GET /profiles/get_profile/dp/{hash_id}`
Назначение: получить профиль кандидата по ID кандидата из подборки (digest candidate).

### 4.7. Вебхуки откликов

1. `GET /webhooks/applications`
Назначение: получить текущий URL вебхука откликов для компании.
Ответ:
- `url` - текущий URL вебхука или `null`;
- `include_contacts` - должен ли webhook присылать отклик с контактами.

2. `PUT /webhooks/applications`
Назначение: установить/обновить URL вебхука откликов для компании.
Payload:
- `url` - валидный HTTP/HTTPS URL.
- `include_contacts` - boolean-флаг, должен ли webhook присылать отклики с раскрытыми контактами.
- `url: null` - отключить вебхук.
Ответ:
- `url` - текущий URL вебхука или `null`;
- `include_contacts` - текущее значение настройки раскрытия контактов для webhook.

Правила:
- у компании поддерживается один URL вебхука откликов;
- изменение затрагивает только компанию текущего авторизованного рекрутера.

#### 4.7.1. Исходящие события вебхука

После настройки URL getmatch отправляет `POST` с JSON payload на указанный endpoint.

Поддерживаемые `event`:
- `application_created` - новый отклик впервые отправлен работодателю;
- `application_status_changed` - изменилось состояние отклика.

Важно:
- webhook `application_created` отправляется только один раз на отклик;
- `application_status_changed` отправляется только при фактическом изменении состояния.

Поле `state` в payload:
- `id` - системный статус: `pending`, `in_progress`, `rejected`, `hired`;
- `name` - человекочитаемое название статуса.

Поле `application.id` — hash_id отклика (используется также в `GET /applications/{candidate_id}`).

Контакты и персональные данные в payload зависят от доступности контактов:
- если в настройке webhook указано `include_contacts=false`, `contact` будет пустым, `last_name` и `birth_date` будут `null`;
- если в настройке webhook указано `include_contacts=true`, webhook возвращает отклик с контактами независимо от текущего внутреннего статуса раскрытия.

## 5. Примеры использования: черновики вакансий

### 5.1. Создать черновик

```bash
curl --request POST \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/drafts" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{
    "payload": {
      "position": "Senior Python Developer",
      "location_requirements": [{"location_raw": "Белград"}],
      "salary_display_from": 5000,
      "salary_display_to": 7000,
      "salary_currency": "eur",
      "salary_taxes": "gross",
      "salary_is_total": false,
      "type": "web_and_tg",
      "language": "eng",
      "description": "Мы ищем Senior Python разработчика..."
    }
  }'
```

Пример ответа (фрагмент):
```json
{
  "id": 14567,
  "status": "filling",
  "errors": null,
  "recruiter_id": 101,
  "company_id": 202,
  "vacancy_id": null,
  "created_at": "2026-03-05T10:15:30.123456",
  "updated_at": "2026-03-05T10:15:30.123456"
}
```

### 5.2. Проверить статус черновика

```bash
curl --request GET \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/drafts/14567" \
  --header "Authorization: Bearer <access_token>"
```

Если черновик отклонен:
```json
{
  "id": 14567,
  "status": "rejected",
  "errors": [
    {
      "code": "required_field",
      "justification": "Добавьте описание вакансии",
      "field_name": "description"
    }
  ]
}
```

### 5.3. Исправить отклоненный черновик

```bash
curl --request PATCH \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/drafts/14567" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{
    "payload": {
      "description": "Полное описание вакансии с задачами и требованиями"
    }
  }'
```

После `PATCH` черновик снова проходит этапы `filling -> validating -> rejected|validated`.

### 5.4. Опубликовать валидный черновик

```bash
curl --request POST \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/drafts/14567/publish" \
  --header "Authorization: Bearer <access_token>"
```

Ответ:
```json
{"status":"queued"}
```

Дальше опрашивайте `GET /employers/{company_id}/vacancies/drafts/{draft_id}`:
- пока идет публикация: `status = "publishing"`;
- после успеха: `status = "accepted"` и заполнен `vacancy_id`.

### 5.5. Снять вакансию с публикации

```bash
curl --request POST \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/<vacancy_id>/archive" \
  --header "Authorization: Bearer <access_token>"
```

Ответ:
```json
{}
```

### 5.6. Настроить вебхук откликов

```bash
curl --request PUT \
  --url "https://getmatch.ru/api/integrations/v1/webhooks/applications" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{
    "url": "https://example.com/getmatch/webhooks/applications",
    "include_contacts": false
  }'
```

Ответ:
```json
{
  "url": "https://example.com/getmatch/webhooks/applications",
  "include_contacts": false
}
```

Отключить вебхук:
```bash
curl --request PUT \
  --url "https://getmatch.ru/api/integrations/v1/webhooks/applications" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{
    "url": null,
    "include_contacts": false
  }'
```

Получить текущую настройку:
```bash
curl --request GET \
  --url "https://getmatch.ru/api/integrations/v1/webhooks/applications" \
  --header "Authorization: Bearer <access_token>"
```

Ответ:
```json
{
  "url": "https://example.com/getmatch/webhooks/applications",
  "include_contacts": false
}
```

### 5.7. Пример payload входящего вебхука отклика без контактов

```json
{
  "event": "application_status_changed",
  "vacancy_hash_id": "k0N5LqA4",
  "state": {
    "id": "in_progress",
    "name": "В работе"
  },
  "application": {
    "id": "pQ2M8Zx1",
    "first_name": "Иван",
    "last_name": null,
    "age": 29,
    "birth_date": null,
    "area": {
      "name": "Белград"
    },
    "cover_letter": "Буду рад обсудить вакансию",
    "contact": [],
    "skill_set": [
      "Python",
      "FastAPI"
    ],
    "education": {
      "primary": [
        {
          "name": "СПбГУ",
          "organization": "Computer Science",
          "result": "Bachelor",
          "year": 2018
        }
      ]
    },
    "experience": [
      {
        "company": "Example LLC",
        "position": "Backend Developer",
        "description": "Разработка и поддержка API",
        "start": "2021-05",
        "end": "2024-02"
      }
    ],
    "created_at": "2025-11-10T09:15:00+0000",
    "updated_at": "2026-03-10T12:45:30+0000"
  }
}
```

### 5.8. Пример payload входящего вебхука отклика с контактами

```json
{
  "event": "application_status_changed",
  "vacancy_hash_id": "k0N5LqA4",
  "state": {
    "id": "in_progress",
    "name": "В работе"
  },
  "application": {
    "id": "pQ2M8Zx1",
    "first_name": "Иван",
    "last_name": "Иванов",
    "age": 29,
    "birth_date": "1996-04-12",
    "area": {
      "name": "Белград"
    },
    "cover_letter": "Буду рад обсудить вакансию",
    "contact": [
      {
        "contact_value": "candidate@example.com",
        "type": {
          "id": "email"
        }
      },
      {
        "contact_value": "@candidate",
        "type": {
          "id": "telegram"
        }
      }
    ],
    "skill_set": [
      "Python",
      "FastAPI"
    ],
    "education": {
      "primary": [
        {
          "name": "СПбГУ",
          "organization": "Computer Science",
          "result": "Bachelor",
          "year": 2018
        }
      ]
    },
    "experience": [
      {
        "company": "Example LLC",
        "position": "Backend Developer",
        "description": "Разработка и поддержка API",
        "start": "2021-05",
        "end": "2024-02"
      }
    ],
    "created_at": "2025-11-10T09:15:00+0000",
    "updated_at": "2026-03-10T12:45:30+0000"
  }
}
```

### 5.9. Получить отклик без контактов (поведение по умолчанию)

```bash
curl --request GET \
  --url "https://getmatch.ru/api/integrations/v1/applications/<candidate_id>" \
  --header "Authorization: Bearer <access_token>"
```

Пример ответа:
```json
{
  "id": "pQ2M8Zx1",
  "first_name": "Иван",
  "last_name": null,
  "birth_date": null,
  "contact": [],
  "cover_letter": "Буду рад обсудить вакансию"
}
```

### 5.10. Получить отклик с раскрытием контактов

```bash
curl --request GET \
  --url "https://getmatch.ru/api/integrations/v1/applications/<candidate_id>?open_contacts=true" \
  --header "Authorization: Bearer <access_token>"
```

Важно:
- этот запрос пытается раскрыть контакты;
- при успешном раскрытии может быть списан лимит раскрытия контактов.

### 5.11. Отклонить отклик с причиной

```bash
curl --request POST \
  --url "https://getmatch.ru/api/integrations/v1/applications/<candidate_id>" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{
    "resolution": "reject",
    "reason": "Недостаточно релевантного опыта",
    "forward_reason": true
  }'
```

Ответ:
```json
{
  "id": "pQ2M8Zx1",
  "state": {
    "id": "rejected",
    "name": "Отказ"
  },
  "updated_at": "2026-03-12T08:40:00+0000"
}
```

### 5.12. Обновить статус отклика на hired

```bash
curl --request PUT \
  --url "https://getmatch.ru/api/integrations/v1/applications/<candidate_id>" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{
    "client_status": "hired"
  }'
```

Ответ:
```json
{
  "id": "pQ2M8Zx1",
  "state": {
    "id": "hired",
    "name": "Нанят"
  },
  "updated_at": "2026-03-12T08:45:00+0000"
}
```

## 6. Базовый пример запроса

```bash
curl --request GET \
  --url "https://getmatch.ru/api/integrations/v1/me" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json"
```

## 7. Типовые коды ответов

- `204 No Content` - успешное удаление
- `200 OK` - успешный запрос
- `400 Bad Request` - некорректные параметры
- `401 Unauthorized` - отсутствует/некорректный/просроченный токен
- `402 Payment Required` - превышен лимит на раскрытие контактов
- `404 Not Found` - объект не найден или недоступен
- `405 Method Not Allowed` - операция недоступна для текущего рекрутера
- `409 Conflict` - конфликт статуса черновика (например, публикация невалидного черновика)
- `422 Unprocessable Entity` - ошибка схемы входных данных
- `429 Too Many Requests` - превышен дневной лимит API
