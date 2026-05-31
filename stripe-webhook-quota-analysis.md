# Stripe Webhook 与配额限流系统深度分析

## 1. Webhook 入口与签名校验

### 1.1 入口路由

[stripe.webhook.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/app/routes/api+/stripe.webhook.ts) 作为 Remix API 路由，直接将请求转发给 `stripeWebhookHandler` 处理。

### 1.2 签名校验流程

[handler.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/handler.ts) 实现了完整的安全校验流程：

| 校验阶段 | 检查项 | 代码位置 | 失败处理 |
|---------|--------|---------|---------|
| **前置检查** | Billing 是否启用 | [L18, L26-L34](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/handler.ts#L18-L34) | 返回 500，`Billing is disabled` |
| **前置检查** | Webhook Secret 是否存在 | [L20-L24](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/handler.ts#L20-L24) | 抛出 Error |
| **签名检查** | `stripe-signature` header | [L36-L47](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/handler.ts#L36-L47) | 返回 400，`No signature found` |
| **Payload 检查** | 请求 body 非空 | [L49-L59](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/handler.ts#L49-L59) | 返回 400，`No payload found` |
| **签名验证** | Stripe SDK 构造事件 | [L61](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/handler.ts#L61-L61) | 抛出异常，被 catch 捕获 |

**核心校验代码**：
```typescript
const event = stripe.webhooks.constructEvent(payload, signature, webhookSecret);
```

## 2. 事件分发与状态映射

### 2.1 事件类型分发

使用 `ts-pattern` 的 match 进行事件路由：

| 事件类型 | 处理函数 |
|---------|---------|
| `customer.subscription.created` | `onSubscriptionCreated` |
| `customer.subscription.updated` | `onSubscriptionUpdated` |
| `customer.subscription.deleted` | `onSubscriptionDeleted` |
| 其他事件 | 静默忽略，返回 200 |

### 2.2 SubscriptionStatus 映射

两个处理文件都使用相同的映射逻辑：

[on-subscription-created.ts L74-L78](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/on-subscription-created.ts#L74-L78)
[on-subscription-updated.ts L82-L86](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/on-subscription-updated.ts#L82-L86)

```typescript
const status = match(subscription.status)
  .with('active', () => SubscriptionStatus.ACTIVE)
  .with('trialing', () => SubscriptionStatus.ACTIVE)
  .with('past_due', () => SubscriptionStatus.PAST_DUE)
  .otherwise(() => SubscriptionStatus.INACTIVE);
```

**映射规则**：
- `active` / `trialing` → `SubscriptionStatus.ACTIVE`
- `past_due` → `SubscriptionStatus.PAST_DUE`
- 其他状态 → `SubscriptionStatus.INACTIVE`

### 2.3 periodEnd 计算

[on-subscription-created.ts L80-L83](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/on-subscription-created.ts#L80-L83)

```typescript
const periodEnd = subscription.status === 'trialing' && subscription.trial_end
  ? new Date(subscription.trial_end * 1000)
  : new Date(subscription.current_period_end * 1000);
```

## 3. ClaimId 提取机制

### 3.1 提取优先级

[on-subscription-updated.ts L148-L158](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/on-subscription-updated.ts#L148-L158)

```typescript
export const extractStripeClaimId = async (priceId: Stripe.Price) => {
  if (priceId.metadata.claimId) {
    return priceId.metadata.claimId;
  }
  const product = await stripe.products.retrieve(productId);
  return product.metadata.claimId || null;
};
```

**优先级顺序**：
1. **Price metadata.claimId** - 优先检查价格层级的元数据
2. **Product metadata.claimId** - 回退到产品层级的元数据

### 3.2 完整 Claim 提取

[on-subscription-updated.ts L165-L182](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/on-subscription-updated.ts#L165-L182)

```typescript
export const extractStripeClaim = async (priceId: Stripe.Price) => {
  const claimId = await extractStripeClaimId(priceId);
  if (!claimId) return null;
  
  const subscriptionClaim = await prisma.subscriptionClaim.findFirst({
    where: { id: claimId },
  });
  return subscriptionClaim || null;
};
```

## 4. Organisation / Subscription / OrganisationClaim 创建与更新

### 4.1 onSubscriptionCreated 流程

[on-subscription-created.ts L28-L107](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/on-subscription-created.ts#L28-L107)

```
subscription.created
    │
    ├─► 检查 subscription.items.data.length === 1
    │    └─► 多 items → 500 错误
    │
    ├─► extractStripeClaim()
    │    └─► claim 不存在 → 500 错误
    │
    └─► 判断 metadata.organisationCreateData
         │
         ├─► 存在 → handleOrganisationCreate() 新建组织
         │
         └─► 不存在 → handleOrganisationUpdate() 更新现有组织
              │
              └─► 组织不存在 / 已有活跃订阅 → 500 错误
```

### 4.2 handleOrganisationCreate - 新建组织

[on-subscription-created.ts L118-L146](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/on-subscription-created.ts#L118-L146)

1. 解析 `metadata.organisationCreateData` JSON
2. Zod schema 校验 `ZStripeOrganisationCreateMetadataSchema`
3. 调用 `createOrganisation()` 创建：
   - `type: OrganisationType.ORGANISATION`
   - 传入 `customerId` 和 `claim`
4. 返回新创建的 `organisationId`

### 4.3 handleOrganisationUpdate - 更新现有组织

[on-subscription-created.ts L156-L213](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/on-subscription-created.ts#L156-L213)

```typescript
// 个人组织升级逻辑
let newOrganisationType: OrganisationType = OrganisationType.ORGANISATION;

// 保留 PERSONAL 类型当且仅当：
// 1. 当前是 PERSONAL 类型
// 2. 订阅的是 INDIVIDUAL claim
if (organisation.type === OrganisationType.PERSONAL && 
    claim.id === INTERNAL_CLAIM_ID.INDIVIDUAL) {
  newOrganisationType = OrganisationType.PERSONAL;
}
```

**注意**：已有活跃订阅的组织会抛出错误 `Organisation already has an active subscription`

### 4.4 Subscription upsert

[on-subscription-created.ts L85-L106](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/on-subscription-created.ts#L85-L106)

```typescript
await prisma.subscription.upsert({
  where: { organisationId },
  create: { /* 完整字段 */ },
  update: { /* 更新字段 */ },
});
```

**唯一性约束**：以 `organisationId` 为唯一键，每个组织只有一条 subscription 记录。

### 4.5 onSubscriptionUpdated 流程

[on-subscription-updated.ts L18-L136](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/on-subscription-updated.ts#L18-L136)

```
subscription.updated
    │
    ├─► 检查 items 数量 = 1
    ├─► 按 customerId 查找 organisation
    │    └─► 不存在 → 500 错误
    ├─► 检测可能的双重订阅警告
    ├─► 对比 previous vs updated claimId
    ├─► PERSONAL 组织升级判断（非 INDIVIDUAL/FREE → ORGANISATION）
    └─► 事务更新
         ├─► 更新 subscription
         └─► newClaimFound → 更新 organisationClaim
```

**关键升级条件**：
```typescript
if (
  updatedSubscriptionClaim.id !== INTERNAL_CLAIM_ID.INDIVIDUAL &&
  updatedSubscriptionClaim.id !== INTERNAL_CLAIM_ID.FREE &&
  organisation.type === OrganisationType.PERSONAL
) {
  // 升级为 ORGANISATION 类型
}
```

## 5. 对 getServerLimits 的影响

[server.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/limits/server.ts)

### 5.1 配额判断优先级

```
getServerLimits()
    │
    ├─► billing 未启用 → SELFHOSTED_PLAN_LIMITS
    │
    ├─► ENTERPRISE claim → PAID_PLAN_LIMITS (无限)
    │
    ├─► subscription 状态 INACTIVE → INACTIVE_PLAN_LIMITS
    │
    ├─► unlimitedDocuments flag → PAID_PLAN_LIMITS (无限)
    │
    └─► 正常计算
         ├─► 统计本月 DOCUMENT 类型 envelopes
         ├─► 统计 directTemplates 数量
         └─► 计算 remaining = quota - used
```

### 5.2 分支详解

| 分支条件 | 结果 | 代码位置 |
|---------|------|---------|
| `!IS_BILLING_ENABLED()` | 返回 `SELFHOSTED_PLAN_LIMITS` | [L46-L52](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/limits/server.ts#L46-L52) |
| `ENTERPRISE` claimId | 返回 `PAID_PLAN_LIMITS`（无限） | [L55-L61](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/limits/server.ts#L55-L61) |
| `INACTIVE` subscription | 返回 `INACTIVE_PLAN_LIMITS` | [L64-L70](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/limits/server.ts#L64-L70) |
| `unlimitedDocuments` flag | 返回 `PAID_PLAN_LIMITS`（无限） | [L74-L80](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/limits/server.ts#L74-L80) |

## 6. 对 createEnvelopeRouteCaller 的影响

[create-envelope.ts L90-L107](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/envelope-router/create-envelope.ts#L90-L107)

```typescript
const { remaining, maximumEnvelopeItemCount } = await getServerLimits({
  userId,
  teamId,
});

if (remaining.documents <= 0) {
  throw new AppError(AppErrorCode.LIMIT_EXCEEDED, {
    message: 'You have reached your document limit for this month.',
  });
}

if (files.length > maximumEnvelopeItemCount) {
  throw new AppError('ENVELOPE_ITEM_LIMIT_EXCEEDED', { ... });
}
```

**检查点**：
1. **文档配额** - `remaining.documents <= 0` 阻止创建
2. **单信封文件数限制** - `maximumEnvelopeItemCount` 限制上传数量

## 7. 对 createTemplateDirectLink 的影响

[create-template-direct-link.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/template/create-template-direct-link.ts)

**注意**：当前实现中 `createTemplateDirectLink` **不直接调用** `assertOrganisationRatesAndLimits` 或检查配额。但在 `getServerLimits` 中统计时会包含 direct templates：

[server.ts L97-L107](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/limits/server.ts#L97-L107)

```typescript
prisma.envelope.count({
  where: {
    type: EnvelopeType.TEMPLATE,
    directLink: { isNot: null },
  },
});
```

## 8. assertOrganisationRatesAndLimits 流程

[assert-organisation-rates-and-limits.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/rate-limit/assert-organisation-rates-and-limits.ts)

### 8.1 执行流程

```
assertOrganisationRatesAndLimits()
    │
    ├─► DANGEROUS_BYPASS_RATE_LIMITS === true → 跳过
    ├─► count === 0 → 无操作
    ├─► count < 0 → 抛出 INVALID_REQUEST
    ├─► 加载 organisationClaim（如未提供）
    ├─► 根据 type 提取 rateLimits 和 quota
    │    ├─► 'api' → apiRateLimits / apiQuota
    │    ├─► 'document' → documentRateLimits / documentQuota
    │    └─► 'email' → emailRateLimits / emailQuota
    ├─► checkOrganisationRateLimits() → 速率限制
    └─► checkMonthlyQuota() → 月度配额
```

### 8.2 与 Claim 的关联

`organisationClaim` 从数据库中读取，包含：
- `apiRateLimits` / `documentRateLimits` / `emailRateLimits`
- `apiQuota` / `documentQuota` / `emailQuota`

这些值在 Stripe webhook 触发时通过 `createOrganisationClaimUpsertData(claim)` 更新。

## 9. checkMonthlyQuota 与并发控制

[check-monthly-quota.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/rate-limit/check-monthly-quota.ts)

### 9.1 原子递增操作

```typescript
const latestMonthlyStat = await prisma.organisationMonthlyStat.upsert({
  where: { organisationId_period },
  update: { [column]: { increment: opts.count } },  // 原子递增
  create: { ... },
});
```

**关键**：使用 Prisma 的 `increment` 操作在数据库层面原子执行，保证并发安全。

### 9.2 只触发一次邮件通知的原理

```typescript
const newCount = latestMonthlyStat[column];
const previousCount = newCount - opts.count;

const isOverQuota = newCount > opts.quota;

// 只有当本次请求恰好跨越阈值时才触发
// previousCount <= quota < newCount
const didCrossQuota = isOverQuota && previousCount <= opts.quota;

if (didCrossQuota) {
  await jobsClient.triggerJob({
    name: 'send.organisation-limit-exceeded.email',
    // ...
  });
}
```

**并发保证的数学原理**：

1. **原子性**：数据库 `increment` 操作是原子的，每个请求获得唯一的、单调递增的 `newCount`
2. **区间唯一性**：对于阈值 Q，只有**恰好一个**请求的 `(previousCount, newCount]` 区间包含 Q
3. **单调性**：所有 post-increment 值严格有序，不会出现重复或乱序

**示例**：
- Q = 100
- 请求 A 递增后 newCount = 99, previousCount = 98 → 不触发
- 请求 B 递增后 newCount = 101, previousCount = 99 → didCrossQuota = true → 触发邮件
- 请求 C 递增后 newCount = 102, previousCount = 101 → 不触发

## 10. 关键分支与边界情况汇总

### 10.1 Webhook 处理分支

| 分支场景 | 处理方式 | 文件 |
|---------|---------|------|
| Billing 关闭 | 返回 500 | [handler.ts L26-L34](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/handler.ts#L26-L34) |
| webhookSecret 缺失 | 抛出 Error | [handler.ts L22-L24](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/handler.ts#L22-L24) |
| 无 stripe-signature | 返回 400 | [handler.ts L39-L47](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/handler.ts#L39-L47) |
| Payload 为空 | 返回 400 | [handler.ts L51-L59](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/handler.ts#L51-L59) |
| 多 subscription items | 500 错误 | [created L32-L42](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/on-subscription-created.ts#L32-L42) |
| Claim 不存在 | 500 错误 | [created L48-L58](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/on-subscription-created.ts#L48-L58) |
| 组织不存在 | 500 错误 | [created L167-L175](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/on-subscription-created.ts#L167-L175) |
| 已有活跃订阅 | 500 错误 | [created L178-L189](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/stripe/webhook/on-subscription-created.ts#L178-L189) |

### 10.2 组织类型转换

| 场景 | 原类型 | 新类型 | 条件 |
|-----|--------|--------|------|
| 个人购买 INDIVIDUAL | PERSONAL | PERSONAL | `claim.id === INDIVIDUAL` |
| 个人升级其他计划 | PERSONAL | ORGANISATION | claim 非 INDIVIDUAL / FREE |
| 新创建组织 | - | ORGANISATION | createOrganisation 默认 |

### 10.3 限流系统分支

| 分支场景 | 行为 | 文件 |
|---------|------|------|
| ENTERPRISE plan | 无限配额 | [server.ts L55-L61](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/limits/server.ts#L55-L61) |
| unlimitedDocuments flag | 无限配额 | [server.ts L74-L80](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/limits/server.ts#L74-L80) |
| INACTIVE subscription | INACTIVE_PLAN_LIMITS | [server.ts L64-L70](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/limits/server.ts#L64-L70) |
| FREE plan | FREE_PLAN_LIMITS | [server.ts L40-L41](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/limits/server.ts#L40-L41) |
| SELFHOSTED (billing off) | SELFHOSTED_PLAN_LIMITS | [server.ts L46-L52](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/limits/server.ts#L46-L52) |

## 11. 数据流向总结

```
Stripe Event
    │
    ▼
stripeWebhookHandler()  [签名校验]
    │
    ├─► customer.subscription.created
    │    └─► onSubscriptionCreated()
    │         ├─► handleOrganisationCreate() / handleOrganisationUpdate()
    │         │    └─► prisma.organisation  (create/update)
    │         │         └─► prisma.organisationClaim  (upsert)
    │         └─► prisma.subscription.upsert()
    │
    └─► customer.subscription.updated
         └─► onSubscriptionUpdated()
              ├─► prisma.subscription.update()
              └─► prisma.organisationClaim.update()  (if newClaimFound)
...
业务操作
    │
    ├─► createEnvelopeRouteCaller()
    │    └─► getServerLimits()
    │         └─► 读取 subscription + organisationClaim
    │
    └─► assertOrganisationRatesAndLimits()
         ├─► checkOrganisationRateLimits()
         └─► checkMonthlyQuota()
              ├─► 原子 increment
              └─► 阈值跨越检测 → send.organisation-limit-exceeded.email
```
