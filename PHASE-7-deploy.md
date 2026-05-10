# PHASE 7 · Firebase Hosting 部署

## 前提

Phase 1-6 完成。所有功能本地测试通过。

## 任务

部署到 Firebase Hosting，配置自定义域名（可选），把 production URL 给到 Leah。

---

## 步骤

### 1. 安装 Firebase CLI

```bash
npm install -g firebase-tools
firebase login
```

### 2. 初始化 Firebase Hosting

```bash
firebase init hosting
```

选择：
- 已有项目：`leahfit-scheduling`（你之前创建的）
- public 目录：`out` （Next.js export 输出）
- 单页应用 (SPA)：**No**（我们用 Next.js 自己的路由）
- 自动部署 GitHub Actions：可选

### 3. 配置 Next.js 静态导出

修改 `next.config.js`：

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'export',
  images: {
    unoptimized: true,
    remotePatterns: [
      { protocol: 'https', hostname: 'firebasestorage.googleapis.com' },
    ],
  },
  trailingSlash: true,
};

module.exports = nextConfig;
```

### 4. 修改 `firebase.json`

```json
{
  "hosting": {
    "public": "out",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [
      { "source": "**", "destination": "/index.html" }
    ],
    "headers": [
      {
        "source": "**/*.@(js|css)",
        "headers": [
          { "key": "Cache-Control", "value": "public,max-age=31536000,immutable" }
        ]
      },
      {
        "source": "**/*.@(jpg|jpeg|png|webp|svg|woff2)",
        "headers": [
          { "key": "Cache-Control", "value": "public,max-age=2592000" }
        ]
      }
    ]
  },
  "firestore": {
    "rules": "firestore.rules",
    "indexes": "firestore.indexes.json"
  },
  "storage": {
    "rules": "storage.rules"
  }
}
```

### 5. 部署 Firestore 规则和索引

把 PHASE 0 文档里的 Firestore 安全规则保存到 `firestore.rules`。

把 Storage 规则保存到 `storage.rules`。

把索引保存到 `firestore.indexes.json`：

```json
{
  "indexes": [
    {
      "collectionGroup": "classes",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "studioId", "order": "ASCENDING" },
        { "fieldPath": "date", "order": "ASCENDING" },
        { "fieldPath": "startTime", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "classes",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "studioId", "order": "ASCENDING" },
        { "fieldPath": "weekKey", "order": "ASCENDING" },
        { "fieldPath": "startTime", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "classes",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "studioId", "order": "ASCENDING" },
        { "fieldPath": "coachId", "order": "ASCENDING" },
        { "fieldPath": "date", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "users",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "studioId", "order": "ASCENDING" },
        { "fieldPath": "role", "order": "ASCENDING" },
        { "fieldPath": "name", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "creditTransactions",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "userId", "order": "ASCENDING" },
        { "fieldPath": "category", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "bookings",
      "queryScope": "COLLECTION_GROUP",
      "fields": [
        { "fieldPath": "userId", "order": "ASCENDING" },
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "bookedAt", "order": "DESCENDING" }
      ]
    }
  ],
  "fieldOverrides": []
}
```

部署：
```bash
firebase deploy --only firestore:rules,firestore:indexes,storage
```

### 6. 部署应用

```bash
npm run build
firebase deploy --only hosting
```

### 7. 验证

打开 Firebase 给的 URL（类似 `https://leahfit-scheduling.web.app`），测试：
- 首页加载
- /login 能登录
- /admin Dashboard 显示数据
- /book/leahfit 公开预约页能打开
- 图片从 Firebase Storage 正常加载

### 8. （可选）自定义域名

如果有自己的域名（比如 `book.leahfit.com`）：

1. Firebase Console > Hosting > 添加自定义域名
2. 按提示在域名 DNS 添加 A/AAAA/TXT 记录
3. 等 SSL 证书自动签发（24 小时内）
4. 测试域名生效

### 9. 准备给学员的入口

部署成功后会拿到：
- Admin 入口：`https://leahfit-scheduling.web.app/login`
- 学员预约链接：`https://leahfit-scheduling.web.app/book/leahfit`

把学员链接生成二维码，可以放在工作室门口、印在名片上、发到微信群。

### 10. 设置 Firebase Auth 邮箱白名单

为了安全，在 Firebase Console > Authentication 限制：
- 只用 Email/Password + Phone（用于学员 OTP）
- 关闭其他登录方式
- 把 Leah 和教练的邮箱手动创建用户，再让 Replit 在 users collection 写入对应的 admin/coach 文档（用相同的 UID）

---

## 完成判定

- 部署成功，URL 可访问
- Firestore 规则生效（未授权请求被拒）
- Storage 规则生效（图片可读，未授权写被拒）
- 所有 phase 的功能在 production 环境正常运行
- 没有 console 错误

完成后告诉我："✅ Phase 7 完成"，并提供：
- Production URL
- Admin 登录邮箱
- 学员预约链接示例
- 二维码 PNG（可选）

---

## 后续维护

每次更新代码：
```bash
npm run build
firebase deploy --only hosting
```

监控：
- Firebase Console > Performance：监控页面加载速度
- Firebase Console > Analytics：看用户行为
- Firebase Console > Firestore > Usage：留意读写次数（免费配额：每日 50K 读 / 20K 写）

如果 Firestore 接近免费配额上限，需要：
- 优化查询（多用本地 cache）
- 减少冗余字段更新
- 考虑升级到付费计划（已经升级 Blaze 了，所以会按用量收费）
