# Public API getmatch

Document for getmatch Public API users.

## 1. Basic Information

- Public API base URL: `https://getmatch.ru/api/integrations/v1`
- Data format: `application/json`

## 2. Authentication

Public API uses OAuth2 Bearer Token.

### 2.1. What You Need to Connect

- `client_id`
- `client_secret`
- `manage_token` (for redirect_uri configuration)
- `redirect_uri`

### 2.2. Authorization Header

Include the following header in all Public API requests (except OAuth requests):

```
Authorization: Bearer <access_token>
```

### 2.3. redirect_uri Configuration

Use the `PUT https://getmatch.ru/api/oauth/clients/{client_id}` endpoint to configure redirect URLs.
You must provide `manage_token` in the authorization header.

Request example:
```bash
curl -X PUT 'https://getmatch.ru/api/oauth/clients/{client_id}' -H 'Content-Type: application/json' -H 'Authorization: Bearer {manage_token}' --data-raw '{"redirect_uris": ["<http://localhost:8080>"]}'
```

Use GET to retrieve the current redirect_uris list:
```bash
curl -X GET 'https://getmatch.ru/api/oauth/clients/{client_id}' -H 'Content-Type: application/json' -H 'Authorization: Bearer {manage_token}'
```

### 2.4. Token Issuance and Refresh (OAuth2)

OAuth endpoints:

- `POST https://getmatch.ru/api/oauth/token` - exchange `authorization_code` for `access_token` and `refresh_token`
- `POST https://getmatch.ru/api/oauth/refresh` - refresh `access_token` using `refresh_token`

To receive an authorization_code, a user must sign in to getmatch using:
https://getmatch.ru/employer/oauth?redirect_uri={redirect_uri}&client_id={client_id}.
After successful sign-in, the user is redirected to the specified redirect_uri with an extra parameter: `code={authorization_code}`.

Request example for access_token:
```bash
curl -X POST 'https://getmatch.ru/api/oauth/token' -H 'Content-Type: application/json' --data-raw '{"grant_type": "authorization_code", "client_id": "{client_id}", "client_secret": "{client_secret}", "code": "{authorization_code}"}'
```

Request example for access_token refresh:
```bash
curl -X POST 'https://getmatch.ru/api/oauth/refresh' -H 'Content-Type: application/json' --data-raw '{"grant_type": "refresh_token", "refresh_token": "{refresh_token}"}'
```

Response example for access_token issuance/refresh:
```json
{"access_token": "{access_token}", "token_type": "bearer", "expires_at": "2026-01-01T09:30:00.012345", "refresh_token": "{refresh_token}"}
```

## 3. Full Specification (Swagger / OpenAPI)

