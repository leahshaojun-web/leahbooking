# PHASE 2 · Admin Dashboard + 课表 + 新增课程

## 前提

Phase 1 已完成。Firebase 已连通。

## 任务

实现 Admin 端三个核心页面：
1. 登录页（Email + Password，仅 Admin/Coach）
2. Admin Dashboard（今日总览）
3. 课表页（日/周视图）+ 新增/编辑课程弹窗

---

## 步骤

### 1. 实现登录页

`src/app/login/page.tsx`：

```tsx
'use client';
import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { signInWithEmailAndPassword } from 'firebase/auth';
import { auth, db } from '@/lib/firebase';
import { doc, getDoc } from 'firebase/firestore';

export default function LoginPage() {
  const router = useRouter();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  async function handleLogin(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);
    setError('');
    try {
      const cred = await signInWithEmailAndPassword(auth, email, password);
      const userDoc = await getDoc(doc(db, 'users', cred.user.uid));
      const role = userDoc.data()?.role;
      if (role === 'admin') router.push('/admin');
      else if (role === 'coach') router.push('/coach');
      else throw new Error('该账号无权访问');
    } catch (e: any) {
      setError(e.message || '登录失败');
    } finally {
      setLoading(false);
    }
  }

  return (
    <main className="min-h-screen flex items-center justify-center bg-bg p-6">
      <div className="w-full max-w-sm">
        <div className="text-center mb-10">
          <div className="inline-flex items-center justify-center w-16 h-16 rounded-2xl bg-gradient-to-br from-green-mid to-green-deep text-white text-3xl font-bold shadow-cta mb-4">L</div>
          <h1 className="text-3xl font-bold text-ink">LeahFit</h1>
          <p className="text-ink-soft text-sm mt-1">工作室管理系统</p>
        </div>

        <form onSubmit={handleLogin} className="space-y-4">
          <input type="email" required placeholder="邮箱"
            value={email} onChange={e=>setEmail(e.target.value)}
            className="w-full px-4 py-3 rounded-xl bg-white border border-ink/10 focus:border-green-brand outline-none" />
          <input type="password" required placeholder="密码"
            value={password} onChange={e=>setPassword(e.target.value)}
            className="w-full px-4 py-3 rounded-xl bg-white border border-ink/10 focus:border-green-brand outline-none" />
          {error && <div className="text-red-dot text-sm">{error}</div>}
          <button type="submit" disabled={loading}
            className="w-full py-3 rounded-pill bg-green-brand text-white font-semibold shadow-cta tappable disabled:opacity-50">
            {loading ? '登录中...' : '登录'}
          </button>
        </form>
      </div>
    </main>
  );
}
```

### 2. Admin 布局（含导航守卫）

`src/app/admin/layout.tsx`：

桌面端左侧 240px 导航 + 内容区；移动端隐藏侧栏，显示底部 tab。

侧栏导航项：
- 🏠 今日总览（/admin）
- 📅 课表排班（/admin/classes）
- 👥 学员管理（/admin/students）
- 💬 微信提醒（/admin/reminders）
- ⚙️ 设置（/admin/settings）

Layout 必须：
- 用 `onAuthStateChanged` 检查登录态
- 未登录跳 `/login`
- 非 admin 角色跳对应页或显示无权限
- 顶部显示 Leah 头像 + 工作室名称

### 3. 创建 ClassCard 组件（核心组件，三端通用）

`src/components/ui/ClassCard.tsx`：

