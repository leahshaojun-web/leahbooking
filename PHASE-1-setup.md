# PHASE 1 · 项目初始化 + Firebase 集成

## 任务

搭建 Next.js 14 项目脚手架，配置 Firebase（Firestore + Auth + Storage），建立基础布局和 design tokens。这一 phase 不实现任何业务功能，只确认技术骨架能跑通。

## 步骤

### 1. 创建 Next.js 项目

```bash
npx create-next-app@14 leahfit-scheduling --typescript --tailwind --app --src-dir --import-alias "@/*" --no-eslint
cd leahfit-scheduling
```

### 2. 安装依赖

```bash
npm install firebase@10
npm install date-fns date-fns-tz
npm install lucide-react
```

### 3. 字体加载

修改 `src/app/layout.tsx`，加载 Noto Sans SC + Inter（**通过 next/font，不要用 link 标签**）：

```tsx
import { Inter, Noto_Sans_SC } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  weight: ['400','500','600','700'],
  variable: '--font-inter',
});

const notoSC = Noto_Sans_SC({
  subsets: ['latin'],
  weight: ['400','500','600','700'],
  variable: '--font-noto-sc',
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="zh-CN" className={`${inter.variable} ${notoSC.variable}`}>
      <body>{children}</body>
    </html>
  );
}
```

### 4. Tailwind 配置

完整替换 `tailwind.config.ts`：

```typescript
import type { Config } from 'tailwindcss';

const config: Config = {
  content: ['./src/**/*.{js,ts,jsx,tsx,mdx}'],
  theme: {
    extend: {
      colors: {
        yellow: { brand: '#F5C842', soft: '#FCE9A8', deep: '#D9A823' },
        green: { brand: '#5DB87C', mid: '#8BC9A1', soft: '#D4EAD9', deep: '#3D8859' },
        red: { dot: '#E8615E' },
        ink: { DEFAULT: '#1F2418', soft: '#5C5C52', faint: '#9C9C90' },
        bg: { DEFAULT: '#F2F0EA' },
      },
      fontFamily: {
        sans: ['var(--font-noto-sc)', 'var(--font-inter)', 'sans-serif'],
        num: ['var(--font-inter)', 'sans-serif'],
      },
      borderRadius: {
        card: '18px',
        pill: '50px',
      },
      boxShadow: {
        card: '0 4px 16px rgba(0,0,0,0.08)',
        cta: '0 4px 12px rgba(93,184,124,0.35)',
        modal: '0 30px 60px -20px rgba(0,0,0,0.25), 0 8px 24px rgba(0,0,0,0.1)',
      },
      animation: {
        'fade-up': 'fadeUp 0.4s ease backwards',
        'wave': 'wave 2.4s ease-in-out infinite',
      },
      keyframes: {
        fadeUp: {
          '0%': { opacity: '0', transform: 'translateY(8px)' },
          '100%': { opacity: '1', transform: 'translateY(0)' },
        },
        wave: {
          '0%,60%,100%': { transform: 'rotate(0)' },
          '10%,30%': { transform: 'rotate(14deg)' },
          '20%': { transform: 'rotate(-8deg)' },
          '40%': { transform: 'rotate(-4deg)' },
          '50%': { transform: 'rotate(10deg)' },
        },
      },
    },
  },
  plugins: [],
};
export default config;
```

### 5. 全局 CSS

