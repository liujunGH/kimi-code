# 消息列表窗口化（virtualization）设计与实施路线

> 分支：`feature/virtual-list`。目标：长会话（数百 turn）下 DOM 布局/绘制成本与历史长度解耦，滚动与流式不再随历史变长而变慢。

## 背景

- 现状：`ChatPane.vue` 用 `v-for` 全量渲染所有 turn（无窗口化）；turn 高度动态（思考流、工具卡、markdown、图片）。
- 已完成的优化（kimi-ui 分支）：工具组默认折叠、折叠即卸载、输出尾部截断、turn 引用和解、scrollKey/observer 减负、历史 600 条封顶。剩下的成本主要是"全量 turn 的布局与绘制"。
- 约束：不能破坏现有跟随状态机（scrollToBottom / 锚点恢复 / pinScroll / 历史向上分页恢复位置）。

## 路线

### 阶段 1：CSS 窗口化（本分支先做）

给 turn 容器加 `content-visibility: auto` + `contain-intrinsic-size: auto <estimate>`：

- 浏览器对离屏 turn 跳过布局与绘制，只保留占位高度；`auto` 变体在渲染过一次后记住真实高度，锚点计算基本不受影响。
- 改动面极小（纯 CSS），结构零变化，滚动跟随/锚定逻辑不动。
- 预期：长会话滚动与流式重排的 GPU/布局成本大幅下降；DOM 节点仍在（内存降幅有限，但节点本身远小于布局产物）。

验证：长会话滚动手感、WebContent footprint、锚点恢复是否漂移（向上翻历史再回来）。

### 阶段 2：结构级真窗口化（视阶段 1 效果决定）

间隔渲染 + 占位条：只渲染视口 ± overscan 的 turn，顶部/底部用实测累计高度做 spacer。

- 需要每 turn 高度缓存（turnId → height，render 后记录）；
- 跟随/锚点逻辑改造：scrollToBottom = 滚动到尾部窗口；锚点恢复改为"逻辑位置"而非 DOM top；
- 风险点：动态高度测量、流式期间窗口内 turn 高度变化、prepend 时的位置恢复。

## 阶段 1 实施点

- `ChatPane.vue` / `ConversationPane.vue` 的 turn 容器（`.a-msg`、`.u-turn` 及 turn 包装元素）：
  - `content-visibility: auto;`
  - `contain-intrinsic-size: auto 220px;`（平均 turn 高度的估计值，仅影响首次渲染前的占位）
- 注意：`pinScrollFor` 的 `getBoundingClientRect().top` 在 content-visibility 元素上仍可工作（有占位/实测高度），但折叠交互的 pin 需要实测验证。

## 验证清单

- [ ] 长会话（本仓库开发会话 300+ 消息）滚动流畅度
- [ ] WebContent footprint 对比（基线 ~380–400MB）
- [ ] 向上翻历史再回到底部：锚点位置不漂移
- [ ] 折叠/展开工具组时 pin 不跳
- [ ] 流式期间底部跟随正常
