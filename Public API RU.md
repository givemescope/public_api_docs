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

`client_id`, `client_secret` и `manage_token` нужно получить через своего менеджера в getmatch.

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

Оба эндпоинта принимают тело как в `application/json`, так и в `application/x-www-form-urlencoded` или `multipart/form-data`.

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

Спецификация генерируется автоматически из кода, в ней описаны все эндпоинты и схемы ответов.
В Swagger UI настроена авторизация (`OAuth2AuthorizationCodeBearer`), поэтому запросы можно
выполнять прямо из браузера по кнопке Authorize.

## 4. Доступные эндпоинты

Ниже перечислены эндпоинты Public API (`/api/integrations/v1/...`).

### 4.1. Общая информация

1. `GET /me`
Назначение: получить информацию о текущем авторизованном рекрутере и его компании.
Ответ: `auth_type`, `id`, `email`, `first_name`, `last_name`, `employer { id, name }`.

2. `GET /employers/{company_id}/recruiters?q=<query>&limit=<int>`
Назначение: найти рекрутеров текущей компании по имени, фамилии или email.
Параметры:
- `company_id` - ID компании (можно получить в `/me`).
- `q` - поисковая строка, поиск выполняется по `first_name`, `last_name`, `email`.
  Строка разбивается на слова, каждое слово должно совпасть хотя бы с одним из полей.
- `limit` - максимальное число результатов (`1..100`, по умолчанию `20`).
Ответ:
- массив рекрутеров с полями `id`, `first_name`, `last_name`, `email`.

### 4.2. Лимиты и квоты

Отдельного эндпоинта для чтения лимитов (`GET /limits/get`) в Public API **нет**.

Ограничений два вида, и они возвращают **разные коды**:

| Тип | Что это | Код | Что делать клиенту |
| --- | --- | --- | --- |
| Частотный лимит | защита от вычитывания базы; счетчики день/месяц, сбрасываются сами | `429` | подождать и повторить после `reset_at` |
| Оплаченная квота | закончился купленный пакет (контакты, публикации вакансий) | `402` | повторять бесполезно, нужна докупка |

#### 4.2.1. Частотные лимиты (429)

Считаются одновременно по рекрутеру и по компании, в дневном и месячном окне.

| Действие | Что его расходует |
| --- | --- |
| `search` | каждый вызов `GET /candidates` |
| `profile_view` | каждый вызов `GET /profiles/get_profile/dp/{hash_id}` |
| `open_contacts` | первое раскрытие контактов конкретного кандидата из общего пула |

Текущие квоты:

| Действие | День, рекрутер | День, компания | Месяц, рекрутер | Месяц, компания |
| --- | --- | --- | --- | --- |
| `search` | 100 | 300 | 500 | 3000 |
| `profile_view` | 300 | 900 | 1500 | 9000 |
| `open_contacts` | 300 | 900 | 1500 | 9000 |

Поверх суточных квот действует бёрст-лимит на поиск: не более 20 запросов в минуту на рекрутера
и 60 в минуту на компанию. При его срабатывании возвращается `429` с
`detail.code = "rate_limit_exceeded"` и заголовком `Retry-After` (в секундах).
Суточная квота при этом **не** расходуется - отклоненный по бёрст-лимиту запрос можно повторить.

Пример `429` по бёрст-лимиту:
```json
{
  "detail": {
    "code": "rate_limit_exceeded",
    "message": "Too many search requests in a short period. Slow down and retry later."
  }
}
```

Пример `429` по исчерпанной суточной квоте:
```json
{
  "detail": {
    "code": "digest_limit_exceeded",
    "message": "Вы исчерпали дневной лимит на поиск кандидатов (100 из 100). Доступ восстановится 19.08 в 03:00 МСК.",
    "scope": "recruiter",
    "period": "day",
    "action": "search",
    "limit": 100,
    "used": 100,
    "reset_at": "2026-08-18T21:00:00+00:00"
  }
}
```

