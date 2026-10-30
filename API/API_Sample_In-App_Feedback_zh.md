# LearnSphere 反馈 API — API 参考文档

> **示例文档。**这是 `API_Template_zh.md` 的一份虚构、完整填写的示例。它是**反馈团队之外的人**阅读的参考文档：App 和网页开发者、LearnSphere 其他团队以及 QA。它由 [openapi_Sample_In-App_Feedback.yaml](openapi_Sample_In-App_Feedback.yaml) 生成，并与 [PRD](../PRD/PRD_Sample_In-App_Feedback_zh.md)、[DRD](../DRD/DRD_Sample_In-App_Feedback_zh.md)、[TRD](../TRD/TRD_Sample_In-App_Feedback_zh.md) 和 [SDD](../SDD/SDD_Sample_In-App_Feedback_zh.md) 示例相匹配，因此产品（"LearnSphere"）、限制、名称和错误码均保持一致。公司、URL 和人物均为虚构。产品界面文案（如 "You've reached today's limit"）保持英文原样，因为该产品首发仅支持英文。
>
> 它说明的是 **API 做什么、怎么调用**。它是怎么构建的（队列、重试、Zendesk、表）写在 SDD 里，这里不重复。

| | |
|---|---|
| **API** | LearnSphere 反馈 API |
| **API 版本** | `v1`（文档版本 1.0.0） |
| **基础 URL** | 生产环境：`https://api.learnsphere.example/v1`<br>预发布环境：`https://api.staging.learnsphere.example/v1` |
| **状态** | 预览版。尚未发布；内部试用计划在项目第 6 周进行。 |
| **负责人** | 反馈服务团队（工程部），`feedback-api@learnsphere.example` |
| **最后更新** | 2026-10-30 |
| **唯一事实来源** | [openapi_Sample_In-App_Feedback.yaml](openapi_Sample_In-App_Feedback.yaml)。本页与该文件不一致时，以该文件为准。 |
| **设计文档** | [PRD](../PRD/PRD_Sample_In-App_Feedback_zh.md) v1.0 · [DRD](../DRD/DRD_Sample_In-App_Feedback_zh.md) v1.0 · [TRD](../TRD/TRD_Sample_In-App_Feedback_zh.md) v1.1 · [SDD](../SDD/SDD_Sample_In-App_Feedback_zh.md) v0.3 |

---

## 1. 概述

### 1.1 API 做什么

反馈 API 让已登录的学员可以：

1. 为反馈**添加截图**（最多 3 张）。
2. 就应用中的问题**提交反馈**。
3. **查看自己的反馈**以及 LearnSphere 客服的回复。
4. **评价问题是否解决**。选择"否"会重新打开该反馈。

每条反馈都会得到一个参考编号，例如 `FB-20261030-0042`。反馈被接受后不久，客服会以工单的形式收到它。

### 1.2 适用对象

| 读者 | 典型用途 |
|--------|-------------|
| 移动端和网页开发者 | 构建反馈表单和"My Feedback"页面 |
| LearnSphere 其他团队 | 读取反馈状态（例如未来的帮助中心） |
| QA | 测试各种流程和错误情况 |

### 1.3 快速开始

