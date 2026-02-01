# `/api-call` 代请求接口：调用参数与响应解析（含配额查询封装逻辑）

本文档整理本 WebUI 工程中对管理端 **`POST /api-call`** 的调用方式、请求/响应结构、错误信息提取规则，以及在配额（quota）功能里对 `/api-call` 的“包裹获取逻辑”（重试/解析/映射）。

> 关键代码入口：`src/services/api/apiCall.ts`、`src/components/quota/quotaConfigs.ts`。

---

## 1. `/api-call` 的定位与作用

- 本前端不会直接携带真实第三方凭证去请求外部站点（例如 Google / ChatGPT 等）。
- 前端将「目标 URL + method + headers + body + authIndex」打包发送给管理端（Management API），由管理端完成：
  1) 选择认证文件/凭证（通过 `authIndex`）
  2) 替换请求头里的占位符（例如 `Authorization: Bearer $TOKEN$`）
  3) 实际向外部发起 HTTP 请求
  4) 将结果以统一格式返回给前端

前端侧封装：`src/services/api/apiCall.ts:69-86`。

---

## 2. 请求路径与基础认证（管理端）

### 2.1 管理端 baseURL 规范化

`src/services/api/client.ts:48-63` 会把用户配置的 `apiBase` 规范化，并拼上 `MANAGEMENT_API_PREFIX`。

- 会移除用户输入里尾部的 `/v0/management`（避免重复）
- 会补全协议（默认 `http://`）
- 最终所有 `apiClient.*` 请求都会以 `this.apiBase` 为 baseURL

### 2.2 管理端认证头（managementKey）

`src/services/api/client.ts:106-120` 请求拦截器里会自动加：

- `Authorization: Bearer ${managementKey}`

> 注意：这与外部目标站点的 `Authorization` 不同；外部目标站点的 Authorization 会被放在 `/api-call` 的 payload 里，由管理端二次处理。

---

## 3. `/api-call` 请求参数（payload）结构

类型定义：`src/services/api/apiCall.ts:8-14`

```ts
export interface ApiCallRequest {
  authIndex?: string;
  method: string;
  url: string;
  header?: Record<string, string>;
  data?: string;
}
```

字段说明：

- `authIndex?: string`
  - 可选；配额查询里必传。
  - 用于指示管理端“使用第几个认证文件/凭证”去对外请求。
  - 前端侧来源：认证文件列表项 `AuthFileItem` 上的 `auth_index` 或 `authIndex`（例如 `src/components/quota/quotaConfigs.ts:128-132`）。

- `method: string`
  - 目标外部请求的方法，如 `GET` / `POST`。

- `url: string`
  - 目标外部请求的完整 URL。

- `header?: Record<string, string>`
  - 目标外部请求的请求头。
  - 工程里常见模式是把外部 token 写成占位符：`Authorization: 'Bearer $TOKEN$'`（例如 `src/utils/quota/constants.ts:62-66`），由管理端替换。

- `data?: string`
  - 目标外部请求的 body，**以字符串形式**传递（通常是 `JSON.stringify(...)`）。
  - 例如 `src/components/quota/quotaConfigs.ts:135` 会构造两个 body 版本以兼容不同字段名。

### 3.1 前端实际发送

调用实现：`src/services/api/apiCall.ts:70-78`

```ts
const response = await apiClient.post('/api-call', payload, config);
```

也就是：向 **管理端**的 `/api-call` 发送一个 JSON payload；管理端再去访问 `payload.url`。

---

## 4. `/api-call` 响应结构与解析逻辑

### 4.1 前端归一化后的返回值：ApiCallResult

类型定义：`src/services/api/apiCall.ts:16-21`

```ts
export interface ApiCallResult<T = any> {
  statusCode: number;
  header: Record<string, string[]>;
  bodyText: string;
  body: T | null;
}
```

### 4.2 管理端返回字段映射

`src/services/api/apiCall.ts:74-84`：

- `statusCode`：优先取 `response.status_code`，其次 `response.statusCode`
- `header`：优先取 `response.header`，其次 `response.headers`
- `bodyText/body`：来自 `response.body`，并通过 `normalizeBody(...)` 归一化

### 4.3 `normalizeBody` 行为（重要）

实现：`src/services/api/apiCall.ts:23-46`

- 若 `response.body` 为 `string`：
  - `trim()` 后能 `JSON.parse` → `body` 变成对象，`bodyText` 保留原始字符串
  - 不能 parse → `body` 仍是字符串
- 若 `response.body` 为对象：
  - 尝试 `JSON.stringify` 写入 `bodyText`
  - `body` 保留原对象