#### 4.2.2. Оплаченные квоты (402)

- закончился пакет открытий контактов - `GET /profiles/get_profile/dp/{hash_id}` вернет `402`
  с `detail.code = "contacts_quota_exhausted"`;
- закончились доступные публикации вакансий - `POST /employers/{company_id}/vacancies/drafts/{draft_id}/publish`
  вернет `402` с текстовым `detail` (`Not enough available vacancy publications`).

Пример `402` на открытии контактов:
```json
{
  "detail": {
    "code": "contacts_quota_exhausted",
    "message": "Contact opening quota is exhausted. Contact your getmatch manager to buy more contacts."
  }
}
```

Важно:
- отклики на собственные вакансии компании (`/negotiations`, `/applications/...`,
  `/profiles/get_profile/a/...`) не расходуют ни частотные лимиты, ни оплаченные квоты;
- ранее исчерпание пакета контактов тоже возвращало `429`. Если ваш клиент ретраит `429`
  с бэкоффом, добавьте отдельную обработку `402`: повтор такого запроса не поможет.

### 4.3. Поиск кандидатов

1. `GET /candidates`
Назначение: искать кандидатов, опубликованных в подборках getmatch (общий пул).
Каждый вызов расходует одну единицу лимита `search` и учитывается в бёрст-лимите (см. 4.2.1).

Карточки возвращаются анонимными: контакты, имя и фото скрыты до тех пор, пока контакты
кандидата не будут открыты через `GET /profiles/get_profile/dp/{id}`.

Параметры:

| Параметр | Тип | Описание |
| --- | --- | --- |
| `q` | string | полнотекстовый запрос по профилям |
| `page` | int | номер страницы, с `0` |
| `per_page` | int | размер страницы (`1..100`, по умолчанию `20`) |
| `sorting` | string | порядок сортировки выдачи |
| `skills` | string | слаги навыков через запятую |
| `locations` | list | локации (значения справочника локаций / городов) |
| `region_statuses` | list | статусы регионов |
| `language` | string | требуемый уровень английского (и выше) |
| `position` | string | должность в любом месте опыта |
| `last_position` | string | должность на последнем месте работы |
| `seniority` | list | `junior`, `middle`, `senior`, `lead`, `c_level` |
| `salary_from` / `salary_to` | int | вилка зарплатных ожиданий |
| `salary_presence` | bool | только с указанными ожиданиями |
| `yoe_from` / `yoe_to` | int | суммарный опыт в годах |
| `age_from` / `age_to` | int | возраст (`0..120`) |
| `manager_experience` | bool | есть управленческий опыт |
| `is_not_job_hopper` | bool | отсеивать частую смену работы |
| `trusted_only` | bool | только кандидаты со статусом `trusted` или `hire_confirmed` |

Ответ:
- `found` - всего найдено;
- `page`, `pages`, `per_page` - пагинация;
- `items` - карточки кандидатов.

Поля карточки:
- `id` - идентификатор для `GET /profiles/get_profile/dp/{id}`;
- `hash_type` - всегда `dp`;
- `published_at` - дата публикации в подборке;
- `specialization_slug`, `specialization_name`;
- `contacts_opened` - `true`, если компания уже оплатила контакты этого кандидата;
- `profile` - анонимная карточка профиля:
  - `name` - `null`, пока контакты не открыты;
  - `photo_url`, `age`, `city`, `country`;
  - `salary_expectations_from`, `salary_expectations_to`, `salary_expectations_currency`;
  - `skills` - список названий, `skills_detailed` - объекты `{name, slug}`;
  - `languages` - словарь `язык -> уровень`;
  - `positions` - опыт работы, `educations` - образование;
  - `trusted` - проверенный кандидат.

Контактов (email, телефон, telegram) в выдаче поиска нет ни в каком виде.

### 4.4. Профили кандидатов

