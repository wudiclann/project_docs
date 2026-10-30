# How to Write an API Reference — Guide

This guide explains what an API reference (API doc) is, how it relates to the SDD, how to use the template, and what to write in each section.

**Use it together with:**
- [API_Template.md](API_Template.md): the blank template to copy and fill in
- [API_Sample_In-App_Feedback.md](API_Sample_In-App_Feedback.md): a completed example, so you can see what "good" looks like. It is fictional and matches the PRD, DRD, TRD, and SDD samples, so its names and numbers are illustrations, not defaults.
- [openapi_Sample_In-App_Feedback.yaml](openapi_Sample_In-App_Feedback.yaml): the OpenAPI file the sample reference is built from
- [SDD_Guide.md](../SDD/SDD_Guide.md): the detailed design that sits behind the API

---

## Part 1: API Doc Basics

### What is an API reference?

An API reference tells **someone who calls your service** what it does and how to use it: how to authenticate, what each endpoint takes and returns, which errors can happen, what the limits are, and what changed between versions. The reader never sees your code.

It is a **living document**. Unlike the PRD, DRD, and TRD, it is not approved once and left alone, and unlike the SDD it does not end as a record. It stays accurate for as long as the API is in use.

### API reference vs. SDD

| | SDD (Section 5, Interface Detail) | API reference |
|---|-----------------------------------|---------------|
| Reader | Engineers who build and maintain the service | Everyone who calls it: app and web developers, other teams, partners, QA |
| Question | *Why is it built this way, and how does it work inside?* | *What can I call, and what do I get back?* |
| Contains | Permissions per endpoint, side effects, internal rules, queue messages, error handling inside the service | Endpoints, schemas, examples, authentication, limits, error codes, retry advice, changelog |
| Does not contain | Public tutorials and quick starts | Tables, queues, retry timings, internal services |
| Written by | Engineers, as they design | Engineers, mostly **generated** from the code |
| Life | Changes while building; ends as the as-built record | Changes with every API change, for as long as the API is used |

A quick test: if a caller needs it to **use the API correctly**, it goes in the API reference. If an engineer needs it to **build or maintain the service**, it goes in the SDD.

### One source of truth

The **OpenAPI file** is the source of truth. It is generated from the code (for example, decorators in NestJS, annotations in Spring, or a schema in FastAPI) or, for a contract-first project, written first and checked against the code in CI.

```text
code or contract  ->  openapi.yaml  ->  published API reference
                         ^
                         |-- SDD Section 5 links here (does not copy it)
```

- The **SDD links** to the OpenAPI file and the published reference. It keeps only what the file can't say.
- The **API reference** is the file, plus the pages a tool can't write (overview, quick start, rules, retry advice, changelog).
- If this guide's template and the OpenAPI file ever differ, the file wins. Fix the document.

### Do you need one?

| Situation | What to do |
|-----------|------------|
| Only your own team calls the service | SDD Section 5 is enough. No separate reference. |
| Other teams, mobile, or web clients call it | Write the API reference. Agree the contract early so they can build in parallel. |
| Partners or the public call it | Publish the reference, with a changelog, deprecation policy, and a getting-started guide. |

### What an API reference is not

| An API reference is not... | Because... |
|----------------------------|------------|
| A copy of the SDD | Callers don't need your internals. Link to nothing internal. |
| Hand-typed schemas | They go stale. Generate them from OpenAPI. |
| A tutorial for the whole product | Cover the API only. Link to product docs for the rest. |
| A promise of how it works inside | Say what callers can rely on. Anything else can change. |

### Who reads it, and what they look for

| Reader | Looking for |
|--------|-------------|
| Developer starting out | Sections 1.3 and 2: a working first call |
| Developer building a feature | Sections 5 and 4: fields, rules, errors |
| QA | Sections 3, 4, 5: limits, error codes, edge cases |
| Another team's engineer | Sections 3 and 7: versioning and what may change |
| On-call or support | Sections 4 and 8: what an error means, who to contact |

### Writing principles

