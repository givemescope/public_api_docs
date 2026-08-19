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

You need to obtain `client_id`, `client_secret`, and `manage_token` through your getmatch manager.

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

Both endpoints accept the body as `application/json`, `application/x-www-form-urlencoded`, or `multipart/form-data`.

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

The specification is generated from the code automatically and covers every endpoint and response
schema. Swagger UI has authentication configured (`OAuth2AuthorizationCodeBearer`), so requests can
be executed straight from the browser via the Authorize button.

## 4. Available Endpoints

Below are the Public API endpoints (`/api/integrations/v1/...`).

### 4.1. General Information

1. `GET /me`
Purpose: get information about the currently authorized recruiter and their company.
Response: `auth_type`, `id`, `email`, `first_name`, `last_name`, `employer { id, name }`.

2. `GET /employers/{company_id}/recruiters?q=<query>&limit=<int>`
Purpose: search recruiters of the current company by first name, last name, or email.
Parameters:
- `company_id` - company ID (can be obtained from `/me`).
- `q` - search string, matched against `first_name`, `last_name`, and `email`.
  The string is split into words, and every word must match at least one of the fields.
- `limit` - maximum number of results (`1..100`, default `20`).
Response:
- an array of recruiters with fields `id`, `first_name`, `last_name`, `email`.

### 4.2. Limits and Quotas

There is **no** separate endpoint for reading limits (`GET /limits/get`).

There are two kinds of restrictions, and they return **different codes**:

| Kind | What it is | Code | What the client should do |
| --- | --- | --- | --- |
| Rate limit | scraping protection; daily/monthly counters that reset on their own | `429` | wait and retry after `reset_at` |
| Paid quota | the purchased package is used up (contacts, vacancy publications) | `402` | retrying will not help, more has to be purchased |

#### 4.2.1. Rate limits (429)

Counted per recruiter and per company at the same time, in a daily and a monthly window.

| Action | What consumes it |
| --- | --- |
| `search` | every `GET /candidates` call |
| `profile_view` | every `GET /profiles/get_profile/dp/{hash_id}` call |
| `open_contacts` | first reveal of a specific pool candidate's contacts |

Current quotas:

| Action | Day, recruiter | Day, company | Month, recruiter | Month, company |
| --- | --- | --- | --- | --- |
| `search` | 100 | 300 | 500 | 3000 |
| `profile_view` | 300 | 900 | 1500 | 9000 |
| `open_contacts` | 300 | 900 | 1500 | 9000 |

On top of the daily quotas there is a search burst limit: at most 20 requests per minute per
recruiter and 60 per minute per company. When it fires, the API returns `429` with
`detail.code = "rate_limit_exceeded"` and a `Retry-After` header (in seconds).
The daily quota is **not** consumed in that case - a burst-rejected request can be retried.

`429` example for the burst limit:
```json
{
  "detail": {
    "code": "rate_limit_exceeded",
    "message": "Too many search requests in a short period. Slow down and retry later."
  }
}
```

`429` example for an exhausted daily quota:
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

#### 4.2.2. Paid quotas (402)

- the contact opening package is used up - `GET /profiles/get_profile/dp/{hash_id}` returns `402`
  with `detail.code = "contacts_quota_exhausted"`;
- no vacancy publications left - `POST /employers/{company_id}/vacancies/drafts/{draft_id}/publish`
  returns `402` with a plain-text `detail` (`Not enough available vacancy publications`).

`402` example for contact opening:
```json
{
  "detail": {
    "code": "contacts_quota_exhausted",
    "message": "Contact opening quota is exhausted. Contact your getmatch manager to buy more contacts."
  }
}
```

Important:
- applications to the company's own vacancies (`/negotiations`, `/applications/...`,
  `/profiles/get_profile/a/...`) consume neither rate limits nor paid quotas;
- an exhausted contacts package used to return `429` as well. If your client retries `429` with a
  backoff, add separate handling for `402`: retrying such a request will not help.

### 4.3. Candidate Search

