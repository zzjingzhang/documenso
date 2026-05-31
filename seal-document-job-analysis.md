# `internal.seal-document` Job 执行流程分析

## 一、Job 触发与 Envelope 定位

### 1.1 从 legacy documentId 定位 Envelope

Job 接收的 payload 包含 `documentId`（legacy 数字 ID），通过以下机制定位 Envelope：

**映射函数**：[mapDocumentIdToSecondaryId](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/utils/envelope.ts#L182-L184)

```typescript
export const mapDocumentIdToSecondaryId = (documentId: number) => {
  return `${envelopeDocumentPrefixId}_${documentId}`; // 例如: document_123
};
```

**数据库查询**：[seal-document.handler.ts 第 41-71 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts#L41-L71)

```typescript
const envelope = await prisma.envelope.findFirstOrThrow({
  where: {
    type: EnvelopeType.DOCUMENT,
    secondaryId: mapDocumentIdToSecondaryId(documentId),
  },
  include: {
    user: { select: { name: true, email: true } },
    documentMeta: true,
    recipients: true,
    fields: { include: { signature: true } },
    envelopeItems: {
      include: {
        documentData: true,
        field: { include: { signature: true } },
      },
    },
  },
});
```

---

## 二、完成/拒签状态判断

### 2.1 CC 收件人预处理

[seal-document.handler.ts 第 83-91 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts#L83-L91)

在判断完成状态前，先将所有 CC 收件人标记为已签署：

```typescript
await prisma.recipient.updateMany({
  where: {
    envelopeId: envelope.id,
    role: RecipientRole.CC,
  },
  data: {
    signingStatus: SigningStatus.SIGNED,
  },
});
```

### 2.2 完成状态判断

[seal-document.handler.ts 第 93-103 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts#L93-L103)

文档被视为"完成"的两种情况：

1. **任意收件人拒签**（REJECTED）→ 文档整体拒签
2. **所有非 CC 收件人已签署**（SIGNED）→ 文档整体完成

```typescript
const isComplete =
  envelope.recipients.some((recipient) => recipient.signingStatus === SigningStatus.REJECTED) ||
  envelope.recipients.every(
    (recipient) => recipient.signingStatus === SigningStatus.SIGNED || recipient.role === RecipientRole.CC,
  );
```

### 2.3 未签必填字段检查

[seal-document.handler.ts 第 126-128 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts#L126-L128)

- **拒签文档**：跳过字段检查
- **完成文档**：必须所有必填字段都已插入

检查函数：[fieldsContainUnsignedRequiredField](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/utils/advanced-fields-helpers.ts#L49)

```typescript
export const fieldsContainUnsignedRequiredField = (fields: Field[]) => 
  fields.some(isFieldUnsignedAndRequired);
```

---

## 三、Re-seal 特殊处理

### 3.1 initialData 恢复

[seal-document.handler.ts 第 130-140 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts#L130-L140)

重新密封时，使用 `initialData` 作为基础，避免字段叠加：

```typescript
if (isResealing) {
  envelopeItems = envelopeItems.map((envelopeItem) => ({
    ...envelopeItem,
    documentData: {
      ...envelopeItem.documentData,
      data: envelopeItem.documentData.initialData,
    },
  }));
}
```

### 3.2 qrToken 生成

[seal-document.handler.ts 第 142-151 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts#L142-L151)

如果 Envelope 没有 qrToken，生成一个：

```typescript
if (!envelope.qrToken) {
  await prisma.envelope.update({
    where: { id: envelope.id },
    data: { qrToken: prefixedId('qr') },
  });
}
```

---

## 四、DOCUMENT_COMPLETED 审计日志

[seal-document.handler.ts 第 154-163 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts#L154-L163)

```typescript
const envelopeCompletedAuditLog = createDocumentAuditLogData({
  type: DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_COMPLETED,
  envelopeId: envelope.id,
  requestMetadata,
  user: null,
  data: {
    transactionId: nanoid(),
    ...(isRejected ? { isRejected: true, rejectionReason: rejectionReason } : {}),
  },
});
```

**持久化时机**：在数据库事务中创建 [第 285-287 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts#L285-L287)

---

## 五、证书与审计日志附加页

### 5.1 配置检查

[seal-document.handler.ts 第 179-180 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts#L179-L180)

```typescript
const needsCertificate = settings.includeSigningCertificate;
const needsAuditLog = settings.includeAuditLog;
```

### 5.2 PDF 生成与附加

[seal-document.handler.ts 第 194-246 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts#L194-L246)

根据配置，使用 Playwright 或传统方式生成：

```typescript
const makeCertificatePdf = async () =>
  usePlaywrightPdf
    ? getCertificatePdf({ documentId, language }).then(buffer => PDF.load(buffer))
    : generateCertificatePdf(certificatePayload);

const makeAuditLogPdf = async () =>
  usePlaywrightPdf
    ? getAuditLogsPdf({ documentId, language }).then(buffer => PDF.load(buffer))
    : generateAuditLogPdf(certificatePayload);
```

**页面复制**：在 `decorateAndSignPdf` 函数中 [第 367-379 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts#L367-L379)

```typescript
if (certificateDoc) {
  await pdfDoc.copyPagesFrom(certificateDoc, pageIndices);
}

if (auditLogDoc) {
  await pdfDoc.copyPagesFrom(auditLogDoc, pageIndices);
}
```

---

## 六、V1/V2 字段插入与 PDF 旋转

### 6.1 V1 字段插入（旧版）

[seal-document.handler.ts 第 382-400 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts#L382-L400)

```typescript
if (envelope.internalVersion === 1) {
  const legacy_pdfLibDoc = await PDFDocument.load(await pdfDoc.save({ useXRefStream: true }));
  
  for (const field of envelopeItemFields) {
    if (field.inserted) {
      if (envelope.useLegacyFieldInsertion) {
        await legacy_insertFieldInPDF(legacy_pdfLibDoc, field);
      } else {
        await insertFieldInPDFV1(legacy_pdfLibDoc, field);
      }
    }
  }
  
  legacy_pdfLibDoc.getForm().flatten();
  await pdfDoc.reload(await legacy_pdfLibDoc.save());
}
```

### 6.2 V2 字段插入（新版）

[seal-document.handler.ts 第 403-454 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts#L403-L454)

V2 使用 [insertFieldInPDFV2](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/pdf/insert-field-in-pdf-v2.ts#L17-L57) 生成覆盖层：

```typescript
if (envelope.internalVersion === 2) {
  const fieldsGroupedByPage = groupBy(envelopeItemFields, (field) => field.page);
  
  for (const [pageNumber, fields] of Object.entries(fieldsGroupedByPage)) {
    const page = pdfDoc.getPage(Number(pageNumber) - 1);
    const pageWidth = page.width;
    const pageHeight = page.height;
    
    const overlayBytes = await insertFieldInPDFV2({ pageWidth, pageHeight, fields });
    const overlayPdf = await PDF.load(overlayBytes);
    const embeddedPage = await pdfDoc.embedPage(overlayPdf, 0);
    
    // 旋转处理
    let translateX = 0;
    let translateY = 0;
    switch (page.rotation) {
      case 90:  translateX = pageHeight; break;
      case 180: translateX = pageWidth; translateY = pageHeight; break;
      case 270: translateY = pageWidth; break;
    }
    
    page.drawPage(embeddedPage, {
      x: translateX,
      y: translateY,
      rotate: { angle: page.rotation },
    });
  }
}
```

**PDF 旋转处理逻辑**：
- **0°**：无需偏移 (0, 0)
- **90°**：X 偏移 = 页面高度
- **180°**：X 偏移 = 页面宽度，Y 偏移 = 页面高度
- **270°**：Y 偏移 = 页面宽度

---

## 七、PKI 签名 (signPdf)

[signPdf 函数](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/signing/index.ts#L38-L54)

```typescript
export const signPdf = async ({ pdf }: SignOptions) => {
  const signer = await getSigner();
  const tsa = getTimestampAuthority();
  
  const { bytes } = await pdf.sign({
    signer,
    reason: 'Signed by Documenso',
    location: NEXT_PUBLIC_WEBAPP_URL(),
    contactInfo: NEXT_PUBLIC_SIGNING_CONTACT_INFO(),
    subFilter: NEXT_PRIVATE_USE_LEGACY_SIGNING_SUBFILTER() 
      ? 'adbe.pkcs7.detached' 
      : 'ETSI.CAdES.detached',
    timestampAuthority: tsa ?? undefined,
    longTermValidation: !!tsa,
    archivalTimestamp: !!tsa,
  });
  
  return bytes;
};
```

**签名配置**：
- **签名传输**：local 或 gcloud-hsm
- **SubFilter**：`ETSI.CAdES.detached`（默认）或 `adbe.pkcs7.detached`（旧版兼容）
- **时间戳**：配置 TSA 时启用 LTV 和归档时间戳

---

## 八、DocumentData 替换

[seal-document.handler.ts 第 262-273 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts#L262-L273)

在事务中更新每个 EnvelopeItem 的 documentDataId：

```typescript
await prisma.$transaction(async (tx) => {
  for (const { oldDocumentDataId, newDocumentDataId } of newDocumentData) {
    await tx.envelopeItem.update({
      where: {
        envelopeId: envelope.id,
        documentDataId: oldDocumentDataId,
      },
      data: {
        documentDataId: newDocumentDataId,
      },
    });
  }
  
  // 更新 Envelope 状态
  await tx.envelope.update({
    where: { id: envelope.id },
    data: {
      status: finalEnvelopeStatus,
      completedAt: new Date(),
    },
  });
  
  // 创建审计日志
  await tx.documentAuditLog.create({ data: envelopeCompletedAuditLog });
});
```

---

## 九、Webhook 与邮件发送

### 9.1 Webhook 触发

[seal-document.handler.ts 第 307-312 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts#L307-L312)

```typescript
await triggerWebhook({
  event: isRejected 
    ? WebhookTriggerEvents.DOCUMENT_REJECTED 
    : WebhookTriggerEvents.DOCUMENT_COMPLETED,
  data: ZWebhookDocumentSchema.parse(mapEnvelopeToWebhookDocumentPayload(updatedEnvelope)),
  userId: updatedEnvelope.userId,
  teamId: updatedEnvelope.teamId ?? undefined,
});
```

### 9.2 完成邮件发送

[seal-document.handler.ts 第 314-328 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts#L314-L328)

发送条件判断：

```typescript
let shouldSendCompletedEmail = sendEmail && !isResealing && !isRejected;

// 特殊情况：重新密封且之前未完成的，发送邮件
if (isResealing && !isDocumentCompleted(envelopeStatus)) {
  shouldSendCompletedEmail = sendEmail;
}

if (shouldSendCompletedEmail) {
  await jobs.triggerJob({
    name: 'send.document.completed.emails',
    payload: { envelopeId, requestMetadata },
  });
}
```

---

## 十、关键差异对比：sealDocument vs. Pending Partial Signed PDF

| 维度 | sealDocument（最终密封） | generatePartialSignedPdf（预览下载） |
|------|--------------------------|--------------------------------------|
| **代码位置** | [seal-document.handler.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/jobs/definitions/internal/seal-document.handler.ts) | [generate-partial-signed-pdf.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/pdf/generate-partial-signed-pdf.ts) |

### 10.1 证书 (Certificate) 差异

| 特性 | sealDocument | Pending PDF |
|------|--------------|-------------|
| **证书页生成** | ✅ 根据 `includeSigningCertificate` 配置生成 | ❌ 不生成 |
| **证书内容** | 包含完整签署信息、交易 ID、QR token、所有签署人信息 | - |
| **Playwright 渲染** | 支持 Playwright 或传统方式生成 | - |
| **文档附加** | 附加在 PDF 末尾 | - |

### 10.2 审计日志 (Audit Log) 差异

| 特性 | sealDocument | Pending PDF |
|------|--------------|-------------|
| **审计日志页** | ✅ 根据 `includeAuditLog` 配置生成 | ❌ 不生成 |
| **日志内容** | 包含完整操作历史 + DOCUMENT_COMPLETED 事件 | - |
| **Playwright 渲染** | 支持 Playwright 或传统方式生成 | - |
| **文档附加** | 附加在证书页之后（或末尾） | - |

### 10.3 PKI 签名差异

| 特性 | sealDocument | Pending PDF |
|------|--------------|-------------|
| **PKI 数字签名** | ✅ 完整 PKI 签名（调用 `signPdf`） | ❌ 无 PKI 签名 |
| **签名传输层** | local / gcloud-hsm | - |
| **SubFilter** | ETSI.CAdES.detached / adbe.pkcs7.detached | - |
| **时间戳 (TSA)** | 配置时启用 LTV 和归档时间戳 | - |
| **签名元数据** | reason、location、contactInfo 完整 | - |

### 10.4 其他关键差异

| 特性 | sealDocument | Pending PDF |
|------|--------------|-------------|
| **使用场景** | 文档最终完成/拒签后生成的正式文档 | PENDING 状态下的临时预览 |
| **PDF 版本** | 升级到 PDF 1.7 | 升级到 PDF 1.7 |
| **Flatten** | ✅ 多次 flatten（字段插入前后） | ✅ 多次 flatten |
| **拒签章** | ✅ 拒签文档添加拒签章 | ❌ 不添加 |
| **文件名后缀** | `_signed.pdf` / `_rejected.pdf` | 无特定后缀 |
| **V1 字段支持** | ✅ 支持 V1 和 V2 两种方式 | ❌ 仅支持 V2 方式 |
| **initialData** | Re-seal 时恢复 initialData | 直接使用当前 data |

---

## 十一、完整执行流程图

```
internal.seal-document Job 触发
        │
        ▼
  ┌─────────────────┐
  │ 定位 Envelope   │  mapDocumentIdToSecondaryId(document_123)
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ CC 收件人处理   │  全部标记为 SIGNED
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ 完成状态判断    │  拒签? 或 全部签署?
  └────────┬────────┘
           │ 不完整则抛出错误
           ▼
  ┌─────────────────┐
  │ 必填字段检查    │  拒签跳过检查
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ Re-seal 处理    │  恢复 initialData
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ 生成 qrToken    │  如不存在
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ 预取 PDF 数据   │  所有 envelopeItems
  └────────┬────────┘
           │
           ▼
  ┌─────────────────────────────────────────────┐
  │ 处理每个 EnvelopeItem:                      │
  │  ┌────────────────────────────────────────┐ │
  │  │ 1. 生成证书页 (配置开启)               │ │
  │  │ 2. 生成审计日志页 (配置开启)           │ │
  │  │ 3. decorateAndSignPdf:                 │ │
  │  │    - PDF 标准化 (flatten, v1.7)        │ │
  │  │    - 添加拒签章 (拒签时)               │ │
  │  │    - 附加证书页/审计日志页             │ │
  │  │    - V1/V2 字段插入 + 旋转处理         │ │
  │  │    - PKI 签名 (signPdf)                │ │
  │  └────────────────────────────────────────┘ │
  └──────────────────┬──────────────────────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │ 数据库事务更新:       │
          │ - DocumentData 替换  │
          │ - Envelope 状态更新  │
          │ - 审计日志创建       │
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │ Webhook 触发         │  COMPLETED / REJECTED
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │ 发送完成邮件         │  (条件满足时)
          └──────────────────────┘
```
