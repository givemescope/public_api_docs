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

### 4.4. Отклики

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

### 4.5. Профили кандидатов

1. `GET /profiles/get_profile/a/{hash_id}`
Назначение: получить профиль кандидата по ID отклика (application).

2. `GET /profiles/get_profile/dp/{hash_id}`
Назначение: получить профиль кандидата по ID кандидата из подборки (digest candidate).

## 5. Пример запроса

```bash
curl --request GET \
  --url "https://getmatch.ru/api/integrations/v1/me" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json"
```

## 6. Типовые коды ответов

- `200 OK` - успешный запрос
- `400 Bad Request` - некорректные параметры
- `401 Unauthorized` - отсутствует/некорректный/просроченный токен
- `402 Payment Required` - превышен лимит на раскрытие контактов
- `404 Not Found` - объект не найден или недоступен
- `429 Too Many Requests` - превышен дневной лимит API