1. `GET /candidates`
Purpose: search candidates published in getmatch collections (the shared pool).
Every call consumes one unit of the `search` limit and counts towards the burst limit (see 4.2.1).

Cards are returned anonymized: contacts, name and photo stay hidden until the candidate's contacts
are opened via `GET /profiles/get_profile/dp/{id}`.

Parameters:

| Parameter | Type | Description |
| --- | --- | --- |
| `q` | string | full text query over profiles |
| `page` | int | page number, starting from `0` |
| `per_page` | int | page size (`1..100`, default `20`) |
| `sorting` | string | result ordering |
| `skills` | string | comma separated skill slugs |
| `locations` | list | locations (values of the location/city dictionaries) |
| `region_statuses` | list | region statuses |
| `language` | string | required English level (and above) |
| `position` | string | job title anywhere in the experience |
| `last_position` | string | job title at the most recent workplace |
| `seniority` | list | `junior`, `middle`, `senior`, `lead`, `c_level` |
| `salary_from` / `salary_to` | int | salary expectations range |
| `salary_presence` | bool | only candidates with stated expectations |
| `yoe_from` / `yoe_to` | int | total years of experience |
| `age_from` / `age_to` | int | age (`0..120`) |
| `manager_experience` | bool | has management experience |
| `is_not_job_hopper` | bool | filter out frequent job changes |
| `trusted_only` | bool | only candidates that pass `trusted` or `hire_confirmed` |

Response:
- `found` - total number of matches;
- `page`, `pages`, `per_page` - pagination;
- `items` - candidate cards.

Card fields:
- `id` - identifier for `GET /profiles/get_profile/dp/{id}`;
- `hash_type` - always `dp`;
- `published_at` - publication date in the collection;
- `specialization_slug`, `specialization_name`;
- `contacts_opened` - `true` when the company already paid for this candidate's contacts;
- `profile` - anonymized profile card:
  - `name` - `null` until contacts are opened;
  - `photo_url`, `age`, `city`, `country`;
  - `salary_expectations_from`, `salary_expectations_to`, `salary_expectations_currency`;
  - `skills` - list of names, `skills_detailed` - objects `{name, slug}`;
  - `languages` - a `language -> level` map;
  - `positions` - work experience, `educations` - education;
  - `trusted` - verified candidate.

Search results never contain contacts (email, phone, telegram) in any form.

### 4.4. Candidate Profiles

1. `GET /profiles/get_profile/a/{hash_id}`
Purpose: get a candidate profile by application ID.
Rules:
- the call approves the application, reveals contacts and notifies the candidate;
- pool limits and quotas are not consumed.

2. `GET /profiles/get_profile/dp/{hash_id}`
Purpose: get a candidate profile by shared pool candidate ID
(digest candidate, the `id` field in `GET /candidates`).
Rules:
- consumes the `profile_view` rate limit;
- if the company has not opened this candidate's contacts yet, the `open_contacts` rate limit and
  one contact from the paid package are consumed as well;
- when a rate limit is exhausted - `429` (retry after `reset_at`);
- when the paid contacts package is used up - `402` (`contacts_quota_exhausted`).

Response fields for both endpoints:
`hash_id`, `hash_type`, `name`, `photo_url`, `birthdate`, `general_info`, `links`, `contacts`,
`locations`, `educations`, `languages`, `positions`, `skills`, `skills_detailed`.

### 4.5. Vacancies

In all endpoints `vacancy_id` is the numeric vacancy ID (for example `14567`), the same one returned
by `GET /vacancies/` and `GET /employers/{company_id}/vacancies/active`.

1. `GET /vacancies/`
Purpose: get the list of active vacancies for the authorized company.
Response: an array of `{id, created_at, name}` objects.

2. `GET /employers/{company_id}/vacancies/active?page=<int>&per_page=<int>`
Purpose: get active company vacancies with pagination. Inspired by HH API.
Parameters:
- `company_id` - company ID (can be obtained from `/me`).
- `page` - page number, starting from `0`.
- `per_page` - page size (`1..200`, default `20`).

