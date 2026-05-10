# Firestore 数据结构 · LeahFit Scheduling OS

## Collections 总览

```
firestore/
├── studios/{studioId}              · 工作室配置
├── users/{userId}                  · 所有用户（admin / coach / student）
├── courseTypes/{typeId}            · 课程类型（普拉提/力量/产后等）
├── classes/{classId}               · 排好的课（具体某天某时间一节课）
│   └── bookings/{bookingId}        · 子集合：这节课的预约记录
├── creditPackages/{packageId}      · 课时包定义
├── creditTransactions/{txId}       · 充值/扣课流水
└── reminders/{reminderId}          · 微信提醒生成记录
```

---

## 1. studios（工作室）

```typescript
{
  id: string,                       // "leahfit-melbourne"
  name: string,                     // "LeahFit Studio"
  slug: string,                     // 公开预约链接用 "leahfit"
  timezone: string,                 // "Australia/Melbourne"
  rooms: string[],                  // ["Studio A", "Studio B", "Strength Room"]
  language: "zh" | "en" | "both",
  brandColor: string,               // "#5DB87C"
  logoUrl: string,                  // Firebase Storage URL
  createdAt: Timestamp,
}
```

---

## 2. users

```typescript
{
  id: string,                       // Firebase Auth UID
  role: "admin" | "coach" | "student",
  studioId: string,                 // 所属工作室

  // 通用
  name: string,                     // "Leah Li"
  nameEn?: string,                  // "Leah"
  wechatName?: string,              // "Leah_Pilates"
  phone?: string,                   // 学员主要用手机号
  email?: string,                   // Admin/Coach 主要用邮箱
  avatarUrl?: string,
  language: "zh" | "en",            // 用户偏好

  // 学员专属
  membershipStatus?: "active" | "paused" | "inactive",
  notes?: string,                   // Admin 的备注
  needsFollowUp?: boolean,          // 重点跟进标记
  joinedAt?: Timestamp,

  // 教练专属
  specialty?: string[],             // ["pilates", "strength"]
  bio?: string,

  createdAt: Timestamp,
  updatedAt: Timestamp,
}
```

---

## 3. courseTypes（课程类型）

```typescript
{
  id: string,                       // "pilates-mat"
  studioId: string,
  nameZh: string,                   // "普拉提小班课"
  nameEn: string,                   // "Pilates Mat"
  category: "pilates" | "strength" | "pt" | "kids" | "postnatal" | "bungee" | "yoga",
  defaultDuration: number,          // 分钟，60
  defaultCapacity: number,          // 3
  minToOpen: number,                // 最少几人开课，1 (PT) / 2 (Pair) / 3+ (Class)
  difficulty: 1 | 2 | 3 | 4 | 5,    // 难度星级
  defaultRoom: string,              // "Studio A"
  coverImageUrl: string,            // 默认封面图，Firebase Storage URL
  creditCost: number,               // 扣几节课时，通常 1
  description?: string,
  isActive: boolean,
  sortOrder: number,
  createdAt: Timestamp,
}
```

---

## 4. classes（排好的具体课程）

```typescript
{
  id: string,                       // 自动生成
  studioId: string,
  courseTypeId: string,             // 关联课程类型
  coachId: string,                  // 关联 users (role=coach)

  // 时间
  startTime: Timestamp,             // 开始时间（UTC）
  endTime: Timestamp,               // 结束时间
  date: string,                     // "2026-05-11" 用于按日查询
  weekKey: string,                  // "2026-W19" 用于按周查询

  // 课程信息（可覆盖 courseType 的默认值）
  nameZh: string,                   // 冗余存储，方便查询不用 join
  nameEn: string,
  category: string,                 // 同上
  difficulty: number,
  capacity: number,                 // 这节课的最大人数
  minToOpen: number,
  room: string,
  coverImageUrl: string,            // 这节课特定的封面图（可覆盖类型默认）

  // 状态
  status: "scheduled" | "confirmed" | "cancelled" | "completed",
  bookedCount: number,              // 已预约人数（冗余，方便排序）
  waitlistCount: number,
  attendanceMarked: boolean,        // 是否已经标记完出席

  // 循环
  recurringId?: string,             // 如果是循环生成的，指向模板 ID
  isRecurring: boolean,

  notes?: string,                   // Admin/Coach 的备注

  // 提醒
  reminderSent: boolean,
  reminderSentAt?: Timestamp,

  createdAt: Timestamp,
  updatedAt: Timestamp,
  createdBy: string,                // userId
}
```

