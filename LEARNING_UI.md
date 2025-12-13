# Vite DevTools（本仓库）UI（Nuxt/Vue）源码学习路线

> 目的：先从 `packages/vite`（UI）切入，按“能跑起来 → 看懂页面与状态 → 追一条典型数据流 → 再逐步接触 Node/RPC”循序渐进。

## 0. 快速跑起来（建议先做到）

- 安装依赖：`pnpm install`
- 生成 Rolldown 数据（UI 依赖）：`pnpm build`
  - 会产出 `packages/vite/.rolldown`（README 有提到）；拉取最新提交后如果 UI 数据不匹配，建议删掉该目录再 `pnpm build` 一次。
- 启动 UI：`pnpm play:ui`（等价于 `pnpm -C packages/vite run dev`）

## 1. UI 入口与骨架（先读这些）

1.1 Nuxt 配置与构建形态

- `packages/vite/src/nuxt.config.ts`
  - `srcDir: 'app'`：真正的 UI 源码入口在 `packages/vite/src/app/`
  - `app.baseURL: '/.devtools-vite/'`：UI 被作为一个静态面板挂载到 devtools 的 host 路径下
  - `modules: ['./modules/rpc']`：UI 内部引入了一个 Nuxt module，把 devtools server 与 RPC functions 注册进来

1.2 App 入口（连接态/加载态）

- `packages/vite/src/app/app.vue`
  - 启动即 `connect()`：建立 RPC 连接
  - 通过 `connectionState` 显示 Error / Loading / `NuxtPage`

1.3 UI 侧 RPC 封装（理解 `useRpc()` 从哪来）

- `packages/vite/src/app/composables/rpc.ts`
  - `connect()`：调用 `getDevToolsRpcClient()`（来自 `@vitejs/devtools-kit/client`）并更新 `connectionState`
  - `useRpc()`：暴露一个 `shallowRef`，页面里用 `rpc.value.$call(...)`

## 2. 路由主链路（建议按顺序读）

> 目标：先看“页面长什么样 + 状态怎么组织”，暂时不追 Node 端实现。

2.1 Session 列表页（入口页）

- `packages/vite/src/app/pages/index.vue`
  - `rpc.value.$call('vite:rolldown:list-sessions')` 拉取 session 列表
  - 选择 session / compare 的交互逻辑（适合新手练习改 UI）

2.2 Session 容器页（最关键：加载 session summary + 控制右侧面板）

- `packages/vite/src/app/pages/session/[session].vue`
  - `onMounted()` 拉取 `vite:rolldown:get-session-summary`，构造 `SessionContext`
  - `useSideNav()` 注册左侧导航
  - 通过 `route.query`（`module/asset/plugin/chunk/package`）控制右侧详情面板展示与关闭（Esc 关闭）

2.3 Session 子页面（挑一个深挖）

- Assets：`packages/vite/src/app/pages/session/[session]/assets.vue`
- Chunks：`packages/vite/src/app/pages/session/[session]/chunks.vue`
- Plugins：`packages/vite/src/app/pages/session/[session]/plugins.vue`
- Packages：`packages/vite/src/app/pages/session/[session]/packages.vue`
- Raw Events：`packages/vite/src/app/pages/session/[session]/raw.vue`
- Graph：`packages/vite/src/app/pages/session/[session]/graph/index.vue`

## 3. UI 状态与可配置项（读完你就能自信改 UI）

- `packages/vite/src/app/state/settings.ts`
  - 用 `useLocalStorage()` 持久化 UI 偏好（视图类型、开关、排序等）
  - 典型用法：页面里读/写 `settings.value.xxx` 来切换 UI

## 4. “不碰 Node”也能完成的第一条练习链路（推荐从这里开始）

选一个页面（例如 Assets）做 3 件事：

4.1 找到它在哪调 RPC

- 例如：`packages/vite/src/app/pages/session/[session]/assets.vue` 中的 `rpc.value.$call('vite:rolldown:get-assets-list', ...)`

4.2 找到它怎么把结果渲染成 UI

- 搜索组件：`packages/vite/src/app/components/`（例如 assets 相关组件在 `components/assets/`）
- 关注：`computed`/`watch`/`useAsyncState` 的数据流

4.3 做一个“可验证的小改动”

- 例子（任选其一）：
  - SearchPanel：增加一个默认筛选/清空按钮
  - 视图切换：把当前 `View as` 的状态在 UI 上更明显（样式/tooltip）
  - 错误态：当 `useAsyncState` 报错时显示更友好的提示

## 5. 常用检索命令（UI 侧非常好用）

- 找所有页面：`rg -n \"pages/\" packages/vite/src/app -S` 或直接看 `packages/vite/src/app/pages/`
- 找所有 RPC 调用点：`rg -n \"\\$call\\(\" packages/vite/src/app -S`
- 找某个 RPC 名称的所有引用：`rg -n \"vite:rolldown:get-assets-list\" packages/vite/src -S`

## 6. 下一步（你准备好再进入 Node/RPC）

当你能在 UI 里熟练定位“页面 → composable → state → component”后，再开始追 RPC 的另一端：

- RPC functions 汇总：`packages/vite/src/node/rpc/index.ts`
- Rolldown 数据读取：`packages/vite/src/node/rolldown/`
- DevTools server 注入到 Nuxt：`packages/vite/src/modules/rpc.ts`
