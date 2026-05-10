# LeahFit Scheduling OS · Replit 实施包

## 工作流

每个 `PHASE-*.md` 文件都是一个**完整自包含的 prompt**，可以直接复制粘贴给 Replit AI。
按顺序执行：先 Phase 1 跑通，再 Phase 2，依此类推。

不要跳着做。每个 phase 都依赖前面的产物。

---

## 文件清单

| 文件 | 内容 | 阶段 |
|---|---|---|
| `00-design-tokens.md` | 颜色/字体/间距/动效规范（粘贴到第一个 prompt 里） | 参考 |
| `01-data-schema.md` | Firestore 数据结构 + 安全规则 | 参考 |
| `PHASE-1-setup.md` | Firebase 项目初始化 + Next.js 脚手架 | 第 1 步 |
| `PHASE-2-admin-core.md` | Admin 端：Dashboard + 课表 + 新增课程 | 第 2 步 |
| `PHASE-3-students.md` | Admin 端：学员管理 + 课时充值 | 第 3 步 |
| `PHASE-4-class-detail.md` | 课程详情页 + 出席标记 + 微信提醒生成 | 第 4 步 |
| `PHASE-5-coach-view.md` | 教练端（受限权限） | 第 5 步 |
| `PHASE-6-student-booking.md` | 学员预约链接（公开页面） | 第 6 步 |
| `PHASE-7-deploy.md` | Firebase Hosting 部署 | 第 7 步 |

---

## 关于 Production URL

部署完成后，你的应用会有两个 URL：

- **Admin/Coach 内部使用**：`https://leahfit-scheduling.web.app/admin`
- **学员预约链接**：`https://leahfit-scheduling.web.app/book/{studio-slug}` （公开，可以发给学员）

实际域名以 Phase 1 创建 Firebase 项目时为准，Replit 跑完 Phase 7 会告诉你具体 URL。

---

## 给 Replit AI 的统一指令前缀

每次粘贴 PHASE 文件之前，先告诉 Replit：

> 这是一个分阶段的项目，我会一次给你一个 phase 的指令。请严格按照指令实施，不要自作主张添加未要求的功能或页面。完成后告诉我："✅ Phase X 完成"，并列出新建/修改了哪些文件。我确认后才继续下一 phase。

---

## 技术栈（不要改）

- **框架**：Next.js 14 App Router (TypeScript)
- **样式**：Tailwind CSS + 自定义 design tokens
- **数据库**：Firebase Firestore（compat SDK，沿用你之前项目的方式）
- **存储**：Firebase Storage（已升级 Blaze 计划）
- **认证**：Firebase Auth（仅 Admin/Coach 需要登录；学员通过 magic link 或手机号）
- **部署**：Firebase Hosting

不要使用：localStorage（不可靠）、其他 CSS 框架、其他后端、其他认证方案。