- 若为 `null/undefined`：
  - `bodyText=''`，`body=null`

> 因此：业务层解析时通常优先用 `result.body`，同时用 `result.bodyText` 兜底（例如 `parseXxx(result.body ?? result.bodyText)`）。

---

## 5. 错误消息提取规则（统一）

工具函数：`src/services/api/apiCall.ts:48-67`

```ts
export const getApiCallErrorMessage = (result: ApiCallResult): string => {
  // 优先从 body.error.message / body.error / body.message 取
  // 再退化到 bodyText
  // 最后拼接 status
}
```

特性：

- 当 `body` 是对象时：优先取 `body.error.message` → `body.error` → `body.message`
- 当 `body` 是字符串时：直接用字符串
- 若 `message` 为空但有 `bodyText`：使用 `bodyText`
- 若有 `statusCode` 且有 `message`：返回 `"${status} ${message}"`

在配额查询中用于构造最终的错误提示与重试决策：
- `src/components/quota/quotaConfigs.ts:153-166`

---

## 6. 配额查询对 `/api-call` 的“包裹获取逻辑”

配额查询并不直接调用 `/api-call`，而是通过 QuotaConfig 的 `fetchQuota` 进行封装。

核心文件：`src/components/quota/quotaConfigs.ts`

### 6.1 Antigravity 配额查询（fetchAvailableModels）

入口函数：`src/components/quota/quotaConfigs.ts:124-202`

#### 6.1.1 请求参数构造

- `authIndex`：`src/components/quota/quotaConfigs.ts:128-132`
- `projectId`：通过解析认证文件内容获得，失败则使用默认值
  - 解析逻辑：`src/components/quota/quotaConfigs.ts:87-117`
  - 默认值：`DEFAULT_ANTIGRAVITY_PROJECT_ID`（`src/components/quota/quotaConfigs.ts:58`）
- `url`：依次尝试 `ANTIGRAVITY_QUOTA_URLS`（`src/utils/quota/constants.ts:56-60`）
- `header`：`ANTIGRAVITY_REQUEST_HEADERS`（`src/utils/quota/constants.ts:62-66`）
- `data`：会尝试两种 JSON body（字段名兼容）：
  - `{"projectId":"..."}`
  - `{"project":"..."}`
  - 构造位置：`src/components/quota/quotaConfigs.ts:135`

#### 6.1.2 重试策略（两层循环）

`src/components/quota/quotaConfigs.ts:142-195`：

- 外层：遍历多个候选 URL（环境/域名差异）
- 内层：对每个 URL，最多尝试两种 body
- 如果 statusCode=400 且错误内容类似“unknown name / cannot find field”，说明 body 字段名不被接受：
  - 识别函数：`isAntigravityUnknownFieldError`（`src/components/quota/quotaConfigs.ts:119-122`）
  - 会切到第二种 body 继续尝试（`src/components/quota/quotaConfigs.ts:159-165`）

#### 6.1.3 成功响应解析

成功条件：`result.statusCode` 在 `[200, 300)`（`src/components/quota/quotaConfigs.ts:153-168`）。

解析步骤：

1) 解析 payload：
   - `parseAntigravityPayload(result.body ?? result.bodyText)`
   - 代码：`src/components/quota/quotaConfigs.ts:169-170`
   - parser：`src/utils/quota/parsers.ts:104-118`

2) 取模型表：
   - `payload.models` 必须是 object 且非数组（`src/components/quota/quotaConfigs.ts:171-175`）

3) 构建可展示分组：
   - `buildAntigravityQuotaGroups(models)`（`src/components/quota/quotaConfigs.ts:177`）
   - 分组定义：`ANTIGRAVITY_QUOTA_GROUPS`（`src/utils/quota/constants.ts:68-89`）
   - builder：`src/utils/quota/builders.ts:158-209`

#### 6.1.4 配额字段映射（remainingFraction 等）

`buildAntigravityQuotaGroups` 内部会对每个命中的 model entry 提取 quotaInfo：

- `getAntigravityQuotaInfo`：`src/utils/quota/builders.ts:115-136`
- quotaInfo 兼容字段：
  - `entry.quotaInfo` / `entry.quota_info`
  - `remainingFraction` / `remaining_fraction` / `remaining`
  - `resetTime` / `reset_time`

`remainingFraction` 归一化：
- `normalizeQuotaFraction`：`src/utils/quota/parsers.ts:40-52`
- 支持数字或百分号字符串（例如 `"37%"` → `0.37`）