1. `GET /profiles/get_profile/a/{hash_id}`
Назначение: получить профиль кандидата по ID отклика (application).
Правила:
- вызов одобряет отклик, раскрывает контакты и уведомляет кандидата;
- лимиты и квоты подборок не расходуются.

2. `GET /profiles/get_profile/dp/{hash_id}`
Назначение: получить профиль кандидата по ID кандидата из общего пула
(digest candidate, поле `id` в `GET /candidates`).
Правила:
- расходует частотный лимит `profile_view`;
- если контакты этого кандидата компанией еще не открывались, дополнительно расходуется
  частотный лимит `open_contacts` и один контакт из оплаченного пакета;
- при исчерпании частотного лимита - `429` (повторить после `reset_at`);
- при исчерпании оплаченного пакета контактов - `402` (`contacts_quota_exhausted`).

Поля ответа обеих ручек:
`hash_id`, `hash_type`, `name`, `photo_url`, `birthdate`, `general_info`, `links`, `contacts`,
`locations`, `educations`, `languages`, `positions`, `skills`, `skills_detailed`.

### 4.5. Вакансии

Во всех ручках `vacancy_id` - числовой ID вакансии (например `14567`), он же приходит в
`GET /vacancies/` и `GET /employers/{company_id}/vacancies/active`.

1. `GET /vacancies/`
Назначение: получить список активных вакансий авторизованной компании.
Ответ: массив объектов `{id, created_at, name}`.

2. `GET /employers/{company_id}/vacancies/active?page=<int>&per_page=<int>`
Назначение: получить активные вакансии компании с пагинацией. Вдохновлено HH API.
Параметры:
- `company_id` - ID компании (можно получить в `/me`).
- `page` - номер страницы, начиная с `0`.
- `per_page` - размер страницы (`1..200`, по умолчанию `20`).

3. `POST /employers/{company_id}/vacancies/{vacancy_id}/archive`
Назначение: снять вакансию с публикации.
Успешный ответ: `{}`.

4. `POST /employers/{company_id}/vacancies/{vacancy_id}/prolong`
Назначение: продлить публикацию вакансии.
Правила:
- продление доступно, только когда до `archive_date` осталось меньше 7 дней,
  иначе `403` (`Too early to prolong vacancy`);
- если у компании не осталось доступных публикаций, запрос не падает: возвращается
  `auto_prolonged = false`, а менеджеру getmatch уходит уведомление.
Ответ:
```json
{
  "auto_prolonged": true,
  "vacancy": {
    "id": "14567",
    "created_at": "2026-03-05T10:15:30+0000",
    "name": "Senior Python Developer",
    "published_at": "2026-03-06T09:00:00+0000",
    "archive_date": "2026-04-06",
    "archived_at": null,
    "is_active": true
  }
}
```

5. `POST /employers/{company_id}/vacancies/{vacancy_id}/boost`
Назначение: поднять вакансию и разослать ее подходящим кандидатам.
Payload:
- `include_viewed` - опциональный boolean, по умолчанию `true`: рассылать ли тем,
  кто уже видел вакансию.
Правила:
- вакансия должна быть активной и не в архиве, иначе `400` (`Vacancy boost unavailable`);
- между поднятиями действует кулдаун 3 дня, иначе `400` (`Boost cooldown not passed`);
- если у компании не осталось доступных публикаций, возвращается `boosted = false`
  и `candidates_sent = 0`, а менеджеру getmatch уходит уведомление.
Ответ:
```json
{
  "boosted": true,
  "vacancy": {"id": "14567", "name": "Senior Python Developer", "is_active": true},
  "candidates_sent": 128,
  "include_viewed": true
}
```

### 4.6. Черновики вакансий

1. `POST /employers/{company_id}/vacancies/drafts`
Назначение: создать черновик вакансии и отправить его на валидацию.
Дополнительно: можно передать `recruiter_id`, чтобы сразу назначить черновик на другого рекрутера этой же компании.

