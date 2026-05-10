# PHASE 4 · 课程详情页 + 出席标记 + 微信提醒生成

## 前提

Phase 1-3 完成。

## 任务

实现课程详情页（Admin/Coach 共用），所有出席标记逻辑，以及微信提醒文案生成功能。

---

## 步骤

### 1. 课程详情页 `/admin/classes/[id]`

**顶部**：
- 大背景图（用 ClassCard 视觉风格放大版，高度 240px）
- 上面叠加：课程名、时间、教练、房间、难度

**Tab 切换**：
- 📋 学员名单
- ⏳ 候补名单
- 💬 提醒文案
- 📝 备注

#### Tab 1: 学员名单

每行一个 booking：

```
┌───────────────────────────────────────┐
│ [头] 姓名 (中文/EN)         [出席状态]│
│      余 N 节 · 上次 5/8 出席   ↘ 切换 │
└───────────────────────────────────────┘
```

出席状态按钮组（4 选 1）：
- ✓ 已出席（绿色实色）
- 🛌 请假（黄色）
- ✗ 缺席（红色）
- — 未标记（灰色）

切换状态时的逻辑：

```typescript
async function markAttendance(bookingId, classId, newStatus) {
  await runTransaction(db, async (tx) => {
    const bRef = doc(db, 'classes', classId, 'bookings', bookingId);
    const bSnap = await tx.get(bRef);
    const booking = bSnap.data();
    const oldStatus = booking.status;
    const wasDeducted = booking.creditDeducted;

    // 决定新状态下是否应该扣课
    // 规则：attended / absent (无故缺席) → 扣
    //       leave_taken (请假) → 不扣
    //       cancelled (已取消) → 不扣
    const shouldDeduct = ['attended', 'absent'].includes(newStatus);

    if (shouldDeduct && !wasDeducted) {
      // 扣课
      await deductCredit(tx, booking.userId, classData.category, classId, bookingId);
      tx.update(bRef, { status: newStatus, creditDeducted: true,
        attendanceMarkedAt: serverTimestamp(), attendanceMarkedBy: currentUserId });
    } else if (!shouldDeduct && wasDeducted) {
      // 退课时
      await refundCredit(tx, booking.userId, classData.category, classId, bookingId);
      tx.update(bRef, { status: newStatus, creditDeducted: false,
        attendanceMarkedAt: serverTimestamp(), attendanceMarkedBy: currentUserId });
    } else {
      tx.update(bRef, { status: newStatus, attendanceMarkedAt: serverTimestamp() });
    }
  });
}
```

**底部操作按钮**：
- "+ 添加学员"（弹搜索框选学员，从已签到名单/候补/手动输入）
- "一键签到全员"（把所有 booked 状态批量改为 attended）
- "标记课程已完成"（改 class.status = completed, attendanceMarked = true）

#### Tab 2: 候补名单

显示 status='waitlist' 的预约。Admin 可以：
- 提升到正式名单（如果有空位）
- 移除候补

#### Tab 3: 提醒文案

参考下面 "微信提醒生成器" 章节。

#### Tab 4: 备注

简单的多行 textarea，自动保存（debounce 1s）。

### 2. 微信提醒生成器

#### 模板配置（先用硬编码，未来可在 settings 页编辑）

```typescript
// src/lib/reminders.ts

export const REMINDER_TEMPLATES = {
  individual_zh: `Hi {{name}}，提醒你 {{date}} {{time}} 有 {{className}}。
地点：{{studio}} · {{room}}
请提前 5 分钟到达。
如果需要请假，请尽早告诉我。
— Leah`,

  group_zh: `{{date}}课程提醒 ☀️

{{classList}}

请大家提前 5 分钟到达 ❤️
有事请提前请假 — Leah`,

  individual_en: `Hi {{name}}, reminder you have {{className}} on {{date}} at {{time}}.
Venue: {{studio}} · {{room}}
Please arrive 5 mins early.
Let me know in advance if you need to cancel.
— Leah`,

  group_en: `{{date}} Class Reminders ☀️

{{classList}}

Please arrive 5 minutes early ❤️
Let me know if you can't make it — Leah`,
};

