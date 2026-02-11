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

### 4.4. Applications

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

### 4.5. Candidate Profiles

1. `GET /profiles/get_profile/a/{hash_id}`
Purpose: get a candidate profile by application ID.

2. `GET /profiles/get_profile/dp/{hash_id}`
Purpose: get a candidate profile by digest candidate ID.

## 5. Request Example

```bash
curl --request GET \
  --url "https://getmatch.ru/api/integrations/v1/me" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json"
```

## 6. Common Response Codes

- `200 OK` - successful request
- `400 Bad Request` - invalid parameters
- `401 Unauthorized` - missing/invalid/expired token
- `402 Payment Required` - contact reveal limit exceeded
- `404 Not Found` - object not found or unavailable
- `429 Too Many Requests` - daily API limit exceeded