2. `GET /employers/{company_id}/vacancies/drafts`
Назначение: получить список черновиков компании (отсортированы по `updated_at DESC`).
Важно: черновики в статусе `accepted` в этот список не попадают.

3. `GET /employers/{company_id}/vacancies/drafts/{draft_id}`
Назначение: получить один черновик компании по ID.

4. `PATCH /employers/{company_id}/vacancies/drafts/{draft_id}`
Назначение: частично обновить черновик, при необходимости сменить `recruiter_id`, и повторно отправить его на валидацию.

5. `POST /employers/{company_id}/vacancies/drafts/{draft_id}/publish`
Назначение: поставить публикацию валидного черновика в очередь.
Успешный ответ: `{"status": "queued"}`.

6. `DELETE /employers/{company_id}/vacancies/drafts/{draft_id}`
Назначение: удалить черновик (если он еще не связан с финальной вакансией).
Успешный ответ: `204 No Content`.

#### 4.6.1. Поля payload для create/update

Дополнительное поле верхнего уровня для `POST` и `PATCH`:
- `recruiter_id` - опциональный ID рекрутера внутри текущей компании. Если не передан:
  - в `POST` черновик назначается на текущего авторизованного рекрутера;
  - в `PATCH` сохраняется текущий `recruiter_id` черновика.

Для `POST` обязательны:
- `position` - название вакансии.
- `salary_display_from`, `salary_display_to` - границы вилки.
- `salary_currency` - `rub` / `usd` / `eur` (также принимаются `₽`, `$`, `€`).
- `salary_taxes` - `net` или `gross`.
- `salary_is_total` - совокупная ли компенсация.
- `language` - `ru` или `eng`.
- `type` - `only_web` или `web_and_tg`.

Опциональные поля:

| Поле | Значения | Описание |
| --- | --- | --- |
| `location_requirements` | список `{"location_raw": "<текст локации>"}` | требования к локации |
| `work_format` | `office`, `hybrid`, `remote`, `relocation_company`, `relocation_candidate` | формат работы, применяется ко всем элементам `location_requirements` |
| `description` | строка `1000..30000` символов | описание вакансии, допускается HTML |
| `seniority` | `junior`, `middle`, `senior`, `lead`, `c_level` | уровень |
| `english_level` | `a1`, `a2`, `b1`, `b2`, `c` (`c1` и `c2` принимаются как `c`) | требуемый английский |
| `salary_hidden` | bool | скрывать ли вилку |
| `salary_hidden_variant` | `approximate_interval`, `interval`, `top_hidden`, `full_hidden` | как именно скрывать вилку |
| `incognito_publication` | bool | скрыть название компании от кандидатов |
| `cover_letter_required` | bool | обязательно ли сопроводительное письмо |
| `cover_letter_placeholder` | строка | подсказка для сопроводительного письма |
| `required_years_of_experience` | int | требуемый опыт в годах |
| `location_validation` | bool | проверять локацию кандидата |
| `auto_prolong` | bool | автопродление публикации |

Важно:
- схема payload строгая: неизвестное поле приводит к `422`;
- минимальная длина `description` - 1000 символов, это частая причина `422`;
- для `PATCH` передается только `payload` с изменяемыми полями (частичное обновление).

#### 4.6.2. Поля ответа

`id`, `status`, `errors`, `recruiter_id`, `recruiter_hash_id`, `company_id`, `vacancy_id`,
`can_publish`, `payload`, `created_at`, `updated_at`.

Поле `can_publish` показывает, хватит ли у компании доступных публикаций, чтобы
опубликовать этот черновик.

#### 4.6.3. Жизненный цикл черновика

- `new` - технический начальный статус.
- `filling` - система автодополняет поля.
- `validating` - выполняется проверка обязательных полей и контента.
- `rejected` - есть ошибки валидации (подробности в `errors`).
- `validated` - черновик готов к публикации.
- `publishing` - публикация поставлена в очередь.
- `accepted` - создана финальная вакансия, в `vacancy_id` записан ID вакансии.

