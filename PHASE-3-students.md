# PHASE 3 · 学员管理 + 课时充值/扣课

## 前提

Phase 2 完成。Admin 能登录、创建课程、看 Dashboard。

## 任务

实现学员管理整套流程：列表、详情、充值、扣课、查看历史。

---

## 步骤

### 1. 学员列表页（/admin/students）

**顶部工具栏**：
- 搜索框（按姓名/微信名/手机号实时筛选）
- Filter 胶囊按钮组：「全部」「Active」「Paused」「Inactive」「重点跟进」「课时≤2」
- 右上角"+ 新增学员"按钮

**列表**（白底圆角卡片，每行一个学员）：

```
┌──────────────────────────────────────────────┐
│ [头]  姓名（中文 / English）            [余]  │
│       微信：xxx · 手机：13xxxxxxxx       8节  │
│       Pilates / Strength            (按 cat) │
│       [Active] [需跟进]                       │
└──────────────────────────────────────────────┘
```

点击行进入详情页 `/admin/students/[id]`。

**搜索逻辑**：本地过滤即可（学员通常 < 200 人），不要每次都查 Firestore。

### 2. 学员详情页 `/admin/students/[id]`

**顶部信息卡**：
- 大头像 + 姓名 + 微信名 + 手机号 + 邮箱
- 加入时间 + 状态标签
- "编辑信息"按钮

**课时余额区**（核心）：

每种课程类型一张胶囊卡片，横向排列：
```
┌──────────┐ ┌──────────┐ ┌──────────┐
│ 普拉提   │ │ 力量训练 │ │ 私教     │
│   8 节   │ │   2 节   │ │   0 节   │
│ +充值    │ │ +充值    │ │ +充值    │
└──────────┘ └──────────┘ └──────────┘
```

≤2 节红色高亮，0 节灰色 + "已耗尽"。

**Tab 切换**：
- 📅 预约历史（按时间倒序，显示日期、课程名、状态）
- 💰 课时流水（充值/扣课/调整记录）
- 📝 备注（多行可编辑）
- 👤 个人信息（编辑姓名、电话、邮箱等）

### 3. 充值弹窗

点击任一课时类型的"+充值"按钮：

字段：
- 课程类型（默认填好，不可改）
- 充值类型：选择已有 creditPackage 或自定义
  - 选 package：显示包内容（10 节 / 90 天 / ¥XXX）
  - 自定义：手填节数、有效期天数、金额
- 备注（可选）
- 支付方式：现金 / 微信 / 转账 / 其他

提交时**用 Firestore Transaction**：
1. 创建 `creditTransactions` 文档（type: purchase, amount: +N, expiresAt）
2. 更新 `users/{id}.creditBalances[category]` += N
3. 更新 `users/{id}.updatedAt`

代码模板：

```typescript
import { runTransaction, doc, collection, serverTimestamp } from 'firebase/firestore';

await runTransaction(db, async (tx) => {
  const userRef = doc(db, 'users', userId);
  const userSnap = await tx.get(userRef);
  const balances = userSnap.data()?.creditBalances || {};
  const newBalance = (balances[category] || 0) + sessions;

  const txRef = doc(collection(db, 'creditTransactions'));
  tx.set(txRef, {
    studioId, userId, type: 'purchase', category,
    amount: sessions, balanceAfter: newBalance,
    packageId: pkgId || null,
    expiresAt: addDays(new Date(), validDays),
    notes, createdAt: serverTimestamp(), createdBy: adminId,
  });

  tx.update(userRef, {
    [`creditBalances.${category}`]: newBalance,
    updatedAt: serverTimestamp(),
  });
});
```

### 4. 手动扣课弹窗

字段：
- 课程类型
- 扣几节（默认 1）
- 原因（缺席未请假 / 临时请假超时 / 其他）
- 备注

逻辑同上但 amount 为负数，type: 'deduct'。

### 5. 新增学员弹窗

`/admin/students` 页的 "+ 新增学员" 按钮：

字段（参考你需求文档）：
- 姓名（必填）
- 英文名
- 微信名
- 手机号（必填，作为登录标识）
- Email
- 偏好语言：中文 / English（默认中文）
- 主要课程类型（多选）
- 会员状态（默认 Active）
- 备注
- 是否需要重点跟进

**关键**：创建学员时
1. 在 `users` collection 写入文档（先不创 Firebase Auth 账号，等学员第一次访问预约链接时再用手机号 OTP 登录创建）
2. 也可以勾选"立即发送预约链接"按钮，生成一条带 token 的链接，复制后通过微信发给学员

### 6. 编辑学员

点击详情页的"编辑信息"按钮，弹同款表单预填数据。

### 7. 列表性能优化

学员超过 50 人时分页：
- 每页 30 条
- 用 Firestore `startAfter` 游标
- 列表底部显示"加载更多"按钮

---

## 完成判定

- /admin/students 显示所有学员
- 搜索能实时过滤
- 点学员进详情页能看到完整信息
- 充值后余额立即更新
- 扣课后余额立即更新
- 新增学员后立即出现在列表

完成后告诉我："✅ Phase 3 完成"。