3. `POST /employers/{company_id}/vacancies/{vacancy_id}/archive`
Purpose: unpublish a vacancy.
Successful response: `{}`.

4. `POST /employers/{company_id}/vacancies/{vacancy_id}/prolong`
Purpose: prolong the vacancy publication.
Rules:
- prolongation is available only when less than 7 days are left until `archive_date`,
  otherwise `403` (`Too early to prolong vacancy`);
- if the company has no publications left, the request does not fail: it returns
  `auto_prolonged = false` and notifies the getmatch manager.
Response:
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
Purpose: boost the vacancy and send it to matching candidates.
Payload:
- `include_viewed` - optional boolean, default `true`: whether to send it to candidates who have
  already seen the vacancy.
Rules:
- the vacancy must be active and not archived, otherwise `400` (`Vacancy boost unavailable`);
- there is a 3-day cooldown between boosts, otherwise `400` (`Boost cooldown not passed`);
- if the company has no publications left, the response contains `boosted = false` and
  `candidates_sent = 0`, and the getmatch manager gets notified.
Response:
```json
{
  "boosted": true,
  "vacancy": {"id": "14567", "name": "Senior Python Developer", "is_active": true},
  "candidates_sent": 128,
  "include_viewed": true
}
```

### 4.6. Vacancy Drafts

1. `POST /employers/{company_id}/vacancies/drafts`
Purpose: create a vacancy draft and submit it for validation.
Additionally: you may pass `recruiter_id` to assign the draft to another recruiter from the same company.

2. `GET /employers/{company_id}/vacancies/drafts`
Purpose: get company drafts (sorted by `updated_at DESC`).
Important: drafts with `accepted` status are not included in this list.

3. `GET /employers/{company_id}/vacancies/drafts/{draft_id}`
Purpose: get a company draft by ID.

4. `PATCH /employers/{company_id}/vacancies/drafts/{draft_id}`
Purpose: partially update a draft, optionally change `recruiter_id`, and re-submit it for validation.

5. `POST /employers/{company_id}/vacancies/drafts/{draft_id}/publish`
Purpose: queue a validated draft for publication.
Successful response: `{"status": "queued"}`.

6. `DELETE /employers/{company_id}/vacancies/drafts/{draft_id}`
Purpose: delete a draft (if it is not linked to a final vacancy yet).
Successful response: `204 No Content`.

#### 4.6.1. payload fields for create/update

Additional top-level field for `POST` and `PATCH`:
- `recruiter_id` - optional recruiter ID within the current company. If omitted:
  - on `POST`, the draft is assigned to the current authorized recruiter;
  - on `PATCH`, the current draft `recruiter_id` is preserved.

Required for `POST`:
- `position` - vacancy title.
- `salary_display_from`, `salary_display_to` - salary range.
- `salary_currency` - `rub` / `usd` / `eur` (`₽`, `$`, `€` are also accepted).
- `salary_taxes` - `net` or `gross`.
- `salary_is_total` - whether compensation is total.
- `language` - `ru` or `eng`.
- `type` - `only_web` or `web_and_tg`.

Optional fields:

| Field | Values | Description |
| --- | --- | --- |
| `location_requirements` | list of `{"location_raw": "<location text>"}` | location requirements |
| `work_format` | `office`, `hybrid`, `remote`, `relocation_company`, `relocation_candidate` | work format, applied to every `location_requirements` item |
| `description` | string of `1000..30000` characters | vacancy description, HTML allowed |
| `seniority` | `junior`, `middle`, `senior`, `lead`, `c_level` | seniority level |
| `english_level` | `a1`, `a2`, `b1`, `b2`, `c` (`c1` and `c2` are accepted as `c`) | required English |
| `salary_hidden` | bool | whether to hide the salary range |
| `salary_hidden_variant` | `approximate_interval`, `interval`, `top_hidden`, `full_hidden` | how exactly to hide the range |
| `incognito_publication` | bool | hide the company name from candidates |
| `cover_letter_required` | bool | whether a cover letter is mandatory |
| `cover_letter_placeholder` | string | cover letter hint |
| `required_years_of_experience` | int | required years of experience |
| `location_validation` | bool | validate the candidate's location |
| `auto_prolong` | bool | auto-prolong the publication |

