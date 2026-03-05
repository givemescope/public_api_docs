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
Важно: запрос может влиять на лимиты раскрытия контактов.

### 4.6. Профили кандидатов

1. `GET /profiles/get_profile/a/{hash_id}`
Назначение: получить профиль кандидата по ID отклика (application).

2. `GET /profiles/get_profile/dp/{hash_id}`
Назначение: получить профиль кандидата по ID кандидата из подборки (digest candidate).

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
- `409 Conflict` - конфликт статуса черновика (например, публикация невалидного черновика)
- `422 Unprocessable Entity` - ошибка схемы входных данных
- `429 Too Many Requests` - превышен дневной лимит API
