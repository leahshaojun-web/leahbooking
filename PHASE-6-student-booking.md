# PHASE 6 · 学员预约链接（公开页面）

## 前提

Phase 1-5 完成。

## 任务

实现可以直接发给学员的预约链接，**无需任何账号即可查看**。学员需要操作（预约/取消）时再用手机号 OTP 简单验证身份。

---

## URL 结构

```
公开访问（任何人都能看到课表）：
https://leahfit-scheduling.web.app/book/leahfit

某个学员的个人入口（带 token 自动识别）：
https://leahfit-scheduling.web.app/book/leahfit?u=USER_TOKEN

学员个人中心（已登录后）：
https://leahfit-scheduling.web.app/book/leahfit/my
```

---

## 步骤

### 1. 公开预约页 `/book/[studioSlug]/page.tsx`

**视觉**：100% 复刻你截图的样式（黄色头 + 横向日期 + 大图背景课程卡 + 底部 tab）。

**结构**：

```
┌──────────────────────────────────┐
│ [黄色 Header]                    │
│   约课管理                        │
├──────────────────────────────────┤
│ 课程表                  📅       │
│                                   │
│ [周五 8] [周六 今] [周日 10]     │
│ [周一 11✓] [周二 12] ...         │
│                                   │
├──────────────────────────────────┤
│ ┌──────────────────────────────┐ │
│ │ [大图] 普拉提小班    09:00   │ │
│ │       难度 ★★★★    10:00结束│ │
│ │                              │ │
│ │ [L] LeahFit          [代约]  │ │
│ │ 👥 2/3 满2人开课             │ │
│ └──────────────────────────────┘ │
│                                   │
│ ┌──────────────────────────────┐ │
│ │ [大图] 力量训练...            │ │
│ └──────────────────────────────┘ │
├──────────────────────────────────┤
│ [今天][课程✓][会员][报表][场馆] │
└──────────────────────────────────┘
```

**实现要点**：

- URL 参数读 studioSlug 和可选的 user token：
  ```typescript
  const params = useParams();
  const searchParams = useSearchParams();
  const studioSlug = params.studioSlug as string;
  const userToken = searchParams.get('u');
  ```

- 通过 slug 查 studio：
  ```typescript
  const studioQ = await getDocs(query(collection(db, 'studios'), where('slug', '==', studioSlug), limit(1)));
  if (studioQ.empty) return <NotFound />;
  const studio = studioQ.docs[0].data();
  ```

- 默认显示今天的课程，可以横向滑日期切换。日期 pill 可显示未来 14 天。

- 课程卡片用 ClassCard 组件，showAction 根据 user 状态：
  - 未登录 / 未识别：showAction='book'，点了之后弹手机号验证
  - 已识别且未预约这节课：showAction='book'
  - 已预约：showAction='booked'，再点弹"取消预约"
  - 课程已满：showAction='waitlist'

### 2. 学员身份验证（无需账号但可识别）

**两种方式**：

**A) 手机号 OTP（首次访问）**

```typescript
import { getAuth, RecaptchaVerifier, signInWithPhoneNumber } from 'firebase/auth';

async function startPhoneAuth(phone) {
  const verifier = new RecaptchaVerifier(auth, 'recaptcha-container', { size: 'invisible' });
  const confirmation = await signInWithPhoneNumber(auth, phone, verifier);
  // 学员输入验证码后：
  await confirmation.confirm(code);
  // 检查这个手机号是否在 users 中已存在
  // 是 → 关联 firebase auth uid 到 user 文档
  // 否 → 创建新的 user 文档（role: 'student'）
}
```

**B) 链接带 token（Admin 发给学员的链接）**