Правила:
- редактирование (`PATCH`) разрешено только для `new`, `rejected`, `validated`, иначе `409`;
- публикация (`/publish`) разрешена только для `validated` и при `vacancy_id = null`, иначе `409`;
- если у компании не осталось доступных публикаций, `/publish` вернет `402`;
- удаление запрещено, если статус `accepted` или уже есть `vacancy_id`, иначе `409`;
- все операции с черновиками доступны любому рекрутеру текущей компании;
- `recruiter_id` можно менять только на активного рекрутера той же компании.

#### 4.6.4. Ошибки валидации

В `errors` возвращается массив объектов:
- `code` - тип ошибки (`required_field`, `max_length`, `content_policy`, `invalid_format`, `bad_words`, `other`);
- `justification` - человекочитаемое объяснение;
- `field_name` - поле с ошибкой (например, `description`, `location_requirements[0].format`).

### 4.7. Отклики

1. `GET /negotiations?vacancy_id=<vacancy_id>`
Назначение: получить коллекции откликов по вакансии (счетчики по статусам).

2. `GET /negotiations/{collection_name}?vacancy_id=<vacancy_id>&page=<int>&per_page=<int>`
Назначение: получить список откликов в выбранной коллекции с пагинацией
(`per_page`: `1..200`, по умолчанию `20`).
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
- если передан `open_contacts=true`, API попытается раскрыть контакты и уведомить кандидата.

Поля ответа:
`id`, `first_name`, `last_name`, `age`, `birth_date`, `area {name}`, `cover_letter`, `contact`,
`photo_url`, `skill_set`, `education {primary[]}`, `experience[]`, `created_at`, `updated_at`.

Важно:
- `created_at` - дата отклика (когда отклик ушел работодателю), а не дата создания профиля;
- `updated_at` - дата последнего ручного обновления профиля кандидатом.

Возможные значения `contact[].type.id`:
- `phone` - телефон;
- `email` - email;
- `telegram` - Telegram username;
- `other` - другая ссылка из профиля кандидата, например GitHub или личный сайт.

4. `POST /applications/{candidate_id}`
Назначение: рассмотреть отклик (одобрить или отклонить).
Payload:
- `resolution` - обязательное поле: `approve` или `reject`.
- `reason` - опциональный текст причины отказа.
- `forward_reason` - опциональный флаг, пересылать ли `reason` кандидату.
Ответ:
- `id` - hash_id отклика;
- `state.id` - итоговый статус (`pending`, `in_progress`, `rejected`, `hired`);
- `state.name` - человекочитаемое название статуса;
- `updated_at` - время последнего изменения статуса.

5. `PUT /applications/{candidate_id}`
Назначение: обновить клиентский статус отклика.
Payload:
- `client_status` - одно из значений: `pending`, `in_progress`, `rejected`, `hired`.
Ответ: тот же, что и у `POST`.

Правила для управления откликами:
- `candidate_id` - это hash_id отклика (берется из `GET /negotiations/...`);
- перевод в `hired` считается финальным: вернуть отклик из `hired` в другой статус нельзя (`400`);
- сброс в `pending` запрещен, если кандидат уже получил уведомление о решении (`409`);
- отклик, отклоненный автоматически, повторно рассмотреть нельзя (`409`);
- изменение статуса доступно только рекрутеру-владельцу вакансии или администратору компании,
  иначе `405`.

### 4.8. Вебхуки откликов

1. `GET /webhooks/applications`
Назначение: получить текущий URL вебхука откликов для компании.
Ответ:
- `url` - текущий URL вебхука или `null`;
- `include_contacts` - должен ли webhook присылать отклик с контактами.

2. `PUT /webhooks/applications`
Назначение: установить/обновить URL вебхука откликов для компании.
Payload:
- `url` - валидный HTTP/HTTPS URL, либо `null`, чтобы отключить вебхук;
- `include_contacts` - boolean-флаг, должен ли webhook присылать отклики с раскрытыми контактами.
Ответ: текущие `url` и `include_contacts`.