提交一条带一张截图的反馈。你需要一个学员访问令牌（见[第 2 章](#2-认证)）。

**第 1 步：申请上传表单**（如果没有截图，跳过第 1、2 步）：

```bash
curl -X POST https://api.learnsphere.example/v1/feedback/attachments/presign \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "contentType": "image/png", "sizeBytes": 2048000 }'
```

响应会给出 `attachmentKey` 和一个 `upload` 表单（`url` 和 `fields`）。

**第 2 步：上传文件。**向 `upload.url` 发送 multipart 表单。先放入 `upload.fields` 的每一项（原样不变），文件放在最后：

```bash
curl -X POST "$UPLOAD_URL" \
  -F "key=…" -F "policy=…" -F "x-amz-signature=…" \
  -F "file=@screenshot.png"
```

**第 3 步：提交反馈：**

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

响应 `201`：

```json
{ "id": "b1c7e5d2-8a40-4c3e-9f11-2a6d5e0c7b33", "reference": "FB-20261030-0042", "status": "received" }
```

---

## 2. 认证

每个调用都需要在 `Authorization` 头中带上学员访问令牌：

```text
Authorization: Bearer <token>
```

| 项目 | 规则 |
|------|------|
| 令牌从哪里来 | LearnSphere 登录流程（Auth 服务）。反馈 API 不签发令牌。 |
| 谁可以提交反馈 | **仅付费学员。**非付费套餐的令牌调用 `POST /feedback` 会得到 `403 AUTH-002`。 |
| 谁可以读取反馈 | 只有提交它的学员。其他人会得到 `404` 而不是 `403`，因此无法猜测 ID。 |
| 令牌过期或无效 | `401 AUTH-001`。请让学员重新登录，并保留他们的草稿。 |

---

## 3. 通用约定

| 主题 | 规则 |
|-------|------|
| 格式 | 请求和响应都使用 JSON。请发送 `Content-Type: application/json`。 |
| 版本 | 主版本号在 URL 中（`/v1`）。响应随时可能新增字段，所以**请忽略不认识的字段**。破坏性变更需要 `/v2`，此后 `/v1` 至少继续可用 6 个月，因为旧版 App 仍在使用。 |
| 标识符 | 反馈和附件的 ID 是 UUID。参考编号（`FB-YYYYMMDD-NNNN`，UTC 日期）用于展示给用户以及向客服提及。 |
| 时间戳 | UTC 的 ISO 8601 格式，例如 `2026-10-30T14:05:11Z`。 |
| 状态值 | `received`（显示为 Received）、`in_progress`（In progress）、`resolved`（Resolved）。 |
| 问题类型 | `video_audio`、`quiz_grading`、`course_content`、`billing`、`app_bug`、`other`。 |
| 重试 | `POST /feedback` 使用相同的 `idempotencyKey` 重试是安全的（见 5.2）。其他 `POST` 调用不是幂等的。 |
| 分页 | 基于游标，每页 20 条，最新的在前。把 `nextCursor` 作为 `cursor` 传入。最后一页的 `nextCursor` 为 `null`。 |
| 历史窗口 | 列表和详情覆盖**最近 12 个月**。更早的反馈会被删除。 |

### 3.1 速率限制

限制按学员计算。超过限制会返回 `429 RATE-001`。

| 端点 | 限制 |
|----------|-------|
| `POST /feedback/attachments/presign` | 每小时 30 次 |
| `POST /feedback` | 每 24 小时 10 条反馈 |
| `GET /feedback` | 每分钟 60 次 |
| `GET /feedback/{id}` | 每分钟 60 次 |
| `POST /feedback/{id}/rating` | 每分钟 10 次 |

---

## 4. 错误

错误使用 HTTP 状态码和如下 JSON 响应体：

```json
{
  "code": "VAL-001",
  "messageKey": "feedback.validation.description",
  "details": { "field": "description" }
}
```

| 字段 | 含义 |
|-------|---------|
| `code` | 稳定的错误码，格式为 `CATEGORY-NNN`。**请在代码中使用它。** |
| `messageKey` | 面向学员的文案所对应的 key。App 把它映射成自己的文案和翻译；API 不发送展示用文字。 |
| `details` | 可选。对于 `VAL-001`，`details.field` 指明哪个字段未通过校验。 |

### 4.1 错误码

| HTTP | 错误码 | 含义 | 客户端应该怎么做 |
|------|------|---------|----------------------------|
| 400 | `VAL-001` | 字段缺失或无效。 | 显示 `details.field` 对应的提示。保留学员已输入的内容。 |
| 400 | `VAL-002` | 截图缺失、类型错误、不属于该学员、超过 3 张或超过 5 MB。 | 显示 "Image is too large (max 5 MB)" 或类型提示。 |
| 401 | `AUTH-001` | 令牌缺失、过期或无效。 | 显示登录弹层，然后恢复草稿。 |
| 403 | `AUTH-002` | 该学员不是付费套餐。 | 显示通用错误。这类学员本应看不到入口。 |
| 404 | `NOTFOUND-001` | 没有这条反馈，或它不属于该学员。 | 显示"未找到"状态。 |
| 409 | `STATE-001` | 不允许评价：反馈尚未解决，或本次解决已评价过。 | 隐藏评价卡片。 |
| 429 | `RATE-001` | 达到请求限制（见 3.1）。 | 对于提交反馈："You've reached today's limit. Please try again tomorrow or email support." |
| 503 | `SYS-001` | 服务暂时不可用。 | 保存草稿并重试（见 4.2）。 |

### 4.2 何时重试

| 结果 | 是否重试？ |
|--------|--------|
| 网络错误、超时或 `503 SYS-001` | **是**，并使用**相同的** `idempotencyKey`。建议依次等待 15 秒、30 秒、60 秒、2 分钟、5 分钟，之后保留草稿并请学员稍后再试。 |
| `401 AUTH-001` | 重新登录之后重试。 |
| `429 RATE-001` | 否。等待窗口过去。 |
| 其他 `4xx` | 否。修正请求。 |

---

## 5. 端点

所有路径都相对于基础 URL。所有调用都需要 `Authorization` 头。

| 方法 | 路径 | 用途 |
|--------|------|---------|
| POST | `/feedback/attachments/presign` | 获取截图的一次性上传表单 |
| POST | `/feedback` | 提交反馈 |
| GET | `/feedback` | 列出你的反馈 |
| GET | `/feedback/{id}` | 读取一条反馈及其回复 |
| POST | `/feedback/{id}/rating` | 评价问题是否解决 |

### 5.1 获取截图上传表单

`POST /feedback/attachments/presign`

为**一个** PNG 或 JPEG 文件返回预签名上传表单。表单 5 分钟后过期，过期后请重新申请。

**请求体**

| 字段 | 类型 | 必填 | 规则 |
|-------|------|----------|-------|
| `contentType` | string | 是 | `image/png` 或 `image/jpeg` |
| `sizeBytes` | integer | 是 | 1 到 5,242,880（5 MB） |

**响应 `200`**

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

保存 `attachmentKey`，创建反馈时把它放进 `attachmentKeys`。上传表单固定了文件类型和大小，因此其他类型或更大的文件会在上传环节被直接拒绝。

**错误：**`VAL-002`（内容类型错误）、`AUTH-001`、`RATE-001`、`SYS-001`。

### 5.2 提交反馈

`POST /feedback`

创建反馈并返回它的参考编号。仅限付费学员。限制：每 24 小时 10 条反馈。

**可以安全重试。**表单打开时就生成 `idempotencyKey`，与草稿一起保存，每次重试都发送同一个值。如果之前的调用已经成功，你会再次收到**原来的** `201` 响应，且不会创建第二条反馈。对已被接受的反馈进行重试，永远不会被"每天 10 条"的限制拦住。

**请求体**

| 字段 | 类型 | 必填 | 规则 |
|-------|------|----------|-------|
| `idempotencyKey` | string（UUID v4） | 是 | 每条反馈一个。重试时复用。 |
| `issueType` | string | 是 | 第 3 章中 6 种问题类型之一 |
| `description` | string | 是 | 去除首尾空格后 10 到 1,000 个字符 |
| `replyEmail` | string | 是 | 有效的邮箱，最多 254 个字符 |
| `attachmentKeys` | string[] | 否 | 最多 3 个。每个都必须是你从 5.1 为该学员获得的 key。 |
| `context.deviceModel`、`osVersion`、`appVersion`、`locale` | string | 是 | 不能为空 |
| `context.clientSubmittedAt` | string | 是 | 学员按下发送键的 ISO 8601 时间 |
| `context.courseId`、`context.lessonId` | string | 否 | 要么都传，要么都不传 |

**响应 `201`**

| 字段 | 类型 | 含义 |
|-------|------|---------|
| `id` | string（UUID） | 用它来读取这条反馈 |
| `reference` | string | 展示给学员，例如 `FB-20261030-0042` |
| `status` | string | 新反馈始终为 `received` |

**接下来会发生什么：**客服会在几分钟内以工单形式收到这条反馈。学员会通过应用内通知、推送，以及（除非已关闭）邮件收到确认，通常在 5 分钟内。API 不会等待这些步骤完成。

**错误：**`VAL-001`、`VAL-002`、`AUTH-001`、`AUTH-002`、`RATE-001`、`SYS-001`。

### 5.3 列出你的反馈

`GET /feedback?cursor={cursor}`

返回学员最近 12 个月的反馈，最新的在前，每页 20 条。

**查询参数**

| 名称 | 必填 | 含义 |
|------|----------|---------|
| `cursor` | 否 | 上一页的 `nextCursor`。获取第一页时不传。 |

**响应 `200`**

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

当反馈自学员上次打开后发生了变化（有新回复或状态变化）时，`hasUnreadUpdate` 为 `true`。用它来显示未读小红点。

**错误：**`AUTH-001`、`RATE-001`、`SYS-001`。

### 5.4 读取一条反馈

`GET /feedback/{id}`

返回列表项中的全部字段，外加描述、截图、回复和学员的评价。

**响应 `200`**

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

| 字段 | 说明 |
|-------|-------|
| `attachments[].url` | 有效期 **5 分钟**的签名链接。不要保存它；需要时重新读取反馈以获得新链接。 |
| `replies` | 客服的公开回复，最早的在前。内部备注永远不会包含在内。 |
| `rating` | 学员对当前这次解决的评价，或 `null`。 |

**副作用：**读取一条反馈会把它的更新标记为已读，因此 `hasUnreadUpdate` 变为 `false`。

**错误：**`NOTFOUND-001`（别人的反馈和超过 12 个月的反馈也返回它）、`AUTH-001`、`RATE-001`、`SYS-001`。

### 5.5 评价解决结果

`POST /feedback/{id}/rating`

让学员说明回复是否有帮助。**每次解决只能评价一次**，并且只在反馈处于 `resolved` 时允许。

**请求体**

| 字段 | 类型 | 必填 | 规则 |
|-------|------|----------|-------|
| `helped` | boolean | 是 | `true` 或 `false` |
| `comment` | string | 否 | 最多 300 个字符 |

**响应 `200`**

| `helped` | 响应中的 `status` | `messageKey` | 会发生什么 |
|----------|--------------------------|--------------|--------------|
| `true` | `resolved` | `feedback.rating.thanks` | 没有后续。 |
| `false` | `in_progress` | `feedback.rating.reopened` | 反馈被重新打开，客服重新接手。 |

如果被重新打开的反馈再次被解决，学员可以再次评价。

**错误：**`VAL-001`（例如评论超过 300 个字符）、`NOTFOUND-001`、`STATE-001`（未解决，或已评价）、`AUTH-001`、`RATE-001`、`SYS-001`。

---

## 6. 事件与 Webhook

第 1 版**不会**向外部调用方发送 webhook。要跟踪一条反馈：

- 展示学员在客服回复或解决反馈时收到的应用内通知或推送，并通过其深度链接 `learnsphere://feedback/{id}` 打开该反馈。
- 或者刷新 `GET /feedback`，检查 `hasUnreadUpdate`。

服务也会接收来自客服工具的事件。该端点仅限内部使用，记录在 SDD 中，不在此处说明。

---

## 7. 变更日志与弃用

| 版本 | 日期 | 变更 |
|---------|------|--------|
| 1.0.0 | 计划在内部试用时发布（项目第 6 周） | 首次发布：截图、反馈、列表和详情、评价。 |

**弃用政策：**我们会在发布说明和本表中公布弃用信息，并让旧版本至少继续运行 6 个月。移除某个版本之前，我们会检查它还剩多少流量。

**变更规则：**新增响应字段、新增可选请求字段或新增端点**不算**破坏性变更。删除或重命名字段、更改类型、把可选字段改为必填则**算**破坏性变更，需要新的主版本。

---

## 8. 支持

| 需求 | 联系方式 |
|------|---------|
| 关于如何使用 API 的问题 | `feedback-api@learnsphere.example` |
| 发现本文档或 OpenAPI 文件有错误 | 在 `learnsphere/feedback-service` 仓库提交 pull request |
| 生产环境问题 | 呼叫反馈服务值班人员（见 `runbooks/feedback-api.md`） |