完整替换 `src/app/globals.css`：

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  --type-pilates: linear-gradient(135deg, #C8B59A, #7A6E5C);
  --type-strength: linear-gradient(135deg, #4A4038, #1A1310);
  --type-pt: linear-gradient(135deg, #685A4A, #2A1F18);
  --type-kids: linear-gradient(135deg, #E5C088, #B88848);
  --type-postnatal: linear-gradient(135deg, #E5A89E, #B0635A);
  --type-bungee: linear-gradient(135deg, #B0A4C8, #7A6DA0);
  --type-yoga: linear-gradient(135deg, #A8CC92, #5C7A4A);
}

html, body {
  background: #F2F0EA;
  color: #1F2418;
  -webkit-font-smoothing: antialiased;
  -webkit-tap-highlight-color: transparent;
  overscroll-behavior-y: contain;
}

* { -webkit-tap-highlight-color: transparent; }

.tappable:active { transform: scale(0.98); transition: transform 0.15s ease; }

/* 课程卡片背景蒙层 */
.class-card-overlay {
  background: linear-gradient(135deg, rgba(0,0,0,0.15) 0%, rgba(0,0,0,0.55) 100%);
}

/* 隐藏滚动条 */
.no-scrollbar::-webkit-scrollbar { display: none; }
.no-scrollbar { scrollbar-width: none; }
```

### 6. Firebase 配置

创建 `src/lib/firebase.ts`：

```typescript
import { initializeApp, getApps, getApp } from 'firebase/app';
import { getFirestore } from 'firebase/firestore';
import { getAuth } from 'firebase/auth';
import { getStorage } from 'firebase/storage';

const firebaseConfig = {
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
  storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: process.env.NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID,
  appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID,
};

export const app = getApps().length ? getApp() : initializeApp(firebaseConfig);
export const db = getFirestore(app);
export const auth = getAuth(app);
export const storage = getStorage(app);
```

### 7. 环境变量

在 Replit 的 Secrets 面板（不是 .env 文件）添加：

```
NEXT_PUBLIC_FIREBASE_API_KEY=
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=
NEXT_PUBLIC_FIREBASE_PROJECT_ID=
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=
NEXT_PUBLIC_FIREBASE_APP_ID=
```

值我会从 Firebase Console 提供给你。如果还没有，先用占位符让代码能编译。

### 8. 核心 TypeScript 类型

创建 `src/types/index.ts`：

```typescript
import { Timestamp } from 'firebase/firestore';

export type UserRole = 'admin' | 'coach' | 'student';
export type CourseCategory = 'pilates' | 'strength' | 'pt' | 'kids' | 'postnatal' | 'bungee' | 'yoga';
export type ClassStatus = 'scheduled' | 'confirmed' | 'cancelled' | 'completed';
export type BookingStatus = 'booked' | 'waitlist' | 'attended' | 'absent' | 'leave_taken' | 'cancelled';

export interface User {
  id: string;
  role: UserRole;
  studioId: string;
  name: string;
  nameEn?: string;
  wechatName?: string;
  phone?: string;
  email?: string;
  avatarUrl?: string;
  language: 'zh' | 'en';
  membershipStatus?: 'active' | 'paused' | 'inactive';
  notes?: string;
  needsFollowUp?: boolean;
  joinedAt?: Timestamp;
  specialty?: string[];
  bio?: string;
  creditBalances?: Record<string, number>;
  createdAt: Timestamp;
  updatedAt: Timestamp;
}

export interface CourseType {
  id: string;
  studioId: string;
  nameZh: string;
  nameEn: string;
  category: CourseCategory;
  defaultDuration: number;
  defaultCapacity: number;
  minToOpen: number;
  difficulty: 1 | 2 | 3 | 4 | 5;
  defaultRoom: string;
  coverImageUrl: string;
  creditCost: number;
  description?: string;
  isActive: boolean;
  sortOrder: number;
}

export interface ClassSession {
  id: string;
  studioId: string;
  courseTypeId: string;
  coachId: string;
  startTime: Timestamp;
  endTime: Timestamp;
  date: string;
  weekKey: string;
  nameZh: string;
  nameEn: string;
  category: CourseCategory;
  difficulty: number;
  capacity: number;
  minToOpen: number;
  room: string;
  coverImageUrl: string;
  status: ClassStatus;
  bookedCount: number;
  waitlistCount: number;
  attendanceMarked: boolean;
  recurringId?: string;
  isRecurring: boolean;
  notes?: string;
  reminderSent: boolean;
  reminderSentAt?: Timestamp;
  createdAt: Timestamp;
  updatedAt: Timestamp;
  createdBy: string;
}

export interface Booking {
  id: string;
  userId: string;
  userName: string;
  userPhone?: string;
  status: BookingStatus;
  bookedAt: Timestamp;
  cancelledAt?: Timestamp;
  attendanceMarkedAt?: Timestamp;
  attendanceMarkedBy?: string;
  creditDeducted: boolean;
  creditTxId?: string;
  notes?: string;
}
```

### 9. 共享 UI 组件骨架

创建 `src/components/ui/` 目录，建以下文件（仅骨架，后续 phase 填充）：

- `ClassCard.tsx`：大图背景课程卡（Phase 2 完整实现）
- `Button.tsx`：胶囊按钮
- `BottomTabBar.tsx`：底部 5 tab 导航
- `DatePills.tsx`：横向日期选择器
- `Avatar.tsx`：圆形头像（带 fallback 字母）

每个文件先建一个空 component 占位即可，导出默认函数返回 `<div>{name}</div>`。

### 10. 路由结构

在 `src/app/` 下建立目录：

```
src/app/
├── page.tsx                          · 首页（暂时显示 "LeahFit OS"）
├── login/
│   └── page.tsx                      · 登录页（Phase 2 实现）
├── admin/
│   ├── layout.tsx                    · Admin 布局（左侧导航 + 内容）
│   ├── page.tsx                      · Admin Dashboard
│   ├── classes/page.tsx
│   ├── students/page.tsx
│   ├── reminders/page.tsx
│   └── settings/page.tsx
├── coach/
│   ├── layout.tsx                    · Coach 布局（移动端单栏）
│   └── page.tsx                      · 教练今日页
└── book/
    └── [studioSlug]/
        ├── page.tsx                  · 学员预约首页
        └── my/page.tsx               · 学员个人中心
```

每个 `page.tsx` 先返回最简内容：

```tsx
export default function Page() {
  return <div className="p-8">Page placeholder</div>;
}
```

### 11. 测试连接

在 `src/app/page.tsx` 写一个最小测试：

```tsx
'use client';
import { useEffect, useState } from 'react';
import { db } from '@/lib/firebase';
import { collection, getDocs, limit, query } from 'firebase/firestore';

export default function Home() {
  const [status, setStatus] = useState('checking...');

  useEffect(() => {
    (async () => {
      try {
        await getDocs(query(collection(db, 'studios'), limit(1)));
        setStatus('✅ Firebase connected');
      } catch (e: any) {
        setStatus('❌ ' + e.message);
      }
    })();
  }, []);

  return (
    <main className="min-h-screen flex items-center justify-center bg-bg">
      <div className="text-center">
        <h1 className="text-4xl font-bold text-ink mb-4">LeahFit Scheduling OS</h1>
        <p className="text-ink-soft">{status}</p>
      </div>
    </main>
  );
}
```

---

## 完成判定

- `npm run dev` 能正常启动，访问 `localhost:3000` 显示 "LeahFit Scheduling OS"
- Firebase Secrets 配置后页面显示 "✅ Firebase connected"
- 所有目录和文件创建完毕
- TypeScript 编译无报错

完成后告诉我："✅ Phase 1 完成"，并列出所有创建的文件路径。