Правила:
- у компании поддерживается один URL вебхука откликов;
- изменение затрагивает только компанию текущего авторизованного рекрутера.

#### 4.8.1. Исходящие события вебхука

После настройки URL getmatch отправляет `POST` с JSON payload на указанный endpoint.

Поддерживаемые `event`:
- `application_created` - новый отклик впервые отправлен работодателю;
- `application_status_changed` - изменилось состояние отклика.

Важно:
- webhook `application_created` отправляется только один раз на отклик;
- `application_status_changed` отправляется только при фактическом изменении состояния.

Поля payload:
- `event` - тип события;
- `vacancy_id` - числовой ID вакансии;
- `state.id` - системный статус: `pending`, `in_progress`, `rejected`, `hired`;
- `state.name` - человекочитаемое название статуса (сейчас на русском);
- `application` - тот же объект, что возвращает `GET /applications/{candidate_id}`,
  плюс поле `profile_created_at` (дата создания профиля кандидата).

Поле `application.id` - hash_id отклика (используется также в `GET /applications/{candidate_id}`).

Контакты и персональные данные в payload зависят от настройки вебхука:
- при `include_contacts=false` поле `contact` будет пустым, `last_name` и `birth_date` - `null`;
- при `include_contacts=true` webhook возвращает отклик с контактами независимо от текущего
  внутреннего статуса раскрытия.

## 5. Примеры использования

### 5.1. Поиск кандидатов

```bash
curl --request GET \
  --url "https://getmatch.ru/api/integrations/v1/candidates?q=python&seniority=senior&per_page=20" \
  --header "Authorization: Bearer <access_token>"
```

Пример ответа (фрагмент):
```json
{
  "found": 254,
  "page": 0,
  "pages": 13,
  "per_page": 20,
  "items": [
    {
      "id": "pQ2M8Zx1",
      "hash_type": "dp",
      "published_at": "2026-08-10T09:15:00+0000",
      "specialization_slug": "backend",
      "specialization_name": "Backend-разработчик",
      "contacts_opened": false,
      "profile": {
        "name": null,
        "age": 29,
        "city": "Белград",
        "country": "Сербия",
        "salary_expectations_from": 5000,
        "salary_expectations_to": 7000,
        "salary_expectations_currency": "eur",
        "skills": ["Python", "FastAPI"],
        "trusted": true
      }
    }
  ]
}
```

### 5.2. Открыть контакты найденного кандидата

```bash
curl --request GET \
  --url "https://getmatch.ru/api/integrations/v1/profiles/get_profile/dp/pQ2M8Zx1" \
  --header "Authorization: Bearer <access_token>"
```

После успешного вызова тот же кандидат в поиске вернется с `contacts_opened: true`
и заполненным `profile.name`.

### 5.3. Создать черновик

```bash
curl --request POST \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/drafts" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{
    "recruiter_id": 101,
    "payload": {
      "position": "Senior Python Developer",
      "location_requirements": [{"location_raw": "Белград"}],
      "work_format": "hybrid",
      "salary_display_from": 5000,
      "salary_display_to": 7000,
      "salary_currency": "eur",
      "salary_taxes": "gross",
      "salary_is_total": false,
      "type": "web_and_tg",
      "language": "eng",
      "seniority": "senior",
      "english_level": "b2",
      "description": "<p>Мы ищем Senior Python разработчика...</p>"
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
  "can_publish": true,
  "created_at": "2026-03-05T10:15:30.123456",
  "updated_at": "2026-03-05T10:15:30.123456"
}
```

### 5.4. Проверить статус черновика

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

### 5.5. Исправить отклоненный черновик

```bash
curl --request PATCH \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/drafts/14567" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{
    "recruiter_id": 115,
    "payload": {
      "description": "Полное описание вакансии с задачами и требованиями"
    }
  }'
```