### 4a. classes/{classId}/bookings 子集合

```typescript
{
  id: string,                       // 自动生成
  userId: string,                   // 学员 ID
  userName: string,                 // 冗余存储
  userPhone?: string,               // 冗余存储

  status: "booked" | "waitlist" | "attended" | "absent" | "leave_taken" | "cancelled",
  bookedAt: Timestamp,
  cancelledAt?: Timestamp,
  attendanceMarkedAt?: Timestamp,
  attendanceMarkedBy?: string,      // adminId or coachId

  creditDeducted: boolean,          // 是否已扣课
  creditTxId?: string,              // 关联 creditTransactions

  notes?: string,                   // 这次预约的备注（学员留言或 Coach 备注）
}
```

---

## 5. creditPackages（课时包定义）

```typescript
{
  id: string,
  studioId: string,
  nameZh: string,                   // "普拉提 10 节包"
  category: string,                 // 限定课程类型
  totalSessions: number,            // 10
  validDays: number,                // 有效期天数，90
  price: number,                    // 单位：分（cents），方便计算
  currency: "AUD" | "CNY",
  isActive: boolean,
  sortOrder: number,
}
```

---

## 6. creditTransactions（课时流水）

```typescript
{
  id: string,
  studioId: string,
  userId: string,                   // 学员 ID

  type: "purchase" | "deduct" | "refund" | "adjust" | "expire",
  category: string,                 // 限定的课程类型，"all" 表示通用
  amount: number,                   // 正数=加，负数=减
  balanceAfter: number,             // 该 category 流水后的余额

  // 关联信息
  packageId?: string,               // 充值时关联
  classId?: string,                 // 扣课时关联
  bookingId?: string,

  expiresAt?: Timestamp,            // 充值的有效期
  notes?: string,
  createdAt: Timestamp,
  createdBy: string,                // adminId
}
```

### 学员当前余额查询逻辑

学员的余额不单独存字段，而是按 category 实时聚合 `creditTransactions`：
```
balance(userId, category) = SUM(amount where userId=X and category=Y and not expired)
```

为了性能，可以在 `users/{userId}` 下挂一个冗余字段：
```typescript
creditBalances: {
  pilates: number,
  strength: number,
  pt: number,
  // ...
}
```
每次 transaction 写入时同步更新这个 map（用 Firestore transaction 保证一致性）。

---

## 7. reminders（提醒生成记录）

```typescript
{
  id: string,
  studioId: string,
  type: "individual" | "group",
  classId: string,                  // 关联课程
  recipientUserIds: string[],       // 哪些学员
  generatedText: string,            // 生成的文案
  language: "zh" | "en",
  copiedAt?: Timestamp,             // Admin 点了复制按钮的时间
  generatedAt: Timestamp,
  generatedBy: string,
}
```

---

## 索引（必建）

在 Firebase Console > Firestore > Indexes 添加：

1. `classes`: `studioId` ASC + `date` ASC + `startTime` ASC
2. `classes`: `studioId` ASC + `weekKey` ASC + `startTime` ASC
3. `classes`: `studioId` ASC + `coachId` ASC + `date` ASC
4. `classes`: `studioId` ASC + `status` ASC + `startTime` ASC
5. `users`: `studioId` ASC + `role` ASC + `name` ASC
6. `users`: `studioId` ASC + `role` ASC + `needsFollowUp` ASC
7. `creditTransactions`: `userId` ASC + `category` ASC + `createdAt` DESC
8. `bookings` (集合组查询): `userId` ASC + `status` ASC + `bookedAt` DESC

