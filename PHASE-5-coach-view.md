# PHASE 5 · 教练端（受限权限）

## 前提

Phase 1-4 完成。Admin 端能跑通完整流程。

## 任务

教练端是 Admin 端的精简版，权限受限。视觉风格统一（柔绿大圆角 + 大图背景课程卡），但只显示与教练相关的信息。

---

## 教练能做什么

✅ 查看自己当天/未来排的课
✅ 查看每节课的学员名单
✅ 标记出席（+ 自动扣课 / 退课）
✅ 添加课程备注
✅ 查看学员的简单信息（姓名、本课的备注）

❌ 不能改课程时间、价格、容量
❌ 不能新增/删除学员
❌ 不能看学员的课时余额（防止隐私泄露给其他教练）
❌ 不能改其他教练的课
❌ 不能生成微信提醒（这归 Admin）

---

## 步骤

### 1. 教练端布局 `/coach/layout.tsx`

移动端单栏，**永远是手机式布局**（即使桌面也居中 max-width: 430px）。

顶部 Header（黄色 banner，类似截图）：
- 左：LeahFit 图标 + "教练端"
- 右：教练姓名头像

底部 Tab 栏：
- 🏠 今日
- 📅 课表
- 👥 学员（仅查看，不能改）
- 👤 我的

### 2. 教练今日页 `/coach/page.tsx`

**顶部"深色绿背景"sticker**（参考之前的 OS 原型）：
```
┌────────────────────────────────────────┐
│ 今天 · Saturday, May 9                 │
│                                        │
│ 你今天有 6 节课                        │
│ 14 位学员 · 3 个房间                   │
│                                        │
│   3      2      1                      │
│ 已完成  即将  未签到                   │
└────────────────────────────────────────┘
```

**"下一节课"区块**：
- 标题：「下一节课 · 11:00 开始」
- 用 ClassCard 组件，但 expanded variant 显示学员名单

**"今日剩余课程"列表**：
- 小一点的 ClassCard
- 每张卡上有 [📋 名单] [✓ 一键签到] 双按钮

### 3. 教练课表 `/coach/classes`

**只显示教练自己的课**：
```typescript
query(collection(db, 'classes'),
  where('studioId', '==', studioId),
  where('coachId', '==', coachId),
  where('startTime', '>=', startOfWeek),
  where('startTime', '<', endOfWeek),
  orderBy('startTime'))
```

视图：周视图（横向 7 列）。

### 4. 教练课程详情 `/coach/classes/[id]`

跟 Admin 课程详情页**几乎相同**，但：
- 隐藏"添加学员"按钮
- 隐藏"移除学员"
- 隐藏课时余额信息（学员名单卡片不显示余额）
- 隐藏"提醒文案" tab
- 隐藏"标记课程已完成"按钮
- 备注 tab 仍可编辑
- 出席标记仍可使用（重要：扣课逻辑同样应用）

### 5. 教练学员页 `/coach/students`

**只列出"教练当前周排课中出现过的学员"**（不是全部学员）：

```typescript
// 拿到本周内 coach 所有 class 的所有 booking 学员去重
const myClasses = await getDocs(
  query(collection(db, 'classes'),
    where('coachId', '==', coachId),
    where('weekKey', '==', currentWeekKey))
);

const studentIds = new Set();
for (const c of myClasses.docs) {
  const bookings = await getDocs(collection(db, 'classes', c.id, 'bookings'));
  bookings.docs.forEach(b => studentIds.add(b.data().userId));
}

const students = await Promise.all([...studentIds].map(id => getDoc(doc(db, 'users', id))));
```

每条学员只显示：
- 姓名 + 头像
- 微信名（方便加好友 / 联系）
- 本周排在你课上的次数（"本周 3 节"）

**点击进入学员卡详情**（精简版）：
- 只显示姓名 / 微信 / 偏好语言 / 教练可见备注
- **不显示**：电话、邮箱、课时余额、其他教练的课、付费记录

### 6. 教练个人页 `/coach/me`

简单展示：
- 头像 / 姓名 / 简介 / 专长
- 退出登录按钮

### 7. 路由保护

`/coach/layout.tsx` 必须检查：

```typescript
useEffect(() => {
  return onAuthStateChanged(auth, async (user) => {
    if (!user) { router.push('/login'); return; }
    const userDoc = await getDoc(doc(db, 'users', user.uid));
    const role = userDoc.data()?.role;
    if (role !== 'coach' && role !== 'admin') {
      router.push('/login');
    }
  });
}, []);
```

Admin 也能访问 /coach 路径（用于 debug 或亲自代教学）。

---

## 完成判定

- Coach 邮箱登录后跳到 /coach
- 今日页只显示自己的课
- 课程详情页能正确标记出席并扣课
- 学员页只显示与自己相关的学员
- 不能访问 Admin 独有功能（点了报错或重定向）

完成后告诉我："✅ Phase 5 完成"。