После `PATCH` черновик снова проходит этапы `filling -> validating -> rejected|validated`.

### 5.6. Опубликовать валидный черновик

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

### 5.7. Снять вакансию с публикации

```bash
curl --request POST \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/<vacancy_id>/archive" \
  --header "Authorization: Bearer <access_token>"
```

Ответ:
```json
{}
```

### 5.8. Продлить и поднять вакансию

```bash
curl --request POST \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/<vacancy_id>/prolong" \
  --header "Authorization: Bearer <access_token>"
```

```bash
curl --request POST \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/<vacancy_id>/boost" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{"include_viewed": false}'
```

### 5.9. Настроить вебхук откликов

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

Отключить вебхук:
```bash
curl --request PUT \
  --url "https://getmatch.ru/api/integrations/v1/webhooks/applications" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{"url": null, "include_contacts": false}'
```

### 5.10. Пример payload входящего вебхука отклика без контактов

```json
{
  "event": "application_status_changed",
  "vacancy_id": 14567,
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
    "area": {"name": "Белград"},
    "cover_letter": "Буду рад обсудить вакансию",
    "contact": [],
    "photo_url": "https://getmatch.ru/uploads/u/folder/avatar.jpg",
    "skill_set": ["Python", "FastAPI"],
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
    "created_at": "2026-03-10T12:45:30+0000",
    "profile_created_at": "2025-11-10T09:15:00+0000",
    "updated_at": "2026-03-10T12:45:30+0000"
  }
}
```

### 5.11. Пример payload входящего вебхука отклика с контактами

Отличается только персональными полями:
```json
{
  "application": {
    "last_name": "Иванов",
    "birth_date": "1996-04-12",
    "contact": [
      {"contact_value": "candidate@example.com", "type": {"id": "email"}},
      {"contact_value": "@candidate", "type": {"id": "telegram"}}
    ]
  }
}
```

### 5.12. Получить отклик без контактов (поведение по умолчанию)

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

### 5.13. Получить отклик с раскрытием контактов

```bash
curl --request GET \
  --url "https://getmatch.ru/api/integrations/v1/applications/<candidate_id>?open_contacts=true" \
  --header "Authorization: Bearer <access_token>"
```

### 5.14. Отклонить отклик с причиной

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
  "state": {"id": "rejected", "name": "Отказ"},
  "updated_at": "2026-03-12T08:40:00+0000"
}
```

### 5.15. Обновить статус отклика на hired

```bash
curl --request PUT \
  --url "https://getmatch.ru/api/integrations/v1/applications/<candidate_id>" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{"client_status": "hired"}'
```

Ответ:
```json
{
  "id": "pQ2M8Zx1",
  "state": {"id": "hired", "name": "Нанят"},
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

- `200 OK` - успешный запрос
- `204 No Content` - успешное удаление черновика
- `400 Bad Request` - некорректные параметры или недопустимый переход статуса отклика
- `401 Unauthorized` - отсутствует/некорректный/просроченный токен
- `402 Payment Required` - закончилась оплаченная квота: пакет открытий контактов
  (`detail.code = "contacts_quota_exhausted"`) или доступные публикации вакансий (`/publish`).
  Повторять запрос бесполезно - нужна докупка
- `403 Forbidden` - операция сейчас недоступна (например, слишком рано продлевать вакансию)
- `404 Not Found` - объект не найден или недоступен
- `405 Method Not Allowed` - операция недоступна для текущего рекрутера
- `409 Conflict` - конфликт состояния (статус черновика, повторное решение по отклику)
- `422 Unprocessable Entity` - ошибка схемы входных данных
- `429 Too Many Requests` - сработал частотный лимит. Два варианта, различаются по `detail.code`:
  `rate_limit_exceeded` - слишком много запросов в минуту, повтор через `Retry-After`;
  `digest_limit_exceeded` - исчерпана суточная или месячная квота (`search` / `profile_view` /
  `open_contacts`), повтор после `reset_at` (см. 4.2)