---

## Firestore 安全规则

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // 辅助函数
    function isSignedIn() { return request.auth != null; }
    function userRole() {
      return get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role;
    }
    function userStudio() {
      return get(/databases/$(database)/documents/users/$(request.auth.uid)).data.studioId;
    }
    function isAdmin() { return isSignedIn() && userRole() == "admin"; }
    function isCoach() { return isSignedIn() && userRole() == "coach"; }
    function isStudent() { return isSignedIn() && userRole() == "student"; }
    function isStudioMember(sid) { return isSignedIn() && userStudio() == sid; }

    // studios
    match /studios/{studioId} {
      allow read: if isStudioMember(studioId);
      allow write: if isAdmin() && userStudio() == studioId;
    }

    // users
    match /users/{userId} {
      allow read: if isSignedIn() && (
        request.auth.uid == userId ||
        isAdmin() ||
        (isCoach() && resource.data.studioId == userStudio())
      );
      allow create: if isAdmin();
      allow update: if isAdmin() ||
        (request.auth.uid == userId &&
          // 学员只能改自己的姓名/语言/头像，不能改 role 或余额
          request.resource.data.role == resource.data.role &&
          request.resource.data.creditBalances == resource.data.creditBalances);
      allow delete: if isAdmin();
    }

    // courseTypes
    match /courseTypes/{typeId} {
      allow read: if isStudioMember(resource.data.studioId);
      allow write: if isAdmin();
    }

    // classes
    match /classes/{classId} {
      allow read: if isStudioMember(resource.data.studioId);
      allow create, delete: if isAdmin();
      allow update: if isAdmin() ||
        // 教练只能改 notes 和 attendanceMarked
        (isCoach() && resource.data.coachId == request.auth.uid &&
          request.resource.data.diff(resource.data).affectedKeys()
            .hasOnly(['notes', 'attendanceMarked', 'updatedAt']));

      // bookings 子集合
      match /bookings/{bookingId} {
        allow read: if isStudioMember(get(/databases/$(database)/documents/classes/$(classId)).data.studioId);
        allow create: if isAdmin() ||
          (isStudent() && request.auth.uid == request.resource.data.userId);
        allow update: if isAdmin() ||
          (isCoach() && get(/databases/$(database)/documents/classes/$(classId)).data.coachId == request.auth.uid) ||
          (isStudent() && request.auth.uid == resource.data.userId &&
            request.resource.data.status in ['cancelled']);
        allow delete: if isAdmin();
      }
    }

    // creditPackages
    match /creditPackages/{packageId} {
      allow read: if isSignedIn();
      allow write: if isAdmin();
    }

    // creditTransactions
    match /creditTransactions/{txId} {
      allow read: if isAdmin() ||
        (isSignedIn() && request.auth.uid == resource.data.userId);
      allow write: if isAdmin();
    }

    // reminders
    match /reminders/{reminderId} {
      allow read, write: if isAdmin();
    }
  }
}
```

---

## Storage 规则

```javascript
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {

    // 课程封面图：所有登录用户和持有公开链接的访客都能读，只有 Admin 能写
    match /courseCovers/{studioId}/{fileName} {
      allow read: if true;  // 公开读，因为学员预约页不强制登录
      allow write: if request.auth != null
        && firestore.get(/databases/(default)/documents/users/$(request.auth.uid)).data.role == 'admin'
        && request.resource.size < 5 * 1024 * 1024
        && request.resource.contentType.matches('image/.*');
    }

    // 用户头像
    match /avatars/{userId}/{fileName} {
      allow read: if true;
      allow write: if request.auth != null
        && (request.auth.uid == userId ||
            firestore.get(/databases/(default)/documents/users/$(request.auth.uid)).data.role == 'admin')
        && request.resource.size < 2 * 1024 * 1024
        && request.resource.contentType.matches('image/.*');
    }
  }
}
```