> UI 展示通常会把 `remainingFraction`（0~1）乘 100 作为百分比；渲染侧逻辑在 `renderAntigravityItems`（`src/components/quota/quotaConfigs.ts:371-387`）。

---

### 6.2 Codex 配额查询（wham/usage）

入口函数：`fetchCodexQuota`（`src/components/quota/quotaConfigs.ts:255-295`）

`/api-call` payload 特点：

- `method: 'GET'`
- `url: CODEX_USAGE_URL`（`src/utils/quota/constants.ts:124`）
- `header`：`CODEX_REQUEST_HEADERS`（`src/utils/quota/constants.ts:126-130`）并额外加入：
  - `Chatgpt-Account-Id`（`src/components/quota/quotaConfigs.ts:271-274`）

响应解析：

- 成功条件：`statusCode` 在 `[200,300)`，否则抛错（`src/components/quota/quotaConfigs.ts:283-285`）
- payload：`parseCodexUsagePayload(result.body ?? result.bodyText)`（`src/components/quota/quotaConfigs.ts:287`，parser 在 `src/utils/quota/parsers.ts:121-136`）
- 组装可展示窗口：`buildCodexQuotaWindows(payload, t)`（`src/components/quota/quotaConfigs.ts:204-253`）

---

### 6.3 Gemini CLI 配额查询（retrieveUserQuota）

入口函数：`fetchGeminiCliQuota`（`src/components/quota/quotaConfigs.ts:297-356`）

`/api-call` payload：

- `method: 'POST'`
- `url: GEMINI_CLI_QUOTA_URL`（`src/utils/quota/constants.ts:92-93`）
- `header: GEMINI_CLI_REQUEST_HEADERS`（`src/utils/quota/constants.ts:95-98`）
- `data: JSON.stringify({ project: projectId })`（`src/components/quota/quotaConfigs.ts:317`）

响应解析：

- payload：`parseGeminiCliQuotaPayload(result.body ?? result.bodyText)`（`src/components/quota/quotaConfigs.ts:324`，parser 在 `src/utils/quota/parsers.ts:138-153`）
- 取 `payload.buckets` 数组（`src/components/quota/quotaConfigs.ts:325-326`）
- 每个 bucket 解析字段（兼容 snake_case）：
  - `modelId/model_id`、`tokenType/token_type`
  - `remainingFraction/remaining_fraction`
  - `remainingAmount/remaining_amount`
  - `resetTime/reset_time`
  - 代码：`src/components/quota/quotaConfigs.ts:328-355`

`remainingFraction` 的兜底规则：

- 若后端未提供 `remainingFraction`，但提供 `remainingAmount`：
  - `remainingAmount <= 0` → `remainingFraction = 0`
- 若仅提供 `resetTime`：
  - 认为已耗尽 → `remainingFraction = 0`

最终会进入 `buildGeminiCliQuotaBuckets` 做分组聚合（`src/utils/quota/builders.ts:33-113`）。

---

## 7. 典型调用示例（前端侧）

下面以 Antigravity 为例，展示前端传给 `/api-call` 的 payload 形态（字段名与真实代码一致）：

```json
{
  "authIndex": "1",
  "method": "POST",
  "url": "https://cloudcode-pa.googleapis.com/v1internal:fetchAvailableModels",
  "header": {
    "Authorization": "Bearer $TOKEN$",
    "Content-Type": "application/json",
    "User-Agent": "antigravity/1.11.5 windows/amd64"
  },
  "data": "{\"projectId\":\"bamboo-precept-lgxtn\"}"
}
```

对应发起位置：`src/components/quota/quotaConfigs.ts:145-151`。

---

## 8. 注意事项与边界说明

1) **前端不负责替换 `$TOKEN$`**
- 例如 `ANTIGRAVITY_REQUEST_HEADERS.Authorization = 'Bearer $TOKEN$'`（`src/utils/quota/constants.ts:62-66`）。
- 真实 token 的注入与外部请求由管理端完成。

2) **`data` 必须是字符串**
- `ApiCallRequest.data?: string`（`src/services/api/apiCall.ts:13-14`）。
- 所以业务层会 `JSON.stringify(...)`。

3) **解析时优先使用 `result.body`，并以 `result.bodyText` 兜底**
- 因为 `normalizeBody` 可能把无法 JSON parse 的内容保留为字符串。

4) **管理端 401 会触发登出事件**
- `src/services/api/client.ts:161-165`。

5) **配额 UI 对 403/404 有特定提示**
- 403：提示检查凭证
- 404：提示需要升级配额查询能力（例如服务端版本不支持）
- 逻辑在 `QuotaCard`：`src/components/quota/QuotaCard.tsx:137-145`
