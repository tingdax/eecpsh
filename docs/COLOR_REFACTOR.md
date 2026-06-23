# 配色重构 — 循环系统语义色 + 深色模式

> 修改文件：[index.html](file:///workspace/index.html) · 2026-06-23

## 语义色（Light 默认）

| 名称 | Token | 主色 | 用途 |
|---|---|---|---|
| 动脉红 (Artery) | `--c-artery` | `#C41E3A` | 品牌主色、核心 CTA、警示、强调 |
| 静脉蓝 (Vein) | `--c-vein` | `#1A5C6E` | 信息、次级操作、链接、章节副色 |
| 神经绿 (Nerve) | `--c-nerve` | `#2E8B57` | 成功、增长、验证通过、推荐标识 |
| 淋巴黄 (Lymph) | `--c-lymph` | `#D4A843` | 强调、高亮、数据徽章、premium 感 |

每个色都提供 4 档：`*-` / `*-soft`（hover） / `*-dark`（pressed） / `*-bg`（背景 8~16% 透明）

## 深色模式（Dark）

四色提亮 12~20% 亮度，保证在 `#0F141A` 深底上的对比度：

| 名称 | Light | Dark |
|---|---|---|
| 动脉红 | `#C41E3A` | `#FF4D6D` |
| 静脉蓝 | `#1A5C6E` | `#4FB3C8` |
| 神经绿 | `#2E8B57` | `#5DDB96` |
| 淋巴黄 | `#D4A843` | `#FFD45E` |

表面层级：
- `--paper` `#FFF8F5` → `#0F141A`（主背景）
- `--paper-elevated` `#FFFFFF` → `#1A2026`（卡片、表格）
- `--paper-sunken` `#F5EBE0` → `#0A0E12`（强调区段、cover、cta、footer）

文字层级：
- `--ink` `#1A1A1A` → `#F5F0E8`
- `--ink2/3/4` 实色 → `rgba(245,240,232,0.85/0.65/0.40)`

## 触发方式

- **自动模式（默认）**：跟随系统 `prefers-color-scheme: dark` + 时间（早 6 点前 / 晚 19 点后）
- **手动模式**：顶栏右侧"☀ / ☾"按钮一键切换；选择持久化到 `localStorage('omay-theme')`
- 手动选择后，自动模式不再覆盖（避免定时器与用户偏好打架）
- 每 15 分钟检查一次时间变化（仅在 auto 时生效）

## 顶栏新增元素

- 主题切换按钮：圆形 40×40，浅色时显示月亮 ☾ 边框静脉蓝；深色时显示太阳 ☀ 边框淋巴黄；hover 微缩放

## 兼容性

- 旧变量名（`--red` / `--blue` / `--gold` / `--gold-bright` / `--green` / `--yellow`）保留为 `var(--c-*)` 别名
- 现有全部 CSS 不需要改动即可跟随新色板
- 暗色模式对以下区块做了显式覆盖（避免硬编码渐变在暗底上消失）：
  - `.section` / `.section-white` / `.section-dark` / `.section-red` / `.section-gold` / `.section-gradient`
  - `.dept-section-dark` / `.dept-section-purple`
  - `.card` / `.toc` / `.data-table` / `table` / `.footer` / `.cta-section` / `.cover`
  - `.sh-section` / `.sh-users-card` / `.sh-bar` / `.sh-user-card` / `.sh-note`
  - `.qr-modal-box` / `.lb-overlay` / `.mobile-page-bar`
  - `h2/h3/h4`、`.section-lead`、`.section-num`、`.dept-name`、`.stage-num`、`.highlight-num`、`.data-source`
  - `a` / `a:hover`

## 后续优化（建议）

1. 把 4 色直接应用到内容语义中（疗效数字 → 神经绿、警示 → 动脉红、推荐级别 → 淋巴黄）
2. 当前自动模式仅依赖时间和系统偏好；可加「时间地点日出日落」算法
3. 增加 `color-scheme: light dark` 声明，让浏览器原生滚动条 / 表单元素跟随
4. 阅读主题（纸质 / 羊皮纸）作为第三种 mode
