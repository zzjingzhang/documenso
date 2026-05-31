# Documenso 嵌入式 Presign Token 完整流程分析

## 概述

本文档分析第三方如何通过 API Token 换取嵌入式 Presign Token，再在 iframe 中创建或更新 Envelope 的完整流程。涵盖计费校验、JWT 结构、权限限制、EnvelopeItem 操作语义等关键细节。

---

## 1. 端到端流程概览

```
第三方后端                          Documenso 后端                         浏览器 iframe
──────────                        ──────────────                        ──────────────

1. POST /embedding/create-presign-token
   Header: Authorization: Bearer <api_token>
   Body: { expiresIn, scope }  ──►
                                   2. 验证 api_token
                                      3. 检查计费 & embedAuthoring flag
                                      4. 生成 JWT presign token
                                  ◄── 5. { token, expiresAt, expiresIn }

6. 将 presign token 传入前端

                                   7. iframe 加载 /embed/v2/authoring/...
                                      ?token=<presign_token>
                                      #<base64(embed_data)>
                                                                      8. loader 调用 verifyEmbeddingPresignToken
                                                                      9. 注入 TrpcProvider (Bearer presignToken)
                                                                      10. 用户在编辑器中操作

                                                                      11. 创建: createEmbeddingEnvelope (mutation)
                                                                          更新: updateEmbeddingEnvelope (mutation)
                                                                          Header: Authorization: Bearer <presign_token>
```

---

## 2. API Token 换取 Presign Token

### 2.1 路由入口

路由定义在 [create-embedding-presign-token.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/embedding-router/create-embedding-presign-token.ts)，OpenAPI 路径为 `POST /embedding/create-presign-token`。

请求从 `Authorization` 头中提取 Bearer Token：

```ts
const authorizationHeader = req.headers.get('authorization');
const [apiToken] = (authorizationHeader || '').split('Bearer ').filter((s) => s.length > 0);
```

请求 Schema 定义在 [create-embedding-presign-token.types.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/embedding-router/create-embedding-presign-token.types.ts)：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `expiresIn` | `number` (0–10080) | 60 | 过期时间，单位为**分钟**，最大 10080 分钟（7 天） |
| `scope` | `string` | 可选 | 资源限定，V2 格式为 `envelopeId:envelope_xxx` |

### 2.2 Billing 开启时的 embedAuthoring Flag 校验

当 `IS_BILLING_ENABLED()` 为 `true` 时（即环境变量 `NEXT_PUBLIC_FEATURE_BILLING_ENABLED=true`），路由会执行额外的计费与权限校验：

```ts
if (IS_BILLING_ENABLED()) {
  const token = await getApiTokenByToken({ token: apiToken });

  if (!token.userId) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, { message: 'Invalid API token' });
  }

  const organisationClaim = await getOrganisationClaimByTeamId({ teamId: token.teamId });

  if (!organisationClaim.flags.embedAuthoring) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, {
      message: 'Embedded Authoring is not included in your current plan. Please contact support.',
    });
  }
}
```

关键点：

1. **仅在 Billing 启用时校验**：如果 `IS_BILLING_ENABLED()` 返回 `false`，则完全跳过 flag 检查，任何 API Token 都可以创建 presign token。
2. **先验证 `token.userId` 必须存在**：若 API Token 没有关联用户，直接拒绝。
3. **通过 `getOrganisationClaimByTeamId` 获取组织声明**：检查 `flags.embedAuthoring` 是否为 `true`，若非则报错 "not included in your current plan"。