Admin 在学员详情页生成"专属预约链接"按钮：
```typescript
async function generateStudentLink(studentId) {
  const token = nanoid(20);
  await updateDoc(doc(db, 'users', studentId), {
    publicAccessToken: token,
    publicAccessTokenCreatedAt: serverTimestamp(),
  });
  return `https://leahfit-scheduling.web.app/book/${studio.slug}?u=${token}`;
}
```

学员通过这个链接进入时自动识别身份（不需要 OTP），但安全等级低（仅限于查看 + 预约自己的课，不能查看敏感数据）。如果做关键操作（取消、查看历史）仍触发 OTP。

### 3. 预约动作 `bookClass()`

```typescript
async function bookClass(classId, userId, category) {
  return await runTransaction(db, async (tx) => {
    // 1. 检查课程容量
    const classRef = doc(db, 'classes', classId);
    const classSnap = await tx.get(classRef);
    const classData = classSnap.data();
    if (classData.bookedCount >= classData.capacity) {
      // 加入候补
      const wRef = doc(collection(db, 'classes', classId, 'bookings'));
      tx.set(wRef, {
        userId, userName: user.name, userPhone: user.phone,
        status: 'waitlist', bookedAt: serverTimestamp(),
        creditDeducted: false,
      });
      tx.update(classRef, { waitlistCount: classData.waitlistCount + 1 });
      return { success: true, waitlist: true };
    }

    // 2. 检查学员课时余额
    const userRef = doc(db, 'users', userId);
    const userSnap = await tx.get(userRef);
    const balance = userSnap.data()?.creditBalances?.[category] || 0;
    if (balance < 1) {
      throw new Error('课时不足，请联系教练充值');
    }

    // 3. 检查是否已预约过这节课
    const existingQ = query(
      collection(db, 'classes', classId, 'bookings'),
      where('userId', '==', userId),
      where('status', 'in', ['booked', 'waitlist'])
    );
    const existing = await getDocs(existingQ); // 注意 getDocs 不能在 transaction 里用
    // 改用：先在 transaction 外查，再在 transaction 内插入
    if (!existing.empty) throw new Error('你已预约过这节课');

    // 4. 写入 booking
    const bRef = doc(collection(db, 'classes', classId, 'bookings'));
    tx.set(bRef, {
      userId, userName: user.name, userPhone: user.phone,
      status: 'booked', bookedAt: serverTimestamp(),
      creditDeducted: false, // 等到出席标记时才扣
    });

    // 5. 更新 class.bookedCount
    tx.update(classRef, { bookedCount: classData.bookedCount + 1 });

    return { success: true, waitlist: false };
  });
}
```

### 4. 取消预约

学员可以在课程开始前 N 小时取消（默认 4 小时，可在 studio 设置）。超过时限取消等同请假，可能扣课（按工作室策略）。

```typescript
async function cancelBooking(classId, bookingId) {
  const classDoc = await getDoc(doc(db, 'classes', classId));
  const startTime = classDoc.data().startTime.toDate();
  const hoursUntil = (startTime - new Date()) / (1000 * 60 * 60);

  if (hoursUntil < 4) {
    if (!confirm('距离开课不到 4 小时，取消可能会扣课时。确定？')) return;
  }

  await runTransaction(db, async (tx) => {
    const bRef = doc(db, 'classes', classId, 'bookings', bookingId);
    tx.update(bRef, { status: 'cancelled', cancelledAt: serverTimestamp() });
    const classRef = doc(db, 'classes', classId);
    tx.update(classRef, { bookedCount: classDoc.data().bookedCount - 1 });
    // 如果有候补，自动提升第一个
    // ...省略
  });
}
```

### 5. 学员个人中心 `/book/[studioSlug]/my`

**Tab 切换**：
- 我的预约（即将上的课）
- 历史记录（已结课）
- 课时余额（按 category 分类显示）

每个预约卡片：用 ClassCard，showAction='booked' 或 'attended'。

底部"联系教练"按钮（点击复制 Leah 微信号到剪贴板）。

### 6. 多语言切换

学员页右上角有「中 / EN」切换胶囊。所有文案准备 i18n：

```typescript
// src/lib/i18n.ts
export const t = {
  zh: {
    bookNow: '代约',
    waitlist: '代排队',
    booked: '已预约',
    full: '满',
    minToOpen: '满 {n} 人开课',
    classFull: '该课已满',
    insufficientCredits: '课时不足，请联系教练充值',
    // ...
  },
  en: { ... },
};
```

存学员选择的语言到 user.language。

### 7. PWA 配置（可选但推荐）

让学员能"添加到主屏幕"获得类似 App 体验：

`public/manifest.json`：
```json
{
  "name": "LeahFit 约课",
  "short_name": "LeahFit",
  "start_url": "/book/leahfit",
  "display": "standalone",
  "background_color": "#F5C842",
  "theme_color": "#5DB87C",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

iOS Safari 添加到主屏幕后，学员就有了一个看起来像原生 App 的入口。

---

## 完成判定

- 公开 URL `https://...web.app/book/leahfit` 任何人都能打开看到课表
- 视觉与你截图 100% 一致（黄色头、横向日期、大图课程卡、底部 tab）
- 课程图片正常显示（来自 Firebase Storage）
- 点击"代约"会要求手机号 OTP（首次）或直接预约（已识别）
- 预约成功后立即更新课程容量
- 已满时自动转候补
- /my 页面能看到自己的预约和余额
- 中英文切换工作

完成后告诉我："✅ Phase 6 完成"。
