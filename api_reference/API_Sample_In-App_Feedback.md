# LearnSphere Feedback API — API Reference

> **Sample document.** This is a fictional, filled-in example of `API_Template.md`. It is the reference that people **outside the feedback team** read: app and web developers, other LearnSphere teams, and QA. It is built from [openapi_Sample_In-App_Feedback.yaml](openapi_Sample_In-App_Feedback.yaml), and it matches the [PRD](../product_requirement_document/PRD_Sample_In-App_Feedback.md), [DRD](../design_requirement_document/DRD_Sample_In-App_Feedback.md), [TRD](../technical_requirement_document/TRD_Sample_In-App_Feedback.md), and [SDD](../software_design_document/SDD_Sample_In-App_Feedback.md) samples, so the product ("LearnSphere"), limits, names, and error codes are the same. The company, URLs, and people are invented. Product UI strings (for example "You've reached today's limit") stay in English because the sample product launches in English.
>
> It says **what the API does and how to call it**. How it is built (queues, retries, Zendesk, tables) is in the SDD and is not repeated here.

| | |
|---|---|
| **API** | LearnSphere Feedback API |
| **API version** | `v1` (document version 1.0.0) |
| **Base URLs** | Production: `https://api.learnsphere.example/v1`<br>Staging: `https://api.staging.learnsphere.example/v1` |
| **Status** | Preview. Not released yet; internal dogfood is planned for week 6 of the project. |
| **Owner** | Feedback service team (Engineering), `feedback-api@learnsphere.example` |
| **Last updated** | 2026-10-30 |
| **Source of truth** | [openapi_Sample_In-App_Feedback.yaml](openapi_Sample_In-App_Feedback.yaml). If this page and the file differ, the file is right. |
| **Design documents** | [PRD](../product_requirement_document/PRD_Sample_In-App_Feedback.md) v1.0 · [DRD](../design_requirement_document/DRD_Sample_In-App_Feedback.md) v1.0 · [TRD](../technical_requirement_document/TRD_Sample_In-App_Feedback.md) v1.1 · [SDD](../software_design_document/SDD_Sample_In-App_Feedback.md) v0.3 |

---

## 1. Overview

### 1.1 What the API does

The Feedback API lets a signed-in learner:

1. **Attach screenshots** to a report (up to 3).
2. **Send a feedback report** about a problem in the app.
3. **See their reports** and the replies from LearnSphere Support.
4. **Say whether the answer helped.** "No" reopens the report.

Each report gets a reference number such as `FB-20261030-0042`. Support receives it as a ticket shortly after the report is accepted.

### 1.2 Who it is for

| Reader | Typical use |
|--------|-------------|
| Mobile and web developers | Build the feedback form and the "My Feedback" screens |
| Other LearnSphere teams | Read report status (for example, a future help center) |
| QA | Test the flows and error cases |

### 1.3 Quick start