```tsx
import Image from 'next/image';
import { ClassSession, CourseCategory } from '@/types';

const FALLBACK_GRADIENTS: Record<CourseCategory, string> = {
  pilates: 'linear-gradient(135deg, #C8B59A, #7A6E5C)',
  strength: 'linear-gradient(135deg, #4A4038, #1A1310)',
  pt: 'linear-gradient(135deg, #685A4A, #2A1F18)',
  kids: 'linear-gradient(135deg, #E5C088, #B88848)',
  postnatal: 'linear-gradient(135deg, #E5A89E, #B0635A)',
  bungee: 'linear-gradient(135deg, #B0A4C8, #7A6DA0)',
  yoga: 'linear-gradient(135deg, #A8CC92, #5C7A4A)',
};

interface Props {
  classData: ClassSession;
  coachName: string;
  coachAvatarUrl?: string;
  attendees?: { id: string; avatarUrl?: string; name: string }[];
  showAction?: 'book' | 'waitlist' | 'booked' | 'admin' | 'coach' | null;
  onAction?: () => void;
  onClick?: () => void;
}

export function ClassCard({ classData, coachName, coachAvatarUrl, attendees = [], showAction, onAction, onClick }: Props) {
  const startTime = classData.startTime.toDate();
  const endTime = classData.endTime.toDate();
  const formatTime = (d: Date) => `${String(d.getHours()).padStart(2,'0')}:${String(d.getMinutes()).padStart(2,'0')}`;

  const isFull = classData.bookedCount >= classData.capacity;
  const stars = '★'.repeat(classData.difficulty) + '☆'.repeat(5 - classData.difficulty);

  return (
    <div onClick={onClick}
      className="relative h-[200px] rounded-card overflow-hidden shadow-card tappable cursor-pointer">
      {/* 背景图 */}
      <div className="absolute inset-0"
        style={{
          backgroundImage: classData.coverImageUrl
            ? `url(${classData.coverImageUrl})`
            : FALLBACK_GRADIENTS[classData.category],
          backgroundSize: 'cover',
          backgroundPosition: 'center',
        }} />
      {/* 蒙层 */}
      <div className="absolute inset-0 class-card-overlay" />

      {/* 内容 */}
      <div className="relative h-full p-5 flex flex-col justify-between text-white">
        {/* 顶部 */}
        <div className="flex justify-between items-start">
          <div className="flex-1 min-w-0">
            <h3 className="text-xl font-bold leading-tight mb-2 drop-shadow-md">
              {classData.nameZh}
            </h3>
            <div className="text-xs text-white/85">
              难度 <span className="text-yellow-400 tracking-wider">{stars}</span>
            </div>
          </div>
          <div className="text-right ml-4 flex-shrink-0">
            <div className="text-2xl font-num font-semibold drop-shadow-md leading-none">
              {formatTime(startTime)}
            </div>
            <div className="text-[11px] text-white/85 mt-1">
              {formatTime(endTime)} 结束
            </div>
          </div>
        </div>

        {/* 底部 */}
        <div className="flex justify-between items-end">
          <div className="flex items-center gap-2.5">
            <div className="w-11 h-11 rounded-full bg-gradient-to-br from-green-mid to-green-deep flex items-center justify-center text-white font-bold border-2 border-white/40 font-num">
              {coachName[0]}
            </div>
            <div>
              <div className="text-xs font-semibold drop-shadow">{coachName}</div>
              <div className="flex items-center gap-2 mt-1.5">
                {attendees.length > 0 && (
                  <div className="flex">
                    {attendees.slice(0, 3).map((a, i) => (
                      <div key={a.id}
                        style={{ marginLeft: i === 0 ? 0 : -6 }}
                        className="w-6 h-6 rounded-full border-[1.5px] border-white/60 bg-white/30 flex items-center justify-center text-[10px] font-bold font-num">
                        {a.name[0]}
                      </div>
                    ))}
                  </div>
                )}
                <span className="text-xs font-medium font-num">
                  {classData.bookedCount}/{classData.capacity}
                </span>
                {isFull && (
                  <span className="bg-red-dot text-white text-[10px] px-1.5 py-0.5 rounded-pill font-medium">
                    满
                  </span>
                )}
                {!isFull && classData.minToOpen > 1 && classData.bookedCount < classData.minToOpen && (
                  <span className="bg-black/45 text-white/95 text-[11px] px-2 py-0.5 rounded-pill backdrop-blur-sm">
                    满 {classData.minToOpen} 人开课
                  </span>
                )}
              </div>
            </div>
          </div>

          {showAction && (
            <ActionButton type={showAction} onClick={(e) => { e.stopPropagation(); onAction?.(); }} />
          )}
        </div>
      </div>
    </div>
  );
}

function ActionButton({ type, onClick }: { type: string; onClick: (e: React.MouseEvent) => void }) {
  const config: Record<string, { label: string; className: string }> = {
    book: { label: '代约', className: 'bg-green-brand text-white shadow-cta' },
    waitlist: { label: '代排队', className: 'bg-white/95 text-green-deep shadow-md' },
    booked: { label: '已预约', className: 'bg-white/95 text-ink-soft' },
    admin: { label: '详情 →', className: 'bg-white/95 text-ink' },
    coach: { label: '签到 ✓', className: 'bg-green-brand text-white shadow-cta' },
  };
  const c = config[type];
  return (
    <button onClick={onClick}
      className={`px-6 py-2.5 rounded-pill font-semibold text-sm tappable ${c.className}`}>
      {c.label}
    </button>
  );
}
```

### 4. Admin Dashboard（今日总览）

`src/app/admin/page.tsx`：

布局参考你的需求文档，从上到下：

**4 格统计卡**（grid-cols-2 在手机，grid-cols-4 在桌面）：
- 今日课程数（绿色）
- 今日预约人数（紫色）
- 今日未发提醒（黄色）
- 课时即将耗尽（红色）

每张卡片：
- 圆角 22px 白底（带浅色背景的 accent 变体）
- 顶部 emoji 图标
- 大数字（32px Inter Bold）
- 小标签（12px ink-soft）
- 趋势文字（11px，绿色=正常 / 红色=警告）

**查询逻辑**：
```typescript
const today = new Date();
const todayStr = format(today, 'yyyy-MM-dd');
const todayClasses = await getDocs(
  query(collection(db, 'classes'),
    where('studioId', '==', studioId),
    where('date', '==', todayStr),
    orderBy('startTime'))
);
```

**今日课程列表**：用 ClassCard 组件渲染，showAction="admin"。

