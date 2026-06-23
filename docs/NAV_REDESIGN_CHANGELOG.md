# 培训手册 — 导航重构 Changelog

> 修改文件：[index.html](file:///workspace/index.html) · 2026-06-23

## 核心变更

| 区域 | 旧实现 | 新实现 |
|---|---|---|
| 桌面端导航 | `position:fixed; left:0; width:240px` 永久侧边栏 | sticky 顶部 64px 主菜单 + 8 个分类、每个下拉子菜单面板 |
| 移动端导航 | 整页 `.nav-overlay` 弹出层 + 8 个手风琴组 | 汉堡按钮（**保留沿用**）→ 抽屉式 `.nav-drawer`，与桌面端同源信息架构 |
| 章节高亮 | 滚动时遍历 sidenav 找 active | 滚动回调统一 `setActiveSection(id)`，同步高亮：顶部主菜单 trigger / 子菜单项 / 抽屉子项 |
| 阅读进度 | 无 | 顶部 3px 渐变进度条（红→金）+ 右下角"返回顶部"按钮 |
| 锚点跳转 | 浏览器默认 | 补偿 70px 顶栏高度的 smooth scroll |
| 移动端单页模式 | `body.mobile-page > section` 隐藏兄弟 | 保留兼容（已可工作） |

## 新增 CSS 组件

- `.topnav` / `.topnav.nav-dark` 毛玻璃顶栏
- `.main-menu` + `.menu-trigger` + `.submenu-panel` 主菜单 / 触发器 / 子菜单
- `.topnav-right` + `.topnav-filter` + `.filter-tab` 右侧 tab 过滤器
- `.hamburger` 移动端按钮（含 X 形变换）
- `.nav-drawer` / `.drawer-group` / `.drawer-header` / `.drawer-body` 移动端抽屉
- `.reading-progress` + `.bar` 进度条
- `.back-to-top` 浮动按钮

## 删除 CSS 组件

- `.sidenav` 整块及其变体（`.sidenav.nav-dark`）
- `.nav-filter`（已迁移到 `.topnav-filter`）
- `.cat-label` / `.nav-dot` / `.nav-num`（与新设计无关）
- `.nav-overlay` 整块、`.nav-group` / `.nav-group-header` / `.nav-group-body`

## 新增 JS 函数

- `setActiveSection(id)` — 统一高亮三处位置
- `updateActiveNav()` — scroll 监听（已加 `passive: true`）
- `toggleDrawerGroup(header)` — 抽屉分组单选展开
- `updateProgress()` — 阅读进度 + 浮动按钮显隐
- 顶栏 / 子菜单 / 抽屉的点击行为：移动端用 showPage() 切页，桌面端用 smooth scroll

## 兼容性

- 所有 `<section id="…">` 锚点未动
- 旧的 `setNavFilter('all' | 'director' | 'channel')` 仍可用
- 旧的 `showPage(id)` / `toggleNav()` / `closeNav()` 仍可用
- 已删除对 `toggleNavGroup` / `autoExpandNavGroup` 的引用（如有遗留 onclick，会失效——但全站已清理）
- `< 900px` 触发移动端单页 + 汉堡模式；`>= 900px` 触发桌面端 sticky 顶栏 + dropdown

## 已知遗留

- 旧 `.sidenav a:hover .nav-dot { animation: dotPulse … }` 等孤立 CSS 规则仍存在（不影响渲染，等待下次清理）
- `prefers-reduced-motion` 已加保护，关闭抽屉 / 进度条动画

## 后续迭代（推荐）

1. 全文搜索（fuse.js）
2. 阅读进度 localStorage 记忆
3. 章节底部"下一篇 / 上一篇"快捷按钮
4. 暗色模式手动开关（替代现在仅按时间自动切换）
5. 拆分单文件 → 多 HTML 切片加载（首屏优化）