- Swagger UI: [https://getmatch.ru/api/integrations/docs](https://getmatch.ru/api/integrations/docs)
- OpenAPI JSON: [https://getmatch.ru/api/integrations/openapi.json](https://getmatch.ru/api/integrations/openapi.json)

## 4. Available Endpoints

Below are the Public API endpoints (`/api/integrations/v1/...`).

### 4.1. General Information

1. `GET /me`
Purpose: get information about the currently authorized recruiter and their company.

### 4.2. API Limits

1. `GET /limits/get`
Purpose: get current and maximum company limits:
- `daily` - daily request limit.
- `contacts` - contact reveal limit.

### 4.3. Vacancies

1. `GET /vacancies/`
Purpose: get the list of active vacancies for the authorized company.

2. `GET /employers/{company_id}/vacancies/active?page=<int>&per_page=<int>`
Purpose: get active company vacancies with pagination. Inspired by HH API.
Parameters:
- `company_id` - company ID (can be obtained from `/me`).
- `page` - page number, starting from `0`.
- `per_page` - page size (`1..200`).

### 4.4. Vacancy Drafts

1. `POST /employers/{company_id}/vacancies/drafts`  
Purpose: create a vacancy draft and submit it for validation.

2. `GET /employers/{company_id}/vacancies/drafts`  
Purpose: get drafts of the current recruiter (sorted by `updated_at DESC`).  
Important: drafts with `accepted` status are not included in this list.

3. `GET /employers/{company_id}/vacancies/drafts/{draft_id}`  
Purpose: get a draft by ID.

4. `PATCH /employers/{company_id}/vacancies/drafts/{draft_id}`  
Purpose: partially update a draft and re-submit it for validation.

5. `POST /employers/{company_id}/vacancies/drafts/{draft_id}/publish`  
Purpose: queue a validated draft for publication.  
Successful response: `{"status": "queued"}`.

6. `DELETE /employers/{company_id}/vacancies/drafts/{draft_id}`  
Purpose: delete a draft (if it is not linked to a final vacancy yet).

#### 4.4.1. payload fields for create/update

Required for `POST`:
- `position` - vacancy title.
- `salary_display_from`, `salary_display_to` - salary range.
- `salary_taxes` - `net` or `gross`.
- `salary_is_total` - whether compensation is total.
- `type` - `only_web` or `web_and_tg`.

Optional fields:
- `location_requirements` - array of objects like `{"location_raw": "<location text>"}`.
- `salary_currency` - `rub` / `usd` / `eur` (`₽`, `$`, `€` are also accepted).
- `salary_hidden`, `cover_letter_required`, `cover_letter_placeholder`.
- `language`, `description`, `required_years_of_experience`.
- `location_validation`, `auto_prolong`.

For `PATCH`, send only the changed fields inside `payload` (partial update).

#### 4.4.2. Draft lifecycle

- `new` - technical initial status.
- `filling` - system auto-fills fields.
- `validating` - required fields and content checks are running.
- `rejected` - validation errors found (see `errors`).
- `validated` - draft is ready for publication.
- `publishing` - publication command is queued.
- `accepted` - final vacancy created, `vacancy_id` is set.

Rules:
- editing (`PATCH`) is allowed only for `new`, `rejected`, `validated`;
- publishing (`/publish`) is allowed only for `validated` and `vacancy_id = null`;
- deletion is forbidden if status is `accepted` or `vacancy_id` is already set.

#### 4.4.3. Validation errors

`errors` is an array of objects:
- `code` - error type (`required_field`, `max_length`, `content_policy`, `invalid_format`, `bad_words`, `other`);
- `justification` - human-readable message;
- `field_name` - field with the issue (for example, `description`, `location_requirements[0].format`).

### 4.5. Applications

1. `GET /negotiations?vacancy_id=<vacancy_id>`
Purpose: get vacancy application collections (status counters).

2. `GET /negotiations/{collection_name}?vacancy_id=<vacancy_id>&page=<int>&per_page=<int>`
Purpose: get the applications list for the selected collection with pagination.
Supported `collection_name` values:
- `all_applications`
- `in_progress_applications`
- `rejected_applications`
- `hired_applications`
- `pending_applications`

3. `GET /applications/{candidate_id}`
Purpose: get a candidate resume/profile from an application.
Important: this request may affect contact reveal limits.

4. `POST /applications/{candidate_id}`
Purpose: review an application (approve or reject).
Payload:
- `resolution` - required: `approve` or `reject`.
- `reason` - optional reject reason text.
- `forward_reason` - optional flag: whether to send `reason` to the candidate.
Response:
- `id` - application hash_id.
- `state.id` - resulting status (`pending`, `in_progress`, `rejected`, `hired`).
- `state.name` - human-readable status label.
- `updated_at` - timestamp of the latest status update.

5. `PUT /applications/{candidate_id}`
Purpose: update application client status.
Payload:
- `client_status` - one of: `pending`, `in_progress`, `rejected`, `hired`.
Response:
- `id` - application hash_id.
- `state.id` - resulting status (`pending`, `in_progress`, `rejected`, `hired`).
- `state.name` - human-readable status label.
- `updated_at` - timestamp of the latest status update.

Rules for application management:
- `candidate_id` is the application hash_id (from `GET /negotiations/...` or `GET /applications/{candidate_id}`);
- `hired` is a final status: it cannot be changed back to another status;
- reset to `pending` is forbidden if the candidate has already been notified about a resolution;
- status update is allowed only for the vacancy owner recruiter (or a company admin);

### 4.6. Candidate Profiles

1. `GET /profiles/get_profile/a/{hash_id}`
Purpose: get a candidate profile by application ID.

2. `GET /profiles/get_profile/dp/{hash_id}`
Purpose: get a candidate profile by digest candidate ID.

### 4.7. Application Webhooks

1. `GET /webhooks/applications`
Purpose: get the current applications webhook URL for the company.
Response: `{"url": "<webhook_url>|null"}`.

2. `PUT /webhooks/applications`
Purpose: set/update the applications webhook URL for the company.
Payload:
- `url` - a valid HTTP/HTTPS URL.
- `url: null` - disable webhooks.
Response: `{"url": "<webhook_url>|null"}`.

Rules:
- one applications webhook URL is supported per company;
- update affects only the current authorized recruiter's company.

#### 4.7.1. Outgoing webhook events

After URL is configured, getmatch sends `POST` requests with JSON payload to this endpoint.

Supported `event` values:
- `application_created` - a new application is sent to the employer for the first time;
- `application_status_changed` - application state has changed.

Important:
- `application_created` webhook is sent only once per application;
- `application_status_changed` is sent only on an actual state change.

`state` field in payload:
- `id` - system status: `pending`, `in_progress`, `rejected`, `hired`;
- `name` - human-readable status name (currently returned in Russian).

`application.id` is the application hash_id (the same ID used in `GET /applications/{candidate_id}`).

Contacts and personal data depend on contacts visibility:
- if contacts are not yet revealed, `contact` may be empty, and `last_name`/`birth_date` may be `null`.

## 5. Usage Examples: Vacancy Drafts

### 5.1. Create a draft

```bash
curl --request POST \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/drafts" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{
    "payload": {
      "position": "Senior Python Developer",
      "location_requirements": [{"location_raw": "Belgrade"}],
      "salary_display_from": 5000,
      "salary_display_to": 7000,
      "salary_currency": "eur",
      "salary_taxes": "gross",
      "salary_is_total": false,
      "type": "web_and_tg",
      "language": "eng",
      "description": "We are looking for a Senior Python engineer..."
    }
  }'
```

Response example (fragment):
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

### 5.2. Check draft status

```bash
curl --request GET \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/drafts/14567" \
  --header "Authorization: Bearer <access_token>"
```

If draft is rejected:
```json
{
  "id": 14567,
  "status": "rejected",
  "errors": [
    {
      "code": "required_field",
      "justification": "Add vacancy description",
      "field_name": "description"
    }
  ]
}
```
`justification` text may vary depending on validation rules and localization.

### 5.3. Fix a rejected draft

```bash
curl --request PATCH \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/drafts/14567" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{
    "payload": {
      "description": "Full vacancy description with responsibilities and requirements"
    }
  }'
```

After `PATCH`, the draft goes through `filling -> validating -> rejected|validated` again.

### 5.4. Publish a validated draft

```bash
curl --request POST \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/drafts/14567/publish" \
  --header "Authorization: Bearer <access_token>"
```

Response:
```json
{"status":"queued"}
```

Then poll `GET /employers/{company_id}/vacancies/drafts/{draft_id}`:
- while in progress: `status = "publishing"`;
- when done: `status = "accepted"` and `vacancy_id` is filled.

### 5.5. Configure applications webhook

```bash
curl --request PUT \
  --url "https://getmatch.ru/api/integrations/v1/webhooks/applications" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{
    "url": "https://example.com/getmatch/webhooks/applications"
  }'
```

Response:
```json
{"url":"https://example.com/getmatch/webhooks/applications"}
```

Disable webhook:
```bash
curl --request PUT \
  --url "https://getmatch.ru/api/integrations/v1/webhooks/applications" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{
    "url": null
  }'
```

### 5.6. Incoming application webhook payload example

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
    "first_name": "Ivan",
    "last_name": "Ivanov",
    "age": 29,
    "birth_date": "1996-04-12",
    "area": {
      "name": "Belgrade"
    },
    "cover_letter": "Happy to discuss this role",
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
          "name": "SPbU",
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
        "description": "API development and support",
        "start": "2021-05",
        "end": "2024-02"
      }
    ],
    "created_at": "2025-11-10T09:15:00+0000",
    "updated_at": "2026-03-10T12:45:30+0000"
  }
}
```

### 5.7. Reject an application with reason

```bash
curl --request POST \
  --url "https://getmatch.ru/api/integrations/v1/applications/<candidate_id>" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{
    "resolution": "reject",
    "reason": "Insufficient relevant experience",
    "forward_reason": true
  }'
```

Response:
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

### 5.8. Set application status to hired

```bash
curl --request PUT \
  --url "https://getmatch.ru/api/integrations/v1/applications/<candidate_id>" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{
    "client_status": "hired"
  }'
```

Response:
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

## 6. Basic Request Example

```bash
curl --request GET \
  --url "https://getmatch.ru/api/integrations/v1/me" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json"
```

## 7. Common Response Codes

- `204 No Content` - successful deletion
- `200 OK` - successful request
- `400 Bad Request` - invalid parameters
- `401 Unauthorized` - missing/invalid/expired token
- `402 Payment Required` - contact reveal limit exceeded
- `404 Not Found` - object not found or unavailable
- `405 Method Not Allowed` - operation is not allowed for the current recruiter
- `409 Conflict` - draft status conflict (for example, publishing a non-validated draft)
- `422 Unprocessable Entity` - request payload schema error
- `429 Too Many Requests` - daily API limit exceeded