Important:
- the payload schema is strict: an unknown field results in `422`;
- the minimum `description` length is 1000 characters, a common cause of `422`;
- for `PATCH`, send only the changed fields inside `payload` (partial update).

#### 4.6.2. Response fields

`id`, `status`, `errors`, `recruiter_id`, `recruiter_hash_id`, `company_id`, `vacancy_id`,
`can_publish`, `payload`, `created_at`, `updated_at`.

The `can_publish` field shows whether the company has enough available publications to publish
this draft.

#### 4.6.3. Draft lifecycle

- `new` - technical initial status.
- `filling` - system auto-fills fields.
- `validating` - required fields and content checks are running.
- `rejected` - validation errors found (see `errors`).
- `validated` - draft is ready for publication.
- `publishing` - publication command is queued.
- `accepted` - final vacancy created, `vacancy_id` is set.

Rules:
- editing (`PATCH`) is allowed only for `new`, `rejected`, `validated`, otherwise `409`;
- publishing (`/publish`) is allowed only for `validated` and `vacancy_id = null`, otherwise `409`;
- if the company has no publications left, `/publish` returns `402`;
- deletion is forbidden if status is `accepted` or `vacancy_id` is already set, otherwise `409`;
- all draft operations are available to any recruiter of the current company;
- `recruiter_id` can only be changed to an active recruiter from the same company.

#### 4.6.4. Validation errors

`errors` is an array of objects:
- `code` - error type (`required_field`, `max_length`, `content_policy`, `invalid_format`, `bad_words`, `other`);
- `justification` - human-readable message;
- `field_name` - field with the issue (for example, `description`, `location_requirements[0].format`).

### 4.7. Applications

1. `GET /negotiations?vacancy_id=<vacancy_id>`
Purpose: get vacancy application collections (status counters).

2. `GET /negotiations/{collection_name}?vacancy_id=<vacancy_id>&page=<int>&per_page=<int>`
Purpose: get the applications list for the selected collection with pagination
(`per_page`: `1..200`, default `20`).
Supported `collection_name` values:
- `all_applications`
- `in_progress_applications`
- `rejected_applications`
- `hired_applications`
- `pending_applications`

3. `GET /applications/{candidate_id}`
Purpose: get a candidate resume/profile from an application.
Query parameters:
- `open_contacts` - optional boolean flag, defaults to `false`.
Rules:
- by default the application is returned in closed form: `contact` is empty, `last_name` and `birth_date` are `null`;
- if `open_contacts=true` is passed, the API will try to reveal contacts and notify the candidate.

Response fields:
`id`, `first_name`, `last_name`, `age`, `birth_date`, `area {name}`, `cover_letter`, `contact`,
`photo_url`, `skill_set`, `education {primary[]}`, `experience[]`, `created_at`, `updated_at`.

Important:
- `created_at` is the application date (when it was sent to the employer), not the profile creation date;
- `updated_at` is the date the candidate last edited the profile manually.

Possible `contact[].type.id` values:
- `phone` - phone number;
- `email` - email address;
- `telegram` - Telegram username;
- `other` - another link from the candidate profile, for example GitHub or a personal website.

4. `POST /applications/{candidate_id}`
Purpose: review an application (approve or reject).
Payload:
- `resolution` - required: `approve` or `reject`.
- `reason` - optional reject reason text.
- `forward_reason` - optional flag: whether to send `reason` to the candidate.
Response:
- `id` - application hash_id;
- `state.id` - resulting status (`pending`, `in_progress`, `rejected`, `hired`);
- `state.name` - human-readable status label;
- `updated_at` - timestamp of the latest status update.

