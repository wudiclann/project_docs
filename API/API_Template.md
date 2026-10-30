# [API Name] — API Reference

| | |
|---|---|
| **API** | [Name] |
| **API version** | [v1] (document version [1.0.0]) |
| **Base URLs** | Production: `[https://…]`<br>Staging: `[https://…]` |
| **Status** | Preview / Beta / Stable / Deprecated |
| **Owner** | [Team name and contact] |
| **Last updated** | [YYYY-MM-DD] |
| **Source of truth** | [Link to the OpenAPI file in the code repository] |
| **Design documents** | [Links to the SDD and TRD this API implements, with versions] |

> **How to use this template**
> - The API reference is for **people outside the team that builds the service**: app and web developers, other teams, partners, and QA. It says what the API does and how to call it. How it is built belongs in the SDD.
> - **Generate it, don't retype it.** The OpenAPI file is the source of truth. Endpoint tables, request and response schemas, and examples should come from it. Write by hand only what a tool can't know: the overview, quick start, rules, retry advice, and changelog.
> - Replace everything in `[brackets]`. Delete guidance lines (blockquotes) and sections that don't apply, such as Webhooks.
> - Write for someone who has never seen your code. Don't mention queues, tables, internal services, or retry timings inside the service.
> - Update this document **in the same change** as the API. The reference is only useful while it is accurate.

---

## 1. Overview

### 1.1 What the API does

[Two or three sentences, then a short list of what a caller can do.]

### 1.2 Who it is for

| Reader | Typical use |
|--------|-------------|
| [Mobile and web developers] | [What they build with it] |
| [Other teams or partners] | [What they do with it] |

### 1.3 Quick start

> Show the shortest path to one successful result, with copy-and-paste calls. Use placeholders for secrets, like `$TOKEN`.

**Step 1. [First call]**

```bash
curl -X POST [URL] \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '[Request body]'
```

[What the response contains and what to do with it.]

**Step 2. [Next call]**

[Repeat for each step.]

---

## 2. Authentication

> Say how a caller gets credentials, how to send them, who is allowed, and what happens when they fail.

| Item | Rule |
|------|------|
| Where credentials come from | [Sign-in flow, API key portal, partner onboarding] |
| How to send them | [`Authorization: Bearer <token>`] |
| Who is allowed | [Plans, roles, scopes] |
| Expired or invalid credentials | [Status and code, and what the caller should do] |
| Access to other users' data | [For example: returns `404`, not `403`] |

---

## 3. Conventions

| Topic | Rule |
|-------|------|
| Format | [JSON; required headers] |
| Versioning | [Where the version is; what counts as a breaking change; how long old versions run] |
| Identifiers | [Types and formats] |
| Timestamps | [ISO 8601 in UTC] |
| Enumerated values | [Status values, types, with the meaning of each] |
| Retries | [Which calls are safe to retry, and how (idempotency key)] |
| Pagination | [Cursor or page; page size; how to get the next page] |
| Data window | [How far back data is available] |

### 3.1 Rate limits

| Endpoint | Limit | Per |
|----------|-------|-----|
| [`POST /…`] | [30 per hour] | [user] |

[What a caller sees when over the limit, and when to try again.]

---

## 4. Errors

[Show the error body once, with a real example.]

```json
{
  "code": "[CATEGORY-001]",
  "messageKey": "[key]",
  "details": {}
}
```

| Field | Meaning |
|-------|---------|
| `code` | [Stable code. Callers should branch on this.] |
| `messageKey` | [How the wording is chosen] |
| `details` | [Extra facts, such as which field failed] |

### 4.1 Error codes

| HTTP | Code | Meaning | What the caller should do |
|------|------|---------|---------------------------|
| [400] | [`VAL-001`] | [When it happens] | [Action] |

### 4.2 When to retry

| Result | Retry? |
|--------|--------|
| [Network error, timeout, 5xx] | [Yes, with the same idempotency key; suggested waits] |
| [429] | [No; wait until the window passes] |
| [Other 4xx] | [No; fix the request] |

---

## 5. Endpoints

> One block per endpoint. If your tool generates this from OpenAPI, keep the generated parts and add the notes a tool can't write: rules, side effects, and what happens next.

| Method | Path | Purpose |
|--------|------|---------|
| [POST] | [`/…`] | [One line] |

### 5.1 [Endpoint name]

`[METHOD] [/path]`

[What it does, who can call it, limits, and anything that surprises callers.]

**Request**

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| [name] | [type] | [Yes/No] | [Limits, formats, allowed values] |

**Response `[200]`**

```json
{}
```

| Field | Notes |
|-------|-------|
| [name] | [Meaning, lifetime, whether to store it] |

**Side effects:** [What changes besides the response, if anything.]

**Errors:** [`CODE`, `CODE`]

---

## 6. Events and Webhooks

> Delete this section if the API sends no events to callers. If you delete it, say so in one line ("Version 1 sends no webhooks") so readers don't wonder.

| Event | When it is sent | Payload | Notes |
|-------|-----------------|---------|-------|
| [`name`] | [Trigger] | [Link to schema] | [Retries, order] |

**Verifying a request:** [Signature header, algorithm, tolerance for old timestamps.]

**Retries:** [How often we retry, for how long, and how the receiver should answer.]

---

## 7. Changelog and Deprecation

| Version | Date | Change |
|---------|------|--------|
| [1.0.0] | [YYYY-MM-DD] | [First release] |

**Deprecation policy:** [How you announce it, how long the old version keeps working.]

**Change rules:** [What is not breaking (new optional fields, new endpoints) and what is (removing, renaming, changing types).]

---

## 8. Support

| Need | Contact |
|------|---------|
| [Questions about using the API] | [Email or channel] |
| [Mistake in this document] | [Where to report it] |
| [Production problem] | [On-call or status page] |
