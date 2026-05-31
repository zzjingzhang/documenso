# DOCUMENT_SENT Webhook 全链路分析

## 1. 端到端流程概览

当 `DOCUMENT_SENT` 事件触发时，数据流经以下六个阶段：

```
业务代码调用 triggerWebhook()
  → 1. 查询匹配的 Webhook 记录
  → 2. 排入 internal.execute-webhook 后台任务
  → 3. 任务调度器内部 HTTP 回调执行 handler
  → 4. handler 执行 SSRF 校验 → POST 请求 → 记录 WebhookCall
  → 5. 失败时 throw → 任务级重试
  → 6. 超过 maxRetries 后任务标记 FAILED
```

---

## 2. 阶段一：查找 enabled 且 eventTriggers 包含 DOCUMENT_SENT 的 Webhook

**入口**：[trigger-webhook.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/webhooks/trigger/trigger-webhook.ts#L13-L36)

```ts
const registeredWebhooks = await getAllWebhooksByEventTrigger({ event, userId, teamId });
```

**查询逻辑**：[get-all-webhooks-by-event-trigger.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/webhooks/get-all-webhooks-by-event-trigger.ts#L12-L24)

```ts
prisma.webhook.findMany({
  where: {
    enabled: true,              // 仅启用状态
    eventTriggers: { has: event }, // 数组字段包含目标事件
    team: buildTeamWhereQuery({ teamId, userId }),
  },
});
```

关键点：
- `enabled: true` 硬性过滤，未启用的 webhook 不会被触发。
- `eventTriggers` 是 Prisma 的 `has` 过滤器，对应 PostgreSQL 数组包含操作，只有 `eventTriggers` 数组中包含 `DOCUMENT_SENT` 的记录才返回。
- `team` 关联查询确保调用者有权限操作该团队。

---

## 3. 阶段二：排入 `internal.execute-webhook` 后台任务

**触发**：[trigger-webhook.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/webhooks/trigger/trigger-webhook.ts#L21-L30)

```ts
await Promise.allSettled(
  registeredWebhooks.map(async (webhook) => {
    await jobs.triggerJob({
      name: 'internal.execute-webhook',
      payload: { event, webhookId: webhook.id, data },
    });
  }),
);
```

- 使用 `Promise.allSettled` 而非 `Promise.all`：即使某个 webhook 入队失败，其他 webhook 仍会继续处理。
- 每个 webhook 独立产生一个 job，互不阻塞。

**Job 定义**：[execute-webhook.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/execute-webhook.ts#L1-L31)

```ts
const EXECUTE_WEBHOOK_JOB_DEFINITION = {
  id: 'internal.execute-webhook',
  name: 'Execute Webhook',
  version: '1.0.0',
  trigger: {
    name: 'internal.execute-webhook',
    schema: EXECUTE_WEBHOOK_JOB_DEFINITION_SCHEMA, // Zod schema 验证 payload
  },
  handler: async ({ payload, io }) => { ... },
};
```

**入队过程**（`LocalJobProvider.triggerJob`）：[local.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/client/local.ts#L193-L218)

```ts
public async triggerJob(options: SimpleTriggerJobOptions) {
  const eligibleJobs = Object.values(this._jobDefinitions)
    .filter((job) => job.trigger.name === options.name);

  await Promise.all(eligibleJobs.map(async (job) => {
    const pendingJob = await prisma.backgroundJob.create({
      data: {
        jobId: job.id,
        name: job.name,
        version: job.version,
        payload: options.payload as Prisma.InputJsonValue,
      },
    });
    // status 默认 PENDING, maxRetries 默认 3

    await this.submitJobToEndpoint({
      jobId: pendingJob.id,
      jobDefinitionId: pendingJob.jobId,
      data: options,
      isRetry: false,
    });
  }));
}
```

- 创建 `BackgroundJob` 记录，初始状态 `PENDING`，`retried = 0`，`maxRetries = 3`（Prisma 默认值）。
- 立即通过 `submitJobToEndpoint` 发起内部 HTTP POST 调用。

---

## 4. 阶段三：内部 HTTP 调度 → 执行 Handler

**提交到内部端点**：[local.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/client/local.ts#L353-L385)

```ts
private async submitJobToEndpoint(options) {
  const endpoint = `${NEXT_PRIVATE_INTERNAL_WEBAPP_URL()}/api/jobs/${jobDefinitionId}/${jobId}`;
  const signature = sign(data);

  const headers = {
    'Content-Type': 'application/json',
    'X-Job-Id': jobId,
    'X-Job-Signature': signature,
  };

  if (isRetry) headers['X-Job-Retry'] = '1';

  await Promise.race([
    fetch(endpoint, { method: 'POST', body: JSON.stringify(data), headers }).catch(() => null),
    new Promise((resolve) => setTimeout(resolve, 150)),
  ]);
}
```

**签名机制 (`X-Job-Signature`)**：

1. [sign()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/crypto/sign.ts#L1-L10)：将 `data` JSON 序列化 → 哈希 → 加密，生成签名。
2. 接收端 [getApiHandler](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/client/local.ts#L259-L261) 用 `verify(options, signature)` 校验，防止伪造请求。

**API Handler 端处理**：[local.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/client/local.ts#L220-L351)

核心状态流转：

```
PENDING → PROCESSING → COMPLETED (成功)
                     → PENDING    (失败，未超重试次数，重新提交)
                     → FAILED     (超过 maxRetries 或 task 超重试)
```

1. CAS 更新：`update where { id, status: PENDING } → PROCESSING`，确保只有一次执行。
2. 调用 `definition.handler({ payload, io })`。
3. 成功：`PROCESSING → COMPLETED`。
4. 失败：
   - 如果 `retried >= maxRetries`（默认 3 次）或 `task.retried >= 3`：`PROCESSING → FAILED`。
   - 否则：`PROCESSING → PENDING`，然后重新调用 `submitJobToEndpoint(isRetry: true)`。

---

## 5. 阶段四：Handler 执行 — SSRF 校验 → POST 请求 → 记录 WebhookCall

**Handler**：[execute-webhook.handler.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/execute-webhook.handler.ts#L1-L46)

```ts
export const run = async ({ payload, io: _io }) => {
  const { event, webhookId, data } = payload;

  const webhook = await prisma.webhook.findUniqueOrThrow({ where: { id: webhookId } });
  const { webhookUrl: url, secret } = webhook;

  const payloadData = {
    event,
    payload: data,
    createdAt: new Date().toISOString(),
    webhookEndpoint: url,
  };

  const result = await executeWebhookCall({ url, body: payloadData, secret });

  await prisma.webhookCall.create({
    data: {
      url,
      event,
      status: result.success ? WebhookCallStatus.SUCCESS : WebhookCallStatus.FAILED,
      requestBody: payloadData as Prisma.InputJsonValue,
      responseCode: result.responseCode,
      responseBody: result.responseBody,
      responseHeaders: result.responseHeaders,
      webhookId: webhook.id,
    },
  });

  if (!result.success) {
    throw new Error(`Webhook execution failed with status ${result.responseCode}`);
  }
};
```

关键设计：**无论成功或失败，都先创建 `WebhookCall` 记录**，然后在失败时 throw 触发重试。这意味着每次重试都会产生一条新的 `WebhookCall` 记录。

**executeWebhookCall**：[execute-webhook-call.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/webhooks/execute-webhook-call.ts#L1-L59)

```ts
const WEBHOOK_TIMEOUT_MS = 10_000;

export const executeWebhookCall = async ({ url, body, secret }) => {
  try {
    await assertNotPrivateUrl(url);  // 运行时 DNS 解析 SSRF 校验

    const response = await fetchWithTimeout(url, {
      method: 'POST',
      body: JSON.stringify(body),
      redirect: 'manual',            // 不自动跟随重定向
      timeoutMs: WEBHOOK_TIMEOUT_MS, // 10 秒超时
      headers: {
        'Content-Type': 'application/json',
        'X-Documenso-Secret': secret ?? '',
      },
    });

    const text = await response.text();
    return {
      success: response.ok,
      responseCode: response.status,
      responseBody: parseBody(text),
      responseHeaders: Object.fromEntries(response.headers.entries()),
    };
  } catch (err) {
    return {
      success: false,
      responseCode: 0,
      responseBody: err instanceof Error ? err.message : 'Unknown error',
      responseHeaders: {},
    };
  }
};
```

---

## 6. 阶段五：失败时重试机制

重试由两层机制协同工作：

### 6.1 Job 级重试

在 [local.ts getApiHandler](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/client/local.ts#L293-L347) 中：

- handler throw → catch 块检查 `backgroundJob.retried >= backgroundJob.maxRetries`。
- 未超限：状态回退 `PROCESSING → PENDING`，`submitJobToEndpoint(isRetry: true)` 重新提交。
- 超限：`PROCESSING → FAILED`，任务终止。
- `maxRetries` 默认 3，即最多执行 4 次（1 次原始 + 3 次重试）。

### 6.2 Task 级幂等与重试

在 [local.ts createJobRunIO](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/client/local.ts#L387-L467) 中，`io.runTask(cacheKey, callback)` 提供：

- **幂等性**：通过 `cacheKey` 哈希生成 `task-${hash}--${jobId}` 作为 `BackgroundJobTask` 的 ID。
  - 如果 task 已存在且 `status === COMPLETED`，直接返回存储的 `result`，不重新执行。
  - 如果 task 不存在，创建新 task 记录。
- **Task 级重试上限**：`task.retried >= 3` 时抛出 `BackgroundTaskExceededRetriesError`，导致整个 job 标记 FAILED。

> **注意**：当前 `execute-webhook.handler.ts` 中的 `io` 参数被命名为 `_io`（未使用），handler 内部并未使用 `io.runTask`。这意味着 webhook 执行没有 task 级幂等保护——每次重试都会完整执行整个 handler（重新查询 webhook、重新 POST、重新创建 WebhookCall）。

---

## 7. ZWebhookUrlSchema 本地 isPrivateUrl 校验 vs executeWebhookCall 运行时 assertNotPrivateUrl DNS 解析校验

这是整个链路中**最关键的安全差异**。

### 7.1 创建时的同步校验：`ZWebhookUrlSchema` + `isPrivateUrl`

**位置**：[schema.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/webhook-router/schema.ts#L5-L10)

```ts
export const ZWebhookUrlSchema = z
  .string()
  .url()
  .refine((url) => !isPrivateUrl(url), {
    message: 'Webhook URL cannot point to a private or loopback address',
  });
```

**`isPrivateUrl` 实现**：[is-private-url.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/webhooks/is-private-url.ts#L1-L82)

| 特性 | 说明 |
|---|---|
| **校验时机** | 创建/编辑 webhook 时，通过 tRPC schema 验证 |
| **校验方式** | **纯同步字符串解析**，`new URL(url)` 取 hostname |
| **检测范围** | 仅检查 URL 字面量中的 hostname：`localhost`、`127.*`、`10.*`、`192.168.*`、`169.254.*`、`172.16-31.*`、`0.0.0.0`、`::1`、`::`、`fe80:*`、`fc*/fd*`（IPv6 ULA）、`::ffff:x.x.x.x`（IPv4-mapped） |
| **DNS 解析** | ❌ **无**。不解析域名 |
| **局限性** | `evil.example.com` CNAME → `127.0.0.1` **不会**被拦截 |

### 7.2 执行时的异步校验：`assertNotPrivateUrl`

**位置**：[assert-webhook-url.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/webhooks/assert-webhook-url.ts#L79-L130)

```ts
export const assertNotPrivateUrl = async (url, options?) => {
  if (isBypassedHost(url)) return;       // SSRF bypass 白名单

  if (isPrivateUrl(url)) throw ...;      // 第一层：同 isPrivateUrl 的同步检查

  const hostname = normalizeHostname(new URL(url).hostname);

  if (ZIpSchema.safeParse(hostname).success) return; // 纯 IP 已被第一层覆盖

  // 第二层：DNS 解析
  const lookupResult = await withTimeout(
    resolveHostname(hostname, { all: true, verbatim: true }),
    WEBHOOK_DNS_LOOKUP_TIMEOUT_MS,  // 250ms 超时
  );

  if (addresses.some(({ address }) => isPrivateUrl(toAddressUrl(address)))) {
    throw new AppError(AppErrorCode.WEBHOOK_INVALID_REQUEST, {
      message: 'Webhook URL resolves to a private or loopback address',
    });
  }
};
```

| 特性 | 说明 |
|---|---|
| **校验时机** | 每次 `executeWebhookCall` 执行时，POST 请求前 |
| **校验方式** | **异步 DNS 解析** `node:dns/promises.lookup` |
| **检测范围** | 第一层同 `isPrivateUrl` 的同步检查 + 第二层 DNS 解析后对所有 A/AAAA 记录逐个检查 |
| **超时** | DNS 查询 250ms (`WEBHOOK_DNS_LOOKUP_TIMEOUT_MS`)，超时返回 `null`（静默放行） |
| **SSRF Bypass** | `NEXT_PRIVATE_WEBHOOK_SSRF_BYPASS_HOSTS` 环境变量中的主机名跳过所有检查 |
| **DNS 异常容错** | DNS 解析失败（非 AppError）时静默放行 |

### 7.3 差异对照表

| 维度 | `ZWebhookUrlSchema` + `isPrivateUrl` | `assertNotPrivateUrl` |
|---|---|---|
| 触发时机 | 创建/编辑 webhook（tRPC 入参校验） | 每次执行 webhook（运行时） |
| 同步/异步 | 同步 | 异步（含 DNS lookup） |
| DNS 解析 | ❌ | ✅（250ms 超时） |
| 拦截字面私有 IP | ✅ | ✅ |
| 拦截域名解析到私有 IP | ❌ | ✅ |
| Bypass 机制 | 无 | `NEXT_PRIVATE_WEBHOOK_SSRF_BYPASS_HOSTS` |
| DNS 失败时行为 | N/A | **放行**（不抛出错误） |
| DNS 超时时行为 | N/A | **放行**（返回 null） |

---

## 8. 关键概念关系图

### 8.1 `X-Documenso-Secret`

- **用途**：Webhook POST 请求中的身份验证头，值为 webhook 记录中存储的 `secret`。
- **来源**：[execute-webhook-call.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/webhooks/execute-webhook-call.ts#L39) 中 `'X-Documenso-Secret': secret ?? ''`。
- **特性**：如果 webhook 未设置 secret，发送空字符串。接收端通过此 header 验证请求确实来自 Documenso。
- **与重试无关**：secret 不影响重试逻辑，仅用于接收端验证。

### 8.2 `WEBHOOK_TIMEOUT_MS`

- **值**：`10_000`（10 秒）。
- **位置**：[execute-webhook-call.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/webhooks/execute-webhook-call.ts#L6)
- **实现**：通过 [fetchWithTimeout](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/utils/timeout.ts#L17-L34) 使用 `AbortController`，超时后 `abort()` 抛出 `AbortError`，被转换为 `"Request timed out after 10000ms"` 错误。
- **与重试关系**：超时导致 `executeWebhookCall` catch 块返回 `{ success: false, responseCode: 0 }`，handler 记录 FAILED WebhookCall 后 throw，触发 job 重试。

### 8.3 `responseBody` 解析

- **位置**：[execute-webhook-call.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/webhooks/execute-webhook-call.ts#L15-L21)
- **逻辑**：`parseBody(text)` 先尝试 `JSON.parse`，失败则原样返回字符串。
- **存储**：通过 Prisma `Json?` 类型存入 `WebhookCall.responseBody`，可以是 JSON 对象或字符串。
- **错误场景**：当 `executeWebhookCall` 捕获异常时，`responseBody` 为 `err.message`（字符串），`responseCode` 为 `0`。

### 8.4 `WebhookCallStatus`

```
enum WebhookCallStatus { SUCCESS, FAILED }
```

- **SUCCESS**：`response.ok === true`（HTTP 2xx）。
- **FAILED**：`response.ok === false`（非 2xx）或请求异常（超时、DNS 错误、SSRF 拦截等）。
- **与重试关系**：每次重试都会创建一条新的 `WebhookCall` 记录。即使最终 job 成功，之前重试产生的 `FAILED` 记录也会保留。
- **与 `BackgroundJobStatus` 的区别**：`WebhookCallStatus` 描述单次 HTTP 调用的结果；`BackgroundJobStatus` 描述整个后台任务的生命周期。

### 8.5 `BackgroundJobStatus`

```
enum BackgroundJobStatus { PENDING, PROCESSING, COMPLETED, FAILED }
```

状态机：

```
PENDING ──→ PROCESSING ──→ COMPLETED  (handler 成功)
                  │
                  ├──→ PENDING  (handler 失败，retried < maxRetries，重新入队)
                  │
                  └──→ FAILED   (retried >= maxRetries 或 task 超重试)
```

- **PENDING**：初始状态 / 重试回退状态。
- **PROCESSING**：CAS 从 PENDING 更新，确保只有一个消费者处理。
- **COMPLETED**：handler 执行成功。
- **FAILED**：超过最大重试次数，任务终止。

### 8.6 `X-Job-Signature`

- **生成**：[local.ts submitJobToEndpoint](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/client/local.ts#L362) 调用 `sign(data)`，对 job payload 进行 HMAC 签名。
- **验证**：[local.ts getApiHandler](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/client/local.ts#L259-L261) 调用 `verify(options, signature)`。
- **目的**：防止未授权方伪造内部 job 调度请求。由于 job 通过 HTTP 回调自身 API 端点执行，签名确保只有系统内部能触发任务。
- **与重试关系**：重试时同样需要签名验证（`isRetry` 不会绕过签名校验）。

### 8.7 `runTask` 幂等性

- **机制**：`io.runTask(cacheKey, callback)` 通过 `sha256(cacheKey) + "--" + jobId` 生成确定性 task ID。
- **幂等保证**：
  1. 查找 `BackgroundJobTask`（id = `task-${hash}--${jobId}`）。
  2. 如果存在且 `status === COMPLETED`：直接返回缓存的 `result`，**不重新执行 callback**。
  3. 如果不存在：创建新 task，执行 callback。
  4. 如果 `task.retried >= 3`：抛出 `BackgroundTaskExceededRetriesError`，job 标记 FAILED。
- **与 webhook handler 的关系**：当前 `execute-webhook.handler.ts` 中 `io` 参数未被使用（`_io`），因此 webhook 执行**没有** task 级幂等保护。这意味着如果 job 在 `WebhookCall.create` 之后、`return` 之前崩溃（虽然当前代码逻辑上不太可能），重试时会重复发送 webhook 请求。

### 8.8 `maxRetries`

- **默认值**：`3`（Prisma schema `BackgroundJob.maxRetries @default(3)`）。
- **检查位置**：[local.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/client/local.ts#L313-L314)
  ```ts
  const jobHasExceededRetries =
    backgroundJob.retried >= backgroundJob.maxRetries && !(error instanceof BackgroundTaskFailedError);
  ```
- **含义**：最多重试 3 次，即 job 总共执行 4 次（1 次原始 + 3 次重试）。
- **与 `runTask` 的交互**：`BackgroundTaskFailedError` 不计入 job 级重试上限——只有 task 级的 3 次上限。但如果 task 超限抛出 `BackgroundTaskExceededRetriesError`，则直接标记 job FAILED。

### 8.9 关系总图

```
triggerWebhook()
  │
  ├─ getAllWebhooksByEventTrigger() ── WHERE enabled=true AND eventTriggers has DOCUMENT_SENT
  │
  └─ jobs.triggerJob("internal.execute-webhook", { event, webhookId, data })
       │
       ├─ prisma.backgroundJob.create({ status: PENDING, maxRetries: 3 })
       │
       └─ submitJobToEndpoint()
            │  sign(data) → X-Job-Signature
            │  POST /api/jobs/{jobDefId}/{jobId}
            │
            └─ getApiHandler()
                 │  verify(signature) ✓
                 │  CAS: PENDING → PROCESSING
                 │
                 └─ handler.run({ payload, io })
                      │
                      ├─ prisma.webhook.findUniqueOrThrow()
                      │
                      └─ executeWebhookCall({ url, body, secret })
                           │
                           ├─ assertNotPrivateUrl(url)     ← DNS 解析 SSRF 检查
                           │   ├─ isBypassedHost() → skip
                           │   ├─ isPrivateUrl() → 字面量检查
                           │   └─ dns.lookup() → 解析后逐 IP 检查
                           │
                           ├─ fetchWithTimeout(url, {
                           │     method: 'POST',
                           │     timeoutMs: 10_000,
                           │     headers: { 'X-Documenso-Secret': secret ?? '' }
                           │   })
                           │
                           └─ return { success, responseCode, responseBody, responseHeaders }
                                │
                                ├─ prisma.webhookCall.create({
                                │     status: SUCCESS | FAILED,
                                │     ...
                                │   })
                                │
                                └─ if (!success) throw → 触发重试
                                      │
                                      └─ catch in getApiHandler:
                                           ├─ retried < maxRetries → PENDING, submitJobToEndpoint(isRetry:true)
                                           └─ retried >= maxRetries → FAILED
```

---

## 9. DNS 重绑定 / 私有地址解析测试场景

### 9.1 攻击场景：DNS 重绑定（DNS Rebinding）

攻击者控制一个域名 `evil.example.com`，配置 DNS 使其在不同时间返回不同 IP：

1. **创建 Webhook 时**（`ZWebhookUrlSchema` 校验）：
   - `evil.example.com` A 记录 → `8.8.8.8`（公网 IP）
   - `isPrivateUrl("https://evil.example.com/hook")` → `false`（hostname 不是 IP，也不在已知私有列表）
   - ✅ 校验通过，webhook 创建成功

2. **执行 Webhook 时**（`assertNotPrivateUrl` 校验）：
   - `evil.example.com` A 记录 → `192.168.1.1`（内网 IP，DNS 已被攻击者重新配置）
   - DNS lookup 返回 `192.168.1.1`
   - `isPrivateUrl("http://192.168.1.1")` → `true`
   - ✅ 被 `assertNotPrivateUrl` 拦截

### 9.2 唯一可利用的窗口：TOCTOU 竞态

`assertNotPrivateUrl` 中存在一个 **Time-of-Check-to-Time-of-Use (TOCTOU)** 竞态窗口：

```
assertNotPrivateUrl(url)    ← DNS lookup 返回公网 IP 8.8.8.8 ✅
         ↓  (时间窗口：几毫秒到几秒)
fetchWithTimeout(url, ...)  ← DNS 重新解析，此时返回 192.168.1.1
         ↓
POST 发送到内网地址 ❌
```

**关键差异**：`assertNotPrivateUrl` 使用 `node:dns/promises.lookup` 解析，而 `fetchWithTimeout` 底层使用操作系统的 DNS 解析器。两次解析之间存在时间差，攻击者可以在此时窗口内切换 DNS 记录。

### 9.3 DNS 解析超时/失败放行场景

[assert-webhook-url.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/webhooks/assert-webhook-url.ts#L95-L129) 中：

```ts
const lookupResult = await withTimeout(
  resolveHostname(hostname, { all: true, verbatim: true }),
  WEBHOOK_DNS_LOOKUP_TIMEOUT_MS,  // 250ms
);

if (!lookupResult) return;  // 超时 → null → 静默放行！

// ...
} catch (err) {
  if (err instanceof AppError) throw err;
  return;  // 其他异常 → 静默放行！
}
```

- **DNS 超时 250ms**：如果 DNS 查询在 250ms 内未响应，`withTimeout` 返回 `null`，函数直接 return（放行）。
- **DNS 解析异常**：非 `AppError` 类型的异常也被静默放行。

**攻击场景**：攻击者配置 DNS 服务器使其响应极慢（>250ms），导致 `assertNotPrivateUrl` 超时放行，然后 `fetch` 发起请求时 DNS 缓存或重试可能成功解析到内网地址。

### 9.4 具体测试场景

**场景名**：DNS 解析超时放行 + 后续 fetch 解析到私有地址

```
前置条件：
1. 攻击者控制域名 slow-evil.example.com
2. 该域名 DNS 配置：首次查询响应极慢（>250ms），TTL 极短（1s）
3. 后续查询返回 127.0.0.1

步骤：
1. 创建 webhook: URL = https://slow-evil.example.com/hook
   - isPrivateUrl() 检查 hostname "slow-evil.example.com" → false ✅
   - webhook 创建成功

2. 触发 DOCUMENT_SENT 事件

3. executeWebhookCall 调用 assertNotPrivateUrl(url)
   - dns.lookup("slow-evil.example.com") 超过 250ms
   - withTimeout 返回 null
   - assertNotPrivateUrl 静默放行

4. fetchWithTimeout(url, ...) 执行
   - Node.js 内部 DNS 解析（可能使用缓存或重新查询）
   - 此时 DNS 返回 127.0.0.1（攻击者已切换记录）
   - POST 请求发送到 127.0.0.1（内网）

预期结果：
- assertNotPrivateUrl 未拦截（超时放行）
- 请求成功发送到内网地址
- WebhookCall 记录 responseCode 为内网服务响应码
- SSRF 攻击成功
```

**修复建议**：
1. 使用自定义 DNS resolver 的结果直接连接（pin DNS resolution），避免 `assertNotPrivateUrl` 和 `fetch` 之间的二次解析。
2. DNS 超时不应静默放行，应默认拒绝。
3. DNS 解析异常也应默认拒绝，而非放行。
4. 在 `fetch` 层面增加 socket 级别的目标地址检查（连接建立后验证实际 IP 是否为私有地址）。