5. `PUT /applications/{candidate_id}`
Purpose: update application client status.
Payload:
- `client_status` - one of: `pending`, `in_progress`, `rejected`, `hired`.
Response: same as for `POST`.

Rules for application management:
- `candidate_id` is the application hash_id (from `GET /negotiations/...`);
- `hired` is a final status: it cannot be changed back to another status (`400`);
- reset to `pending` is forbidden if the candidate has already been notified about a resolution (`409`);
- an automatically rejected application cannot be reviewed again (`409`);
- status update is allowed only for the vacancy owner recruiter or a company admin, otherwise `405`.

### 4.8. Application Webhooks

1. `GET /webhooks/applications`
Purpose: get the current applications webhook URL for the company.
Response:
- `url` - current webhook URL or `null`;
- `include_contacts` - whether webhook payloads should include revealed contacts.

2. `PUT /webhooks/applications`
Purpose: set/update the applications webhook URL for the company.
Payload:
- `url` - a valid HTTP/HTTPS URL, or `null` to disable webhooks;
- `include_contacts` - boolean flag that controls whether webhook payloads include contacts.
Response: current `url` and `include_contacts`.

Rules:
- one applications webhook URL is supported per company;
- update affects only the current authorized recruiter's company.

#### 4.8.1. Outgoing webhook events

After URL is configured, getmatch sends `POST` requests with JSON payload to this endpoint.

Supported `event` values:
- `application_created` - a new application is sent to the employer for the first time;
- `application_status_changed` - application state has changed.

Important:
- `application_created` webhook is sent only once per application;
- `application_status_changed` is sent only on an actual state change.

Payload fields:
- `event` - event type;
- `vacancy_id` - numeric vacancy ID;
- `state.id` - system status: `pending`, `in_progress`, `rejected`, `hired`;
- `state.name` - human-readable status name (currently returned in Russian);
- `application` - the same object returned by `GET /applications/{candidate_id}`, plus the
  `profile_created_at` field (candidate profile creation date).

`application.id` is the application hash_id (the same ID used in `GET /applications/{candidate_id}`).

Contacts and personal data depend on the webhook setting:
- with `include_contacts=false`, `contact` is empty and `last_name`/`birth_date` are `null`;
- with `include_contacts=true`, webhook payloads include contacts regardless of the internal contact
  visibility state of the application.

## 5. Usage Examples

### 5.1. Search candidates

```bash
curl --request GET \
  --url "https://getmatch.ru/api/integrations/v1/candidates?q=python&seniority=senior&per_page=20" \
  --header "Authorization: Bearer <access_token>"
```