Send a report with one screenshot. You need a learner access token ([Section 2](#2-authentication)).

**Step 1. Ask for an upload form** (skip steps 1 and 2 if there is no screenshot):

```bash
curl -X POST https://api.learnsphere.example/v1/feedback/attachments/presign \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "contentType": "image/png", "sizeBytes": 2048000 }'
```

The response gives you `attachmentKey` and an `upload` form (`url` and `fields`).

**Step 2. Upload the file.** Send a multipart form to `upload.url`. Put every entry of `upload.fields` first, exactly as returned, and the file last:

```bash
curl -X POST "$UPLOAD_URL" \
  -F "key=…" -F "policy=…" -F "x-amz-signature=…" \
  -F "file=@screenshot.png"
```

**Step 3. Send the report:**

```bash
curl -X POST https://api.learnsphere.example/v1/feedback \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "idempotencyKey": "6f0c0e8e-3b7a-4f1e-9c52-0d1f6a7b8c90",
    "issueType": "video_audio",
    "description": "Video freezes at 3:42",
    "replyEmail": "sam@example.com",
    "attachmentKeys": ["feedback/0b6d1f0a-…/a1b2c3d4-….png"],
    "context": {
      "courseId": "sql-101", "lessonId": "l-004",
      "deviceModel": "iPhone14,5", "osVersion": "iOS 17.4",
      "appVersion": "5.8.0", "locale": "en-US",
      "clientSubmittedAt": "2026-10-30T14:05:11Z"
    }
  }'
```

Response `201`:

```json
{ "id": "b1c7e5d2-8a40-4c3e-9f11-2a6d5e0c7b33", "reference": "FB-20261030-0042", "status": "received" }
```

---

## 2. Authentication

Every call needs a learner access token in the `Authorization` header:

```text
Authorization: Bearer <token>
```

| Item | Rule |
|------|------|
| Where the token comes from | The LearnSphere sign-in flow (Auth service). The Feedback API does not issue tokens. |
| Who can send reports | **Paying learners only.** A token for any other plan gets `403 AUTH-002` on `POST /feedback`. |
| Who can read a report | Only the learner who sent it. Anyone else gets `404`, not `403`, so IDs cannot be guessed. |
| Expired or invalid token | `401 AUTH-001`. Ask the learner to sign in again. Keep their draft. |

---

## 3. Conventions

| Topic | Rule |
|-------|------|
| Format | JSON in and out. Send `Content-Type: application/json`. |
| Versioning | The major version is in the URL (`/v1`). New fields may be added to responses at any time, so **ignore fields you do not know**. A breaking change needs `/v2`, and `/v1` keeps working for at least 6 months afterwards, because old app versions stay in use. |
| Identifiers | Report and attachment IDs are UUIDs. The reference (`FB-YYYYMMDD-NNNN`, UTC date) is for showing to people and quoting to Support. |
| Timestamps | ISO 8601 in UTC, for example `2026-10-30T14:05:11Z`. |
| Status values | `received` (shown as Received), `in_progress` (In progress), `resolved` (Resolved). |
| Issue types | `video_audio`, `quiz_grading`, `course_content`, `billing`, `app_bug`, `other`. |
| Retries | `POST /feedback` is safe to retry with the same `idempotencyKey` (see 5.2). Other `POST` calls are not idempotent. |
| Pagination | Cursor based, 20 items per page, newest first. Use `nextCursor` as `cursor`. `nextCursor` is `null` on the last page. |
| History window | Lists and details cover the **last 12 months**. Older reports are deleted. |

### 3.1 Rate limits

Limits are per learner. Going over a limit returns `429 RATE-001`.

| Endpoint | Limit |
|----------|-------|
| `POST /feedback/attachments/presign` | 30 per hour |
| `POST /feedback` | 10 reports per 24 hours |
| `GET /feedback` | 60 per minute |
| `GET /feedback/{id}` | 60 per minute |
| `POST /feedback/{id}/rating` | 10 per minute |

---

## 4. Errors

Errors use the HTTP status code and this JSON body:

```json
{
  "code": "VAL-001",
  "messageKey": "feedback.validation.description",
  "details": { "field": "description" }
}
```

| Field | Meaning |
|-------|---------|
| `code` | Stable code, `CATEGORY-NNN`. **Use this in your code.** |
| `messageKey` | Key for the learner-facing text. The app maps it to its own wording and translations; the API does not send display text. |
| `details` | Optional. For `VAL-001`, `details.field` names the field that failed. |

### 4.1 Error codes

| HTTP | Code | Meaning | What your client should do |
|------|------|---------|----------------------------|
| 400 | `VAL-001` | A field is missing or invalid. | Show the message for `details.field`. Keep what the learner typed. |
| 400 | `VAL-002` | A screenshot is missing, the wrong type, not yours, more than 3, or over 5 MB. | Show "Image is too large (max 5 MB)" or the type message. |
| 401 | `AUTH-001` | Token missing, expired, or invalid. | Show the sign-in sheet, then restore the draft. |
| 403 | `AUTH-002` | The learner is not on a paying plan. | Show a generic error. The entry point should be hidden for these learners. |
| 404 | `NOTFOUND-001` | No such report, or it is not the learner's. | Show the "not found" state. |
| 409 | `STATE-001` | Rating not allowed: the report is not resolved, or this resolution is already rated. | Hide the rating card. |
| 429 | `RATE-001` | A request limit was reached (see 3.1). | For reports: "You've reached today's limit. Please try again tomorrow or email support." |
| 503 | `SYS-001` | The service is temporarily unavailable. | Save a draft and retry (see 4.2). |

### 4.2 When to retry

| Result | Retry? |
|--------|--------|
| Network error, timeout, or `503 SYS-001` | **Yes**, with the **same** `idempotencyKey`. We suggest waiting 15 s, 30 s, 60 s, 2 min, then 5 min, and then keeping the draft and asking the learner to try again. |
| `401 AUTH-001` | After signing in again. |
| `429 RATE-001` | No. Wait until the window passes. |
| Other `4xx` | No. Fix the request. |

---

## 5. Endpoints

All paths are relative to the base URL. All calls need the `Authorization` header.

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/feedback/attachments/presign` | Get a one-time upload form for a screenshot |
| POST | `/feedback` | Send a report |
| GET | `/feedback` | List your reports |
| GET | `/feedback/{id}` | Read one report with its replies |
| POST | `/feedback/{id}/rating` | Say whether the resolution helped |

### 5.1 Get an upload form for a screenshot

`POST /feedback/attachments/presign`

Returns a presigned upload form for **one** PNG or JPEG file. The form expires after 5 minutes. Request a new one if it expires.

**Request body**

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `contentType` | string | Yes | `image/png` or `image/jpeg` |
| `sizeBytes` | integer | Yes | 1 to 5,242,880 (5 MB) |

**Response `200`**

```json
{
  "attachmentKey": "feedback/0b6d1f0a-…/a1b2c3d4-….png",
  "upload": {
    "url": "https://…s3…",
    "fields": { "key": "…", "policy": "…", "x-amz-signature": "…" }
  },
  "expiresInSeconds": 300
}
```

Keep `attachmentKey` and send it in `attachmentKeys` when you create the report. The upload form fixes the file type and size, so a different file type or a larger file is rejected by the upload itself.

**Errors:** `VAL-002` (wrong content type), `AUTH-001`, `RATE-001`, `SYS-001`.

### 5.2 Send a report

`POST /feedback`

Creates the report and returns its reference. Paying learners only. Limit: 10 reports per 24 hours.

**Safe to retry.** Create the `idempotencyKey` when the form opens, store it with the draft, and send the same value on every retry. If an earlier call succeeded, you get the **original** `201` response again and no second report is created. A retry of an accepted report is never blocked by the 10-per-day limit.

**Request body**

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `idempotencyKey` | string (UUID v4) | Yes | One per report. Reuse it on retries. |
| `issueType` | string | Yes | One of the 6 issue types in Section 3 |
| `description` | string | Yes | 10 to 1,000 characters after trimming spaces |
| `replyEmail` | string | Yes | Valid email, up to 254 characters |
| `attachmentKeys` | string[] | No | Up to 3. Each must be a key you received from 5.1 for this learner. |
| `context.deviceModel`, `osVersion`, `appVersion`, `locale` | string | Yes | Not empty |
| `context.clientSubmittedAt` | string | Yes | ISO 8601 time when the learner pressed Send |
| `context.courseId`, `context.lessonId` | string | No | Send both or neither |

**Response `201`**

| Field | Type | Meaning |
|-------|------|---------|
| `id` | string (UUID) | Use this to read the report |
| `reference` | string | Show this to the learner, for example `FB-20261030-0042` |
| `status` | string | Always `received` for a new report |

**What happens next:** Support receives the report as a ticket within minutes. The learner gets an acknowledgment by in-app notification, push, and (unless turned off) email, normally within 5 minutes. The API does not wait for any of this.

**Errors:** `VAL-001`, `VAL-002`, `AUTH-001`, `AUTH-002`, `RATE-001`, `SYS-001`.

### 5.3 List your reports

`GET /feedback?cursor={cursor}`

Returns the learner's reports from the last 12 months, newest first, 20 per page.

**Query parameters**

| Name | Required | Meaning |
|------|----------|---------|
| `cursor` | No | The `nextCursor` of the previous page. Leave it out for the first page. |

**Response `200`**

```json
{
  "items": [
    {
      "id": "b1c7e5d2-8a40-4c3e-9f11-2a6d5e0c7b33",
      "reference": "FB-20261030-0042",
      "issueType": "video_audio",
      "status": "in_progress",
      "createdAt": "2026-10-30T14:05:12Z",
      "hasUnreadUpdate": true
    }
  ],
  "nextCursor": null
}
```

`hasUnreadUpdate` is `true` when the report changed (a reply, or a new status) since the learner last opened it. Use it for the unread dot.

**Errors:** `AUTH-001`, `RATE-001`, `SYS-001`.

### 5.4 Read one report

`GET /feedback/{id}`

Returns everything in the list item, plus the description, screenshots, replies, and the learner's rating.

**Response `200`**

```json
{
  "id": "b1c7e5d2-8a40-4c3e-9f11-2a6d5e0c7b33",
  "reference": "FB-20261030-0042",
  "issueType": "video_audio",
  "status": "resolved",
  "createdAt": "2026-10-30T14:05:12Z",
  "hasUnreadUpdate": false,
  "description": "Video freezes at 3:42",
  "attachments": [
    { "url": "https://files.learnsphere.example/feedback/a1b2c3d4.png?X-Amz-Expires=300", "contentType": "image/png" }
  ],
  "replies": [
    {
      "agentFirstName": "Maya",
      "body": "Thanks for the report. We fixed a playback issue in app 5.8.1. Please update and try again.",
      "postedAt": "2026-10-30T16:40:00Z"
    }
  ],
  "rating": null
}
```

| Field | Notes |
|-------|-------|
| `attachments[].url` | A signed link that works for **5 minutes**. Do not store it; read the report again for a fresh link. |
| `replies` | Public replies from Support, oldest first. Internal notes are never included. |
| `rating` | The learner's rating for the current resolution, or `null`. |

**Side effect:** reading a report marks its updates as seen, so `hasUnreadUpdate` becomes `false`.

**Errors:** `NOTFOUND-001` (also returned for another learner's report and for reports older than 12 months), `AUTH-001`, `RATE-001`, `SYS-001`.

### 5.5 Rate the resolution

`POST /feedback/{id}/rating`

Lets the learner say whether the answer helped. Allowed **once per resolution**, and only while the report is `resolved`.

**Request body**

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `helped` | boolean | Yes | `true` or `false` |
| `comment` | string | No | Up to 300 characters |

**Response `200`**

| `helped` | `status` in the response | `messageKey` | What happens |
|----------|--------------------------|--------------|--------------|
| `true` | `resolved` | `feedback.rating.thanks` | Nothing more. |
| `false` | `in_progress` | `feedback.rating.reopened` | The report is reopened and Support picks it up again. |

If a reopened report is resolved again, the learner can rate it again.

**Errors:** `VAL-001` (for example, a comment over 300 characters), `NOTFOUND-001`, `STATE-001` (not resolved, or already rated), `AUTH-001`, `RATE-001`, `SYS-001`.

---

## 6. Events and Webhooks

Version 1 does **not** send webhooks to outside callers. To follow a report:

- Show the in-app notification or push the learner receives when Support replies or resolves the report, and open the report from its deep link `learnsphere://feedback/{id}`.
- Or refresh `GET /feedback` and check `hasUnreadUpdate`.

The service also receives events from the support tool. That endpoint is internal and is documented in the SDD, not here.

---

## 7. Changelog and Deprecation

| Version | Date | Change |
|---------|------|--------|
| 1.0.0 | Planned for dogfood (project week 6) | First release: screenshots, reports, list and detail, rating. |

**Deprecation policy:** we announce deprecations in the release notes and in this table, and keep the old version running for at least 6 months. Before removing a version we check how much traffic it still receives.

**Change rules:** adding a response field, an optional request field, or a new endpoint is **not** a breaking change. Removing or renaming a field, changing a type, or making an optional field required **is**, and needs a new major version.

---

## 8. Support

| Need | Contact |
|------|---------|
| Questions about using the API | `feedback-api@learnsphere.example` |
| Found a mistake in this document or the OpenAPI file | Open a pull request on the `learnsphere/feedback-service` repository |
| Production problem | Page the Feedback service on-call (see `runbooks/feedback-api.md`) |