export function generateIndividualReminder(booking, classData, studio, language='zh') {
  const tpl = REMINDER_TEMPLATES[`individual_${language}`];
  const date = format(classData.startTime.toDate(), language === 'zh' ? 'M月d日' : 'MMM d');
  const time = format(classData.startTime.toDate(), 'HH:mm');
  return tpl
    .replace('{{name}}', booking.userName)
    .replace('{{date}}', date)
    .replace('{{time}}', time)
    .replace('{{className}}', language === 'zh' ? classData.nameZh : classData.nameEn)
    .replace('{{studio}}', studio.name)
    .replace('{{room}}', classData.room);
}

export function generateGroupReminder(classes, studio, language='zh') {
  const tpl = REMINDER_TEMPLATES[`group_${language}`];
  const date = format(classes[0].startTime.toDate(), language === 'zh' ? 'M月d日' : 'MMM d');
  const classList = classes.map(c => {
    const time = format(c.startTime.toDate(), 'HH:mm');
    const name = language === 'zh' ? c.nameZh : c.nameEn;
    const names = c.bookings.map(b => b.userName).join(' / ');
    return language === 'zh'
      ? `${time} ${name}\n学员：${names}\n地点：${studio.name} ${c.room}`
      : `${time} ${name}\nStudents: ${names}\nVenue: ${studio.name} ${c.room}`;
  }).join('\n\n');
  return tpl.replace('{{date}}', date).replace('{{classList}}', classList);
}
```

#### 提醒文案 UI

在课程详情 Tab 3 和 Admin 端 `/admin/reminders` 页都用同一个组件：

```
┌────────────────────────────────────────────┐
│ 模式: ◉ 单人提醒  ◯ 群体提醒              │
│ 语言: ◉ 中文     ◯ English                 │
│ ┌──────────────────────────────────────┐  │
│ │ [文案预览，深色背景，monospace]       │  │
│ │                                       │  │
│ │ Hi Amy，提醒你...                     │  │
│ └──────────────────────────────────────┘  │
│ [📋 复制文案]  [↻ 重新生成]                │
└────────────────────────────────────────────┘
```

复制按钮逻辑：
```typescript
await navigator.clipboard.writeText(text);
toast('已复制到剪贴板');
// 同时记录到 reminders collection
await addDoc(collection(db, 'reminders'), {
  studioId, classId, type, recipientUserIds,
  generatedText: text, language,
  copiedAt: serverTimestamp(),
  generatedAt: serverTimestamp(),
  generatedBy: currentUserId,
});
// 更新 class.reminderSent = true
await updateDoc(doc(db, 'classes', classId), {
  reminderSent: true,
  reminderSentAt: serverTimestamp(),
});
```

### 3. Admin 提醒中心 `/admin/reminders`

集中页面：
- 顶部 tab：明日 / 今日 / 自定义日期
- 当天所有课程列表，每条课程一张卡片
  - 显示：时间、课程名、学员名单、提醒状态
  - 操作：
    - 「生成单人提醒」展开 N 条单独的文案（每条独立的复制按钮）
    - 「生成群体提醒」一键合并整天文案
- 顶部按钮"一键生成今日全部提醒"（生成所有未发送提醒的课程文案）

### 4. 学员预约/取消的扣课逻辑

放在 PHASE 6 实现学员预约时再用，先在 `src/lib/credits.ts` 准备好辅助函数：

```typescript
export async function deductCredit(tx, userId, category, classId, bookingId) {
  const userRef = doc(db, 'users', userId);
  const userSnap = await tx.get(userRef);
  const balances = userSnap.data()?.creditBalances || {};
  const current = balances[category] || 0;
  const newBalance = current - 1;

  const txRef = doc(collection(db, 'creditTransactions'));
  tx.set(txRef, {
    studioId, userId, type: 'deduct', category,
    amount: -1, balanceAfter: newBalance,
    classId, bookingId,
    notes: '课时扣除',
    createdAt: serverTimestamp(), createdBy: 'system',
  });
  tx.update(userRef, {
    [`creditBalances.${category}`]: newBalance,
    updatedAt: serverTimestamp(),
  });
  return txRef.id;
}

export async function refundCredit(tx, userId, category, classId, bookingId) {
  // 同上但 amount: +1, type: 'refund'
}
```

---

## 完成判定

- 进入课程详情页能看到完整学员名单和候补
- 切换出席状态会自动扣课/退课，余额实时变化
- 提醒文案能正确生成（中英文、单人/群体）
- 复制按钮工作，记录写入 reminders 集合
- /admin/reminders 中心页能看到当天所有课程的提醒生成入口

完成后告诉我："✅ Phase 4 完成"。