Response example (fragment):
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
      "specialization_name": "Backend Developer",
      "contacts_opened": false,
      "profile": {
        "name": null,
        "age": 29,
        "city": "Belgrade",
        "country": "Serbia",
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

### 5.2. Open contacts of a found candidate

```bash
curl --request GET \
  --url "https://getmatch.ru/api/integrations/v1/profiles/get_profile/dp/pQ2M8Zx1" \
  --header "Authorization: Bearer <access_token>"
```

After a successful call, the same candidate is returned by search with `contacts_opened: true`
and a filled `profile.name`.

### 5.3. Create a draft

```bash
curl --request POST \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/drafts" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{
    "recruiter_id": 101,
    "payload": {
      "position": "Senior Python Developer",
      "location_requirements": [{"location_raw": "Belgrade"}],
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
      "description": "<p>We are looking for a Senior Python engineer...</p>"
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
  "can_publish": true,
  "created_at": "2026-03-05T10:15:30.123456",
  "updated_at": "2026-03-05T10:15:30.123456"
}
```

### 5.4. Check draft status

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

### 5.5. Fix a rejected draft

```bash
curl --request PATCH \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/drafts/14567" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{
    "recruiter_id": 115,
    "payload": {
      "description": "Full vacancy description with responsibilities and requirements"
    }
  }'
```

After `PATCH`, the draft goes through `filling -> validating -> rejected|validated` again.

### 5.6. Publish a validated draft

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

### 5.7. Unpublish a vacancy

```bash
curl --request POST \
  --url "https://getmatch.ru/api/integrations/v1/employers/<company_id>/vacancies/<vacancy_id>/archive" \
  --header "Authorization: Bearer <access_token>"
```

Response:
```json
{}
```

### 5.8. Prolong and boost a vacancy

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

### 5.9. Configure applications webhook

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

Disable webhook:
```bash
curl --request PUT \
  --url "https://getmatch.ru/api/integrations/v1/webhooks/applications" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{"url": null, "include_contacts": false}'
```

### 5.10. Incoming application webhook payload example without contacts

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
    "first_name": "Ivan",
    "last_name": null,
    "age": 29,
    "birth_date": null,
    "area": {"name": "Belgrade"},
    "cover_letter": "Happy to discuss this role",
    "contact": [],
    "photo_url": "https://getmatch.ru/uploads/u/folder/avatar.jpg",
    "skill_set": ["Python", "FastAPI"],
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
    "created_at": "2026-03-10T12:45:30+0000",
    "profile_created_at": "2025-11-10T09:15:00+0000",
    "updated_at": "2026-03-10T12:45:30+0000"
  }
}
```

### 5.11. Incoming application webhook payload example with contacts

Only the personal fields differ:
```json
{
  "application": {
    "last_name": "Ivanov",
    "birth_date": "1996-04-12",
    "contact": [
      {"contact_value": "candidate@example.com", "type": {"id": "email"}},
      {"contact_value": "@candidate", "type": {"id": "telegram"}}
    ]
  }
}
```

### 5.12. Get an application without contacts (default behavior)

```bash
curl --request GET \
  --url "https://getmatch.ru/api/integrations/v1/applications/<candidate_id>" \
  --header "Authorization: Bearer <access_token>"
```

Response example:
```json
{
  "id": "pQ2M8Zx1",
  "first_name": "Ivan",
  "last_name": null,
  "birth_date": null,
  "contact": [],
  "cover_letter": "Happy to discuss this role"
}
```

### 5.13. Get an application and reveal contacts

```bash
curl --request GET \
  --url "https://getmatch.ru/api/integrations/v1/applications/<candidate_id>?open_contacts=true" \
  --header "Authorization: Bearer <access_token>"
```

### 5.14. Reject an application with reason

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
  "state": {"id": "rejected", "name": "Отказ"},
  "updated_at": "2026-03-12T08:40:00+0000"
}
```

### 5.15. Set application status to hired

```bash
curl --request PUT \
  --url "https://getmatch.ru/api/integrations/v1/applications/<candidate_id>" \
  --header "Authorization: Bearer <access_token>" \
  --header "Content-Type: application/json" \
  --data '{"client_status": "hired"}'
```

Response:
```json
{
  "id": "pQ2M8Zx1",
  "state": {"id": "hired", "name": "Нанят"},
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

- `200 OK` - successful request
- `204 No Content` - successful draft deletion
- `400 Bad Request` - invalid parameters or a forbidden application status transition
- `401 Unauthorized` - missing/invalid/expired token
- `402 Payment Required` - a paid quota is used up: the contact opening package
  (`detail.code = "contacts_quota_exhausted"`) or available vacancy publications (`/publish`).
  Retrying will not help - more has to be purchased
- `403 Forbidden` - the operation is not available right now (for example, too early to prolong a vacancy)
- `404 Not Found` - object not found or unavailable
- `405 Method Not Allowed` - operation is not allowed for the current recruiter
- `409 Conflict` - state conflict (draft status, repeated application resolution)
- `422 Unprocessable Entity` - request payload schema error
- `429 Too Many Requests` - a rate limit fired. Two cases, told apart by `detail.code`:
  `rate_limit_exceeded` - too many requests per minute, retry after `Retry-After`;
  `digest_limit_exceeded` - the daily or monthly quota is spent (`search` / `profile_view` /
  `open_contacts`), retry after `reset_at` (see 4.2)