参见 [IS_BILLING_ENABLED 定义](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/constants/app.ts#L16)。

### 2.3 getApiTokenByToken 的 bypassRateLimit 差异

[getApiTokenByToken](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/public-api/get-api-token-by-token.ts) 函数接受 `bypassRateLimit` 参数（默认 `false`）。

在 presign token 创建流程中，**同一个 API Token 被调用了两次** `getApiTokenByToken`，且 `bypassRateLimit` 参数不同：

| 调用位置 | `bypassRateLimit` | 原因 |
|----------|-------------------|------|
| 路由层（billing 校验时） | `false`（默认） | 此时需要执行速率限制检查，防止暴力获取 presign token |
| [createEmbeddingPresignToken](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/embedding-presign/create-embedding-presign-token.ts#L25) 内部 | `true` | 已在路由层做过速率限制，避免重复计数 |

当 `bypassRateLimit = false` 时，函数会调用 `assertOrganisationRatesAndLimits` 检查组织的 API 调用频率限制：

```ts
if (!bypassRateLimit) {
  await assertOrganisationRatesAndLimits({
    organisationId: apiToken.team.organisationId,
    organisationClaim: apiToken.team.organisation.organisationClaim,
    type: 'api',
    count: 1,
  });
}
```

> **注意**：若 `IS_BILLING_ENABLED()` 为 `false`，路由层不会调用 `getApiTokenByToken`，速率限制仅在 `createEmbeddingPresignToken` 内部以 `bypassRateLimit: true` 调用时跳过。这意味着在非计费模式下，presign token 创建**完全不做速率限制检查**。

---

## 3. JWT Presign Token 结构

### 3.1 签发

在 [createEmbeddingPresignToken](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/embedding-presign/create-embedding-presign-token.ts) 中使用 `jose` 库签发 JWT：

```ts
const secret = new TextEncoder().encode(validatedToken.token);

const token = await new SignJWT({
  aud: String(validatedToken.teamId ?? validatedToken.userId),
  sub: String(validatedToken.id),
  scope,
})
  .setProtectedHeader({ alg: 'HS256' })
  .setIssuedAt(now.toJSDate())
  .setExpirationTime(expiresAt.toJSDate())
  .sign(secret);
```

### 3.2 JWT Claims 详解

| Claim | 值 | 说明 |
|-------|-----|------|
| `aud` (Audience) | `teamId ?? userId` 的字符串形式 | 优先使用 `teamId`，若 API Token 不属于任何团队则回退到 `userId`。验证时必须匹配 `apiToken.teamId` 或 `apiToken.userId` |
| `sub` (Subject) | API Token 的数据库 ID（`validatedToken.id`） | 用于验证时查找 API Token 记录 |
| `scope` | 可选，如 `envelopeId:envelope_xxx` | 限制 presign token 只能操作指定资源 |
| `iat` (Issued At) | 签发时间 | 由 `setIssuedAt` 自动设置 |
| `exp` (Expiration) | 过期时间 | 由 `setExpirationTime` 自动设置 |

### 3.3 HS256 签名密钥

**签名密钥 = API Token 的原始明文值（`validatedToken.token`）**。

这是一个关键设计：API Token 在数据库中存储的是哈希值，但 JWT 签名使用的是原始明文。验证时需要先通过 `sub`（API Token ID）从数据库查出记录，再用其 `token` 字段（此处实际存储的是哈希值）验证签名。

> **修正**：查看代码发现，数据库中的 `apiToken.token` 存储的是哈希值，而签发时 `validatedToken.token` 来自 `getApiTokenByToken` 返回的对象——但 Prisma 查询返回的是数据库中存储的值。实际上签名密钥使用的是 **数据库中存储的 token 值**（哈希后的），而验证时也使用同一哈希值，因此签名可以正常验证。

### 3.4 生产环境最短过期时间

```ts
const isDevelopment = env('NODE_ENV') !== 'production';
const minExpirationMinutes = isDevelopment ? 0 : 5;

const effectiveExpiresIn = expiresIn !== undefined && expiresIn >= minExpirationMinutes
  ? expiresIn
  : 60;
```

| 环境 | 最低过期时间 | 默认过期时间 |
|------|-------------|-------------|
| 开发（`NODE_ENV !== 'production'`） | 0 分钟（可用于测试立即过期） | 60 分钟 |
| 生产（`NODE_ENV === 'production'`） | **5 分钟** | 60 分钟 |

- 若传入 `expiresIn` 小于最低值，则回退到默认 60 分钟
- 最大值为 10080 分钟（7 天），由 Zod Schema 限制

---

## 4. Presign Token 验证

### 4.1 验证流程

[verifyEmbeddingPresignToken](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/embedding-presign/verify-embedding-presign-token.ts) 的完整验证步骤：

1. **解码 JWT**（不做签名验证）：使用 `decodeJwt` 提取 claims
2. **校验必需 claims**：`sub` 和 `aud` 必须存在且为字符串
3. **类型转换与校验**：将 `sub` 转为 `tokenId`（number），`aud` 转为 `audienceId`（number），校验是否为合法整数
4. **查找 API Token**：通过 `tokenId` 从数据库查找 `apiToken` 记录，并 include `user`
5. **校验 API Token 必须有 userId**：`apiToken.userId` 和 `apiToken.user` 必须存在
6. **校验 audience 匹配**：`audienceId` 必须等于 `apiToken.teamId` 或 `apiToken.userId`
7. **校验 scope 匹配**（如果验证时传入了 `scope` 参数）
8. **JWT 签名验证**：使用 `apiToken.token`（哈希值）作为密钥，调用 `jwtVerify`

### 4.2 API Token 的 userId 必需性

验证代码中明确要求：

```ts
if (!apiToken.userId || !apiToken.user) {
  throw new AppError(AppErrorCode.UNAUTHORIZED, {
    message: 'Invalid presign token: API token does not have a user attached',
  });
}
```

这意味着：
- 如果 API Token 记录中的 `userId` 为 `null`，验证失败
- 如果 API Token 关联的 `user` 不存在，验证失败
- 返回的 `userId` 来自 `apiToken.userId`，后续所有操作都依赖此值进行权限检查

> 这与路由层 billing 检查中的 `!token.userId` 校验一致，但验证层的检查是**无条件的**——即使 billing 未开启，presign token 验证也要求 userId 存在。

### 4.3 scope=envelopeId:... 如何限制更新操作

在 [updateEmbeddingEnvelope](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/embedding-router/update-embedding-envelope.ts) 中，验证时显式传入了 scope：

```ts
const apiToken = await verifyEmbeddingPresignToken({
  token: presignToken,
  scope: `envelopeId:${envelopeId}`,
});
```

验证逻辑中：

```ts
if (decodedToken.scope && scope && decodedToken.scope !== scope) {
  throw new AppError(AppErrorCode.UNAUTHORIZED, {
    message: 'Presign token scope not matched',
  });
}
```

**scope 校验规则**：

| JWT 中的 scope | 验证时传入的 scope | 结果 |
|----------------|-------------------|------|
| `envelopeId:envelope_abc` | `envelopeId:envelope_abc` | ✅ 匹配 |
| `envelopeId:envelope_abc` | `envelopeId:envelope_xyz` | ❌ 拒绝 |
| `undefined` | `envelopeId:envelope_abc` | ✅ 通过（JWT 无 scope 限制则不校验） |
| `envelopeId:envelope_abc` | `undefined` | ✅ 通过（验证方未要求 scope 则不校验） |

**设计含义**：

- 创建 presign token 时，若指定了 `scope: "envelopeId:envelope_abc"`，则此 token **只能**用于更新 `envelope_abc`
- 若创建时未指定 scope，则 token 可用于操作任何 envelope（但仍然受 userId/teamId 权限限制）
- 这是一种**最小权限**机制：为更新操作签发带 scope 的 presign token，防止 token 被用于操作非目标 envelope

**对比 createEmbeddingEnvelope**：创建时不传 scope（因为 envelope 尚未创建，没有 envelopeId）：

```ts
const apiToken = await verifyEmbeddingPresignToken({ token: presignToken });
```

---

## 5. iframe 中的 Authoring Layout

### 5.1 Loader 验证

[_layout.tsx](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/app/routes/embed+/v2+/authoring+/_layout.tsx) 的 loader 从 URL 参数中提取 token 并验证：

```ts
export const loader = async ({ request }: Route.LoaderArgs) => {
  const url = new URL(request.url);
  const token = url.searchParams.get('token');

  if (!token) {
    throw new Response('Invalid token', { status: 404 });
  }

  const result = await verifyEmbeddingPresignToken({ token }).catch(() => null);

  if (!result) {
    throw new Response('Invalid token', { status: 404 });
  }

  const organisationClaim = await getOrganisationClaimByTeamId({ teamId: result.teamId });
  const teamSettings = await getTeamSettings({ userId: result.userId, teamId: result.teamId });

  return {
    token,
    userId: result.userId,
    teamId: result.teamId,
    organisationClaim,
    preferences: { aiFeaturesEnabled: teamSettings.aiFeaturesEnabled },
  };
};
```

### 5.2 Provider 注入与 TrpcProvider

Layout 将 presign token 注入到 `TrpcProvider` 的 headers 中：

```tsx
<TrpcProvider headers={{ authorization: `Bearer ${token}`, 'x-team-Id': team.id.toString() }}>
```

这意味着所有子路由的 tRPC 请求都会自动携带 presign token 作为 Bearer Token，无需前端额外处理认证。

### 5.3 Hash 参数解析（白标 / 主题 / 语言）

前端通过 URL hash 传递嵌入配置（base64 编码的 JSON）：

```ts
const hash = window.location.hash.slice(1);
const result = ZBaseEmbedDataSchema.safeParse(JSON.parse(decodeURIComponent(atob(hash))));
const { css, cssVars, darkModeDisabled, language } = result.data;
```

- `css` / `cssVars`：自定义样式，仅在 `embedAuthoringWhiteLabel` flag 启用时注入
- `darkModeDisabled`：禁用暗色模式
- `language`：国际化语言切换

---

## 6. createEmbeddingEnvelope：创建 Envelope

### 6.1 复用 createEnvelopeRouteCaller

[createEmbeddingEnvelope](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/embedding-router/create-embedding-envelope.ts) 的核心逻辑是复用 [createEnvelopeRouteCaller](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/envelope-router/create-envelope.ts#L65)：

```ts
const apiToken = await verifyEmbeddingPresignToken({ token: presignToken });
const { userId, teamId } = apiToken;

return await createEnvelopeRouteCaller({
  userId,
  teamId,
  input,
  options: {
    bypassDefaultRecipients: true,
  },
  apiRequestMetadata: ctx.metadata,
  logger: ctx.logger,
});
```

### 6.2 bypassDefaultRecipients 的含义

`bypassDefaultRecipients: true` 传递到 `createEnvelope` 函数，其效果是：

- **不自动注入模板的默认收件人**：普通创建流程中，如果模板有预设收件人，会自动添加到新 envelope
- **嵌入场景由前端控制收件人**：嵌入编辑器会在前端 UI 中让用户手动添加收件人，因此后端不需要自动添加默认收件人
- 这避免了默认收件人与用户自定义收件人冲突的问题

参见 [create-envelope.ts#L198](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/envelope-router/create-envelope.ts#L198) 中 `bypassDefaultRecipients` 的传递。

---

## 7. updateEmbeddingEnvelope：更新 Envelope

### 7.1 整体流程

[updateEmbeddingEnvelope](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/embedding-router/update-embedding-envelope.ts) 按 5 个步骤执行：

1. **Step 1**：更新 EnvelopeItem（新增/替换/删除/重排/重命名）
2. **Step 2**：更新 Envelope 通用数据和 meta
3. **Step 3**：更新收件人
4. **Step 4**：更新字段
5. **Step 5**：处理附件（set 语义）

### 7.2 EnvelopeItem 操作的区分逻辑

根据 [类型定义](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/embedding-router/update-embedding-envelope.types.ts#L34-L66)，每个 envelopeItem 包含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `string` | 可能是真实 ID 或 `PRESIGNED_` 前缀的临时 ID |
| `title` | `string` | 标题 |
| `order` | `number` | 排序序号 |
| `index` | `number?` | 新增项的文件索引（在 `files` 数组中的位置） |
| `replaceFileIndex` | `number?` | 替换项的文件索引（仅适用于已有真实 ID 的项） |

Zod refine 确保 `index` 和 `replaceFileIndex` 不能同时出现。

代码中按以下逻辑分类：

#### 7.2.1 判断是新增还是已有项

```ts
const isNewEnvelopeItem = item.id.startsWith(PRESIGNED_ENVELOPE_ITEM_ID_PREFIX);
```

[PRESIGNED_ENVELOPE_ITEM_ID_PREFIX](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/utils/embed-config.ts#L5) 定义为 `"PRESIGNED_"`。

- ID 以 `PRESIGNED_` 开头 → 新增项（尚未上传到服务器，客户端分配的临时 ID）
- ID 不以 `PRESIGNED_` 开头 → 已有项（数据库中的真实 ID）

#### 7.2.2 新增（Create）

当 `isNewEnvelopeItem === true` 时：

```ts
const newEnvelopeItemFile = item.index !== undefined ? files[item.index] : undefined;

envelopeItemsToCreate.push({
  embeddedEnvelopeItemId: item.id,       // 保留 PRESIGNED_ 前缀的临时 ID
  title: item.title,
  order: item.order,
  file: newEnvelopeItemFile,
});
```

- `item.index` 指向 `files` 数组中的文件
- 如果 `index` 缺失或对应文件不存在，抛出 `INVALID_BODY` 错误
- 创建后，临时 ID 映射到真实 ID 存入 `embeddedEnvelopeItemIdMapping`

#### 7.2.3 替换 PDF（Replace）

当 `isNewEnvelopeItem === false` 且 `item.replaceFileIndex !== undefined` 时：

```ts
envelopeItemsToReplace.push({
  envelopeItemId: envelopeItem.id,           // 真实 ID
  oldDocumentDataId: envelopeItem.documentDataId,
  title: item.title,
  order: item.order,
  file: replaceFile,
});
```

- 替换已有项的 PDF 文件，但不创建新的占位符字段
- 字段清理在后续步骤中处理
- 调用 `UNSAFE_replaceEnvelopeItemPdf`

#### 7.2.4 更新元数据（Update）

当 `isNewEnvelopeItem === false` 且 `replaceFileIndex === undefined` 且 title/order 有变化时：

```ts
const hasEnvelopeItemChanged = envelopeItem.title !== item.title || envelopeItem.order !== item.order;

if (hasEnvelopeItemChanged) {
  envelopeItemsToUpdate.push({
    envelopeItemId: envelopeItem.id,
    title: item.title,
    order: item.order,
  });
}
```

- 仅更新 title 和/或 order
- 调用 `UNSAFE_updateEnvelopeItems`

#### 7.2.5 删除（Delete）

**差集逻辑**：数据库中存在但请求 payload 中不存在的 envelopeItem 被视为删除：

```ts
const envelopeItemIdsToDelete = envelope.envelopeItems
  .filter((item) => !data.envelopeItems.some((i) => i.id === item.id))
  .map((item) => item.id);
```

- 调用 `UNSAFE_deleteEnvelopeItem`
- 并发度为 2 的 `pMap` 执行删除

### 7.3 PRESIGNED_ENVELOPE_ITEM_ID_PREFIX 映射机制

映射表 `embeddedEnvelopeItemIdMapping` 的生命周期：

1. **前端生成**：嵌入编辑器为新上传的文件分配 `PRESIGNED_xxx` 格式的临时 ID
2. **后端创建**：`UNSAFE_createEnvelopeItems` 接受 `clientId` 参数（即 `PRESIGNED_xxx`），创建后返回真实 ID
3. **映射构建**：

```ts
createdEnvelopeItems.forEach((item) => {
  if (!item.clientId) {
    throw new AppError(AppErrorCode.NOT_FOUND, { message: 'Client ID not found' });
  }
  embeddedEnvelopeItemIdMapping[item.clientId] = item.id;
});
```

4. **后续引用**：在字段更新阶段，字段的 `envelopeItemId` 如果以 `PRESIGNED_` 开头，会被替换为真实 ID：

```ts
let envelopeItemId = field.envelopeItemId;
if (envelopeItemId.startsWith(PRESIGNED_ENVELOPE_ITEM_ID_PREFIX)) {
  envelopeItemId = embeddedEnvelopeItemIdMapping[envelopeItemId];
}
```

这确保了新建项关联的字段能正确引用到数据库中的真实 envelopeItem ID。

### 7.4 getEnvelopeItemPermissions 限制

[getEnvelopeItemPermissions](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/utils/envelope.ts#L241) 在 envelopeItem 修改前进行权限检查：

```ts
if (willEnvelopeItemsBeModified) {
  const permissions = getEnvelopeItemPermissions(envelope, envelope.recipients);
  // ...
}
```

权限矩阵：

| Envelope 状态 | canTitleBeChanged | canFileBeChanged | canOrderBeChanged |
|---------------|-------------------|------------------|-------------------|
| 已完成/已拒绝/已删除 | ❌ | ❌ | ❌ |
| 模板（TEMPLATE） | ✅ | ✅ | ✅ |
| 文档草稿（DRAFT） | ✅ | ✅ | ✅ |
| 文档待签（PENDING） | ✅ | ❌ | 仅当无活跃收件人时 ✅ |

其中"活跃收件人"定义为：

```ts
const hasActiveRecipients = recipients.some(
  (recipient) =>
    recipient.role !== RecipientRole.CC &&
    (recipient.signingStatus === SigningStatus.SIGNED ||
     recipient.signingStatus === SigningStatus.REJECTED ||
     recipient.sendStatus === SendStatus.SENT),
);
```

在 `updateEmbeddingEnvelope` 中的具体检查：

| 操作类型 | 检查的权限 | 含义 |
|----------|-----------|------|
| 删除/新增项 | `canFileBeChanged` | 文件是否可变更 |
| order 变更 | `canOrderBeChanged` | 排序是否可变更 |
| title 变更 | `canTitleBeChanged` | 标题是否可变更 |

此外还有 **envelopeItem 数量限制**：

```ts
if (resultingEnvelopeItemCount > organisationClaim.envelopeItemCount) {
  throw new AppError('ENVELOPE_ITEM_LIMIT_EXCEEDED', { ... });
}
```

### 7.5 收件人更新：setDocumentRecipients vs setTemplateRecipients

根据 `envelope.type` 分派不同的收件人设置函数：

```ts
const { recipients: updatedRecipients } = await match(envelope.type)
  .with(EnvelopeType.DOCUMENT, async () =>
    setDocumentRecipients({ userId, teamId, id, recipients, requestMetadata }),
  )
  .with(EnvelopeType.TEMPLATE, async () =>
    setTemplateRecipients({ userId, teamId, id, recipients }),
  )
  .exhaustive();
```

#### [setDocumentRecipients](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/recipient/set-document-recipients.ts)

- **set 语义**：传入的 recipients 列表即为最终状态
- **新增**：不在已有列表中的（`id` 为空或不存在）→ 创建新 recipient
- **更新**：在已有列表中且数据有变化 → 更新（但已交互的 recipient 不可修改）
- **删除**：已有但不在新列表中的 → 删除，并向已发送的 recipient 发送移除邮件
- **actionAuth**：若设置了 `actionAuth`，需要 `cfr21` flag
- **角色变更**：若角色变为 CC/VIEWER，自动清除该 recipient 的所有 fields
- **审计日志**：记录创建/更新/删除操作

#### [setTemplateRecipients](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/recipient/set-template-recipients.ts)

- 同样是 **set 语义**
- **额外限制**：若模板启用了直接链接（`directLink`），则：
  - 不能删除直接链接对应的 recipient
  - 不能将直接链接 recipient 的角色设为 CC
  - 直接链接 recipient 的 email/name 会被强制替换为常量值
- **无审计日志**：模板操作不记录审计日志
- **无邮件通知**：模板操作不发送邮件

两者都使用 `upsert` 模式，通过 `clientId`（由 `nanoid()` 生成）关联前端和后端的 recipient 记录。

### 7.6 字段更新：setFieldsForDocument vs setFieldsForTemplate

同样根据 `envelope.type` 分派：

```ts
await match(envelope.type)
  .with(EnvelopeType.DOCUMENT, async () =>
    setFieldsForDocument({ userId, teamId, id, fields, requestMetadata }),
  )
  .with(EnvelopeType.TEMPLATE, async () =>
    setFieldsForTemplate({ userId, teamId, id, fields }),
  )
  .exhaustive();
```

#### [setFieldsForDocument](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/field/set-fields-for-document.ts)

- **set 语义**：传入的 fields 列表为最终状态
- **新增**：`id` 为空 → 创建 field，并验证 fieldMeta
- **更新**：`id` 已存在 → 更新位置和元数据（但已交互 recipient 的 fields 不可修改）
- **删除**：已有但不在新列表中的 → 删除
- **recipient 关联校验**：每个 field 必须关联到一个有效的 recipient
- **envelopeItem 关联校验**：每个 field 必须关联到一个有效的 envelopeItem
- **字段类型验证**：TEXT、NUMBER、CHECKBOX、RADIO、DROPDOWN 各有独立的验证逻辑
- **审计日志**：记录创建/更新/删除操作

#### [setFieldsForTemplate](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/field/set-fields-for-template.ts)

- 同样是 **set 语义**
- **无交互限制**：模板的 recipient 不会"已交互"，因此不检查 `canRecipientFieldsBeModified`
- **无审计日志**
- 其他验证逻辑（recipient/envelopeItem 关联、字段类型验证）与 document 相同

在嵌入更新场景中，字段更新前会进行 **PRESIGNED_ ID 映射**：

```ts
let envelopeItemId = field.envelopeItemId;
if (envelopeItemId.startsWith(PRESIGNED_ENVELOPE_ITEM_ID_PREFIX)) {
  envelopeItemId = embeddedEnvelopeItemIdMapping[envelopeItemId];
}
```

同时，收件人 ID 通过 `clientId` 关联：

```ts
const recipientId = updatedRecipients.find((r) => r.clientId === recipient.clientId)?.id;
```

### 7.7 附件的 set 语义

附件采用 **完全替换**（set）语义：

```ts
if (hasEnvelopeAttachmentsChanged) {
  await prisma.envelopeAttachment.deleteMany({
    where: { envelopeId: envelope.id },
  });

  if (data.attachments.length > 0) {
    await prisma.envelopeAttachment.createMany({
      data: data.attachments.map((attachment) => ({
        envelopeId: envelope.id,
        label: attachment.label,
        data: attachment.data,
        type: attachment.type,
      })),
    });
  }
}
```

**变更检测**：

1. 数量不同 → 视为已变更
2. 按 `id` 查找，若找不到 → 视为已变更
3. 若找到了，比较 `label`、`data`、`type` 三个字段 → 有差异则视为已变更

**操作流程**：

1. **先删后建**：先删除该 envelope 的所有附件，再批量创建新的
2. **非增量**：不是 diff-patch，而是全量替换

类型定义中附件的 Schema：

```ts
attachments: EnvelopeAttachmentSchema.pick({
  type: true,
  label: true,
  data: true,
}).extend({
  id: z.string().optional(),
}).array(),
```

注释明确标注：

> This is a set command: when provided, all existing attachments are deleted and replaced with the provided list.

---

## 8. 安全设计总结

| 层级 | 机制 | 说明 |
|------|------|------|
| API Token 认证 | Bearer Token + 数据库查询 + 哈希比对 | 确保 token 有效且未过期 |
| 速率限制 | `assertOrganisationRatesAndLimits` | 防止 API 滥用（bypassable） |
| 计费校验 | `IS_BILLING_ENABLED` + `embedAuthoring` flag | 仅在计费开启时检查 |
| JWT 签名 | HS256 + API Token 哈希作为密钥 | 防止 token 伪造 |
| JWT 过期 | `exp` claim + 生产环境最少 5 分钟 | 防止 token 永久有效 |
| Scope 限制 | `envelopeId:xxx` claim | 限制 token 只能操作指定 envelope |
| Audience 校验 | `aud` = teamId/userId | 确保 token 属于正确的上下文 |
| userId 必需 | verifyEmbeddingPresignToken 强制检查 | 确保操作可追溯 |
| 权限矩阵 | `getEnvelopeItemPermissions` | 基于状态和收件人交互情况限制操作 |
| 数量限制 | `organisationClaim.envelopeItemCount` | 防止超出套餐限制 |
| 完成状态保护 | `DocumentStatus.COMPLETED` 拒绝修改 | 防止篡改已完成文档 |

---

## 9. 关键文件索引

| 文件 | 职责 |
|------|------|
| [create-embedding-presign-token.ts (route)](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/embedding-router/create-embedding-presign-token.ts) | Presign Token 创建路由 |
| [create-embedding-presign-token.types.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/embedding-router/create-embedding-presign-token.types.ts) | 请求/响应 Schema |
| [create-embedding-presign-token.ts (lib)](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/embedding-presign/create-embedding-presign-token.ts) | JWT 签发逻辑 |
| [verify-embedding-presign-token.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/embedding-presign/verify-embedding-presign-token.ts) | JWT 验证逻辑 |
| [create-embedding-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/embedding-router/create-embedding-envelope.ts) | 嵌入式创建 Envelope 路由 |
| [update-embedding-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/embedding-router/update-embedding-envelope.ts) | 嵌入式更新 Envelope 路由 |
| [update-embedding-envelope.types.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/embedding-router/update-embedding-envelope.types.ts) | 更新请求 Schema |
| [_layout.tsx](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/app/routes/embed+/v2+/authoring+/_layout.tsx) | 嵌入式编辑器 Layout |
| [create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/envelope-router/create-envelope.ts) | createEnvelopeRouteCaller 定义 |
| [get-api-token-by-token.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/public-api/get-api-token-by-token.ts) | API Token 验证与速率限制 |
| [envelope.ts (utils)](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/utils/envelope.ts) | getEnvelopeItemPermissions 等工具函数 |
| [embed-config.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/utils/embed-config.ts) | PRESIGNED_ENVELOPE_ITEM_ID_PREFIX 定义 |
| [set-document-recipients.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/recipient/set-document-recipients.ts) | 文档收件人 set 语义 |
| [set-template-recipients.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/recipient/set-template-recipients.ts) | 模板收件人 set 语义 |
| [set-fields-for-document.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/field/set-fields-for-document.ts) | 文档字段 set 语义 |
| [set-fields-for-template.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/field/set-fields-for-template.ts) | 模板字段 set 语义 |
| [app.ts (constants)](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/constants/app.ts) | IS_BILLING_ENABLED 定义 |
