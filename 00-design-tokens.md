# Design Tokens · LeahFit Scheduling OS

这份规范贯穿三端（Admin / Coach / Student），所有颜色、间距、字号都从这里取。

## 颜色系统

```css
:root {
  /* === 主色 · Yellow Brand Header === */
  --yellow: #F5C842;
  --yellow-soft: #FCE9A8;
  --yellow-deep: #D9A823;

  /* === 操作色 · Green CTA === */
  --green: #5DB87C;
  --green-mid: #8BC9A1;
  --green-soft: #D4EAD9;
  --green-deep: #3D8859;

  /* === 状态色 === */
  --red-dot: #E8615E;        /* 已满 */
  --amber-warn: #D9A823;     /* 候补/警告 */

  /* === 中性色 === */
  --bg: #F2F0EA;             /* 页面底色 */
  --surface: #FFFFFF;        /* 卡片底色 */
  --ink: #1F2418;            /* 主文字 */
  --ink-soft: #5C5C52;       /* 辅文字 */
  --ink-faint: #9C9C90;      /* 弱文字/占位 */
  --rule: rgba(31,36,24,0.08);

  /* === 课程类型色（用于无图片时的 fallback 渐变） === */
  --type-pilates: linear-gradient(135deg, #C8B59A, #7A6E5C);   /* 普拉提 暖米色 */
  --type-strength: linear-gradient(135deg, #4A4038, #1A1310);  /* 力量 深灰 */
  --type-pt: linear-gradient(135deg, #685A4A, #2A1F18);        /* 私教 棕褐 */
  --type-kids: linear-gradient(135deg, #E5C088, #B88848);      /* 儿童 琥珀 */
  --type-postnatal: linear-gradient(135deg, #E5A89E, #B0635A); /* 产后 玫瑰 */
  --type-bungee: linear-gradient(135deg, #B0A4C8, #7A6DA0);    /* 飞绳 薰衣草 */
  --type-yoga: linear-gradient(135deg, #A8CC92, #5C7A4A);      /* 瑜伽 柔绿 */
}
```

## 字体

```html
<link href="https://fonts.googleapis.com/css2?family=Noto+Sans+SC:wght@400;500;600;700&family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
```

```css
body {
  font-family: 'Noto Sans SC', 'Inter', -apple-system, sans-serif;
  font-size: 15px;
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
}

/* 时间数字、容量数字、百分比用 Inter（更紧凑） */
.font-num { font-family: 'Inter', sans-serif; }
```

## 圆角

| 元素 | 值 |
|---|---|
| 大卡片（课程卡） | `18px` |
| 中卡片 | `16px` |
| 小卡片 | `12px` |
| 按钮（胶囊） | `50px` |
| 按钮（标准） | `14px` |
| 头像 | `50%` |
| 输入框 | `12px` |

## 间距

8px 网格系统：4 / 8 / 12 / 16 / 20 / 24 / 32 / 40 / 56

## 阴影

```css
--shadow-card: 0 4px 16px rgba(0,0,0,0.08);
--shadow-cta: 0 4px 12px rgba(93,184,124,0.35); /* 绿色按钮专用 */
--shadow-modal: 0 30px 60px -20px rgba(0,0,0,0.25), 0 8px 24px rgba(0,0,0,0.1);
```

## 课程卡片规范（核心组件）

每节课都是一张 200px 高的图片背景卡片，结构如下：

```
┌────────────────────────────────────────┐
│ [背景图 + 半透明深色蒙层]              │
│                                        │
│  课程名称              09:00          │
│  难度 ★★★★☆        10:00 结束        │
│                                        │
│                                        │
│  [教练头]              ┌──────────┐   │
│  LeahFit               │  代约    │   │
│  👥👥 2/3 满2人开课    └──────────┘   │
└────────────────────────────────────────┘
```

**蒙层**：`linear-gradient(135deg, rgba(0,0,0,0.15) 0%, rgba(0,0,0,0.55) 100%)`
图片越亮蒙层加深一点确保白字可读。

**容量标签**：
- 已满：红色圆点 `#E8615E` + 白字 "满"
- 候补：灰色胶囊 + "满 N 人开课" 文字
- 已预约：绿色对勾 + "已预约"

**操作按钮**：
- 可预约：实色绿色胶囊「代约」/「预约」
- 已满：白底绿字「代排队」
- 已预约：白底灰字「已预约」+ 二次点击展开取消选项

## 动效

```css
/* 卡片点击反馈 */
.tappable:active { transform: scale(0.98); transition: transform 0.15s ease; }

/* 进入动画（错峰） */
@keyframes fadeUp {
  from { opacity: 0; transform: translateY(8px); }
  to { opacity: 1; transform: translateY(0); }
}
.class-card { animation: fadeUp 0.4s ease backwards; }
.class-card:nth-child(1) { animation-delay: 0.05s; }
.class-card:nth-child(2) { animation-delay: 0.10s; }
.class-card:nth-child(3) { animation-delay: 0.15s; }

/* 挥手 */
@keyframes wave {
  0%,60%,100% { transform: rotate(0); }
  10%,30% { transform: rotate(14deg); }
  20% { transform: rotate(-8deg); }
  40% { transform: rotate(-4deg); }
  50% { transform: rotate(10deg); }
}
.wave { display: inline-block; animation: wave 2.4s ease-in-out infinite; transform-origin: 70% 70%; }
```

## 响应式断点

```css
/* Mobile first */
/* < 600px : phone */
/* 600-900px : tablet (iPad mini, iPad) */
/* > 900px : desktop (Admin only, 双栏布局) */
```

学员端和教练端**永远单栏移动端布局**（即使桌面也居中显示 max-width: 430px）。
Admin 端在桌面端展开为双栏（左侧导航 240px + 右侧内容）。

## 图标

使用 emoji（你截图里就是这种风格）：

| 用途 | emoji |
|---|---|
| 普拉提 | 🧘‍♀️ |
| 力量 | 💪 |
| 儿童 | 🤸 |
| 产后 | 🌸 |
| 飞绳 | 🪁 |
| 瑜伽 | 🧘 |
| 时间 | 🕘 / 🕙 / 🕛 |
| 主页 | 🏠 |
| 课表 | 📅 |
| 学员 | 👥 |
| 提醒 | 💬 |
| 设置 | ⚙️ |
| 警告 | ⚠️ |
| 复制 | 📋 |
| 出席 | ✓ |