**底部"续费提醒"区块**：白底圆角卡片，列出 `creditBalances < 2` 的学员。每行：头像 + 姓名 + 课程类型 + 上次预约 + 余额（红色大字）。

### 5. 课表页（/admin/classes）

`src/app/admin/classes/page.tsx`：

**顶部**：
- 标题"课表排班"
- 日/周视图切换（segmented control）
- 左右箭头切换日期 + "今天"按钮
- 右上角 + 按钮新增课程

**周视图**：
- 7 天网格（手机端可横向滚动，每天一列宽 130px）
- 每一列顶部显示日期 + 周几
- 列内按时间顺序堆叠 mini class cards（高度紧凑，仅显示时间 + 课程名 + n/N）
- 今天那一列高亮（绿色顶栏）

**日视图**：
- 单日课程用完整 ClassCard 组件竖向堆叠
- 时间从早到晚

### 6. 新增/编辑课程弹窗

复用一个 `<ClassFormDialog>` 组件，触发场景：
- Dashboard 的 "新增课程" 按钮
- 课表页的 "+" 按钮
- 点击已有课程卡片"编辑"

字段（按你需求文档）：
- 课程类型（下拉，从 courseTypes 加载）
- 日期（date picker）
- 开始时间（time picker，默认根据 courseType.defaultDuration 自动算结束）
- 结束时间
- 教练（下拉，从 users where role=coach）
- 房间（下拉，从 studio.rooms）
- 最大人数（数字，默认从 courseType）
- 最少开课人数
- 难度（1-5 星选择）
- 课程封面图（**关键**：上传到 Firebase Storage，路径 `courseCovers/{studioId}/{classId}.jpg`，留空则用 courseType 的默认图）
- 是否循环（toggle）
  - 频率：每周 / 每两周（仅当循环开启）
  - 结束：N 周后 / 指定日期
- 备注（textarea）

提交时的写入逻辑：
1. 如果是循环课程，生成 N 个 class 文档（每个独立 ID，但 `recurringId` 指向同一个母 ID）
2. 上传图片：先把 class 写入拿 ID，再上传图片到 storage，最后更新 class.coverImageUrl
3. 写入完成后关闭弹窗，触发列表重新查询

### 7. 图片上传组件

`src/components/ui/ImageUploader.tsx`：

```tsx
'use client';
import { useState } from 'react';
import { ref, uploadBytes, getDownloadURL } from 'firebase/storage';
import { storage } from '@/lib/firebase';

interface Props {
  studioId: string;
  classId: string;
  initialUrl?: string;
  onUploaded: (url: string) => void;
}

export function ImageUploader({ studioId, classId, initialUrl, onUploaded }: Props) {
  const [preview, setPreview] = useState(initialUrl || '');
  const [uploading, setUploading] = useState(false);

  async function handleFile(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0];
    if (!file) return;
    if (file.size > 5 * 1024 * 1024) {
      alert('图片不能超过 5MB');
      return;
    }
    setUploading(true);
    try {
      const path = `courseCovers/${studioId}/${classId}-${Date.now()}.${file.name.split('.').pop()}`;
      const r = ref(storage, path);
      await uploadBytes(r, file);
      const url = await getDownloadURL(r);
      setPreview(url);
      onUploaded(url);
    } catch (e: any) {
      alert('上传失败：' + e.message);
    } finally {
      setUploading(false);
    }
  }

  return (
    <div>
      <label className="block">
        <input type="file" accept="image/*" onChange={handleFile} className="hidden" />
        <div className="relative h-40 rounded-card overflow-hidden cursor-pointer bg-ink/5 border-2 border-dashed border-ink/15 hover:border-green-brand">
          {preview ? (
            <img src={preview} alt="" className="w-full h-full object-cover" />
          ) : (
            <div className="absolute inset-0 flex items-center justify-center text-ink-soft text-sm">
              {uploading ? '上传中...' : '点击上传课程封面图（≤5MB）'}
            </div>
          )}
        </div>
      </label>
    </div>
  );
}
```

### 8. 种子数据

为了能立即看到效果，写一个一次性脚本 `scripts/seed.ts`，跑一次创建：
- 1 个 studio：`leahfit-melbourne`
- 1 个 admin user（用你提供的邮箱）
- 7 个 courseTypes（pilates / strength / pt / kids / postnatal / bungee / yoga）
- 5 个示例 class（今天/明天的几节课，分布不同时间和类型）

执行方式：
```bash
npx tsx scripts/seed.ts
```

---

## 完成判定

- 用 Admin 邮箱密码登录后跳到 `/admin`
- Dashboard 显示真实统计数据
- 今日课程列表用 ClassCard 渲染，能看到背景图（上传过的）或 fallback 渐变
- 点击"新增课程"打开弹窗，能上传图片，提交后课程出现在列表
- 周视图和日视图都能正常切换
- 编辑已有课程能预填表单

完成后告诉我："✅ Phase 2 完成"，并提供 Admin 登录测试用的邮箱密码（或确认我提供的）。