1. **Start with a working example.** A caller who has made one successful call understands the rest faster.
2. **Write what callers can rely on.** Say what is guaranteed. Leave out what is only how it works today.
3. **Be exact.** Limits, lengths, allowed values, and timeouts as numbers, not "reasonable".
4. **Show real examples.** Realistic requests and responses, not `string`.
5. **Explain every error.** For each code: when it happens and what the caller should do.
6. **Say what is safe to retry.** Most integration bugs are retries that duplicate or are skipped.
7. **Don't leak the inside.** No table names, queue names, or internal service names. They create dependencies you can't remove.
8. **Keep it in step with the code.** Update it in the same change as the API.

---

## Part 2: The Writing Process

Don't fill the template top to bottom. A better order:

| Step | Do this | Sections |
|------|---------|----------|
| 1. Agree the contract | Take the endpoints from the TRD's interface overview; agree the shape with the callers | 5 |
| 2. Annotate the code | Add summaries, descriptions, examples, and error responses so the OpenAPI file is complete | 5 |
| 3. Generate the file | Build `openapi.yaml`; check it in CI against the running code | — |
| 4. Describe the conventions | Authentication, versioning, pagination, limits | 2, 3 |
| 5. Document the errors | One table, with what to do for each | 4 |
| 6. Write the quick start | One end-to-end path with copy-and-paste calls | 1 |
| 7. Add the rest | Webhooks, changelog, support | 6, 7, 8 |
| 8. Test the examples | Run every example against staging | all |
| 9. Publish and announce | Put it where callers will find it; link it from the SDD | — |

### Choosing a size

| Situation | Include | Skip |
|-----------|---------|------|
| **Full**: partners or the public | Everything, plus a getting-started guide | Nothing |
| **Standard**: other internal teams (default) | Sections 1–5, 7, 8 | Webhooks if none |
| **Short**: one or two endpoints for one other team | 1.3, 2, 4, 5 | The rest. A single page is fine. |

---

## Part 3: Section-by-Section Guide

For each section: **what it's for**, **what to write**, **tips**, and **common mistakes**.

---

### Document Header

**Write:** API name, API version and document version, base URLs, status (Preview / Beta / Stable / Deprecated), owner and contact, last-updated date, link to the OpenAPI file, links to the SDD and TRD.

**Common mistake:** No owner or contact. Callers need to know whom to ask.

---

### 1. Overview

**What it's for:** let a reader decide in one minute whether this API is what they need, and make a first call in five.

**Write:**
- 1.1: what it does, as a short list of things a caller can do
- 1.2: who it is for
- 1.3: a quick start with copy-and-paste calls for one complete path

**Tips:** use placeholders for secrets (`$TOKEN`). Show the response, and say what to keep from it for the next call.

**Common mistake:** a quick start that skips a step (for example, how to get a token), so it fails the first time.

---

### 2. Authentication

**What it's for:** answer "how do I get in?" and "why did I get a 401 or 403?"

**Write:** where credentials come from, how to send them, who is allowed, what happens when they fail, and what happens when a caller asks for someone else's data.

**Tips:** if you hide other users' data behind `404`, say so, or callers will be confused. Never put real tokens in the document.

---

### 3. Conventions

**What it's for:** rules that apply to every endpoint, written once.

**Write:** format, versioning and what counts as a breaking change, identifiers, timestamps, enumerated values, retries and idempotency, pagination, data window, and a **rate limit table** (3.1).

**Tips:** tell callers to **ignore unknown fields**. That one sentence lets you add fields without a new version.

**Common mistake:** rate limits missing, or described as "reasonable".

---

### 4. Errors

**What it's for:** let callers handle every failure on purpose.

**Write:** the error body once, a table of codes with what to do for each (4.1), and a retry table (4.2).

**Tips:**
- Callers should branch on the stable `code`, not on message text.
- If the client owns the wording, say the API returns a key, not display text.
- Say which errors can be retried and which cannot.

**Common mistake:** listing codes without saying what to do.

---

### 5. Endpoints

**What it's for:** the reference itself.

**Write:** one block per endpoint: method and path, purpose, who can call it, limits, request fields, response, side effects, errors. Generate the tables from OpenAPI, and add the notes a tool can't write.

**Tips:**
- Lead with what is surprising: idempotency, expiry of links, "reading marks it as seen".
- Link each error code back to Section 4.
- Show an example for the common case and for one error.

**Common mistake:** documenting the happy path only.

---

### 6. Events and Webhooks

**What it's for:** tell callers how they learn that something changed.

**Write:** events, payloads, signature check, retry behavior, and how the receiver should answer. If there are none, say so in one line and say how callers should follow changes instead (polling, push).

**Common mistake:** deleting the section, so readers wonder whether webhooks exist.

---

### 7. Changelog and Deprecation

**What it's for:** let callers plan for change.

**Write:** a table of versions with dates and changes, the deprecation policy (notice, overlap period), and what counts as breaking.

**Tips:** add an entry in the same change as the API change. Mark breaking changes clearly.

---

### 8. Support

**Write:** where to ask questions, where to report a mistake in the document, and where to report a production problem.

---

## Part 4: Working With OpenAPI

| Topic | Advice |
|-------|--------|
| Generate, don't hand-write | Use the framework's OpenAPI support. Hand-edited files drift from the code. |
| Check in CI | Regenerate the file in the build and fail if it differs from the committed copy. |
| Fill in the descriptions | A generated file with no descriptions or examples is a schema, not a reference. Add summaries, field descriptions, and examples in the code. |
| Reuse error responses | Define common errors once under `components/responses` and reference them. |
| Keep internal endpoints out | Endpoints only an internal system calls (for example, a vendor's webhook) are documented in the SDD, not in the public file. |
| Render it | Tools such as Swagger UI or Redoc turn the file into a browsable page. Add the hand-written pages around it. |
| Validate it | Run an OpenAPI linter in CI. |
| Version it | The `info.version` field is the API document version. Bump it with every change callers can see. |

---

## Part 5: Quality Checklist

**Accuracy**
- [ ] The reference is built from the OpenAPI file, which is generated from or checked against the code
- [ ] Every example was run against a real environment
- [ ] Limits, lengths, and allowed values match the code, the SDD, and the DRD

**Usability**
- [ ] A new caller can make a successful call from the quick start alone
- [ ] Authentication is explained, including what fails and why
- [ ] Every error code says what the caller should do
- [ ] It says which calls are safe to retry, and how

**Completeness**
- [ ] Rate limits, pagination, and the data window are stated
- [ ] Versioning, breaking-change rules, and deprecation policy are stated
- [ ] Owner, contact, and changelog are present
- [ ] No `[brackets]` or guidance text left over

**Safety**
- [ ] No internal names (tables, queues, internal hosts) or real secrets
- [ ] Internal-only endpoints are left out
- [ ] Examples contain no real personal data

---

## Part 6: Common Mistakes at a Glance

| Mistake | Fix |
|---------|-----|
| Retyping schemas by hand | Generate them from OpenAPI |
| Copying the API section into the SDD | Link to the reference from the SDD |
| Quick start that doesn't work | Run it against staging before publishing |
| Errors without actions | Add "what the caller should do" to every row |
| No retry guidance | State what is safe to retry and which key to reuse |
| Describing internals | Describe what callers can rely on |
| Changing the API and not the doc | Update both in the same change |
| No changelog | Add an entry for every change callers can see |
| Placeholder-only examples (`"string"`) | Use realistic values |

---

## Part 7: Keeping the API Reference Useful

An API reference earns its keep when a caller can integrate without asking you a single question, and trusts what they read.

- **Update with the API.** Change the reference in the same pull request as the API.
- **Test it.** Run the examples in CI or before each release.
- **Watch the questions.** Every question a caller asks is a gap in the document. Fix the document, not just the answer.
- **Announce changes.** Use the changelog and release notes. For breaking changes, give the overlap period you promised.
- **Retire cleanly.** Mark a version Deprecated before removing it, and remove it only after traffic has stopped.
