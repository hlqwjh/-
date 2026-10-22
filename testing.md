# 测试与验收指南

本文说明如何在本地运行知径的类型检查、API 烟测与浏览器端到端（E2E）测试，以及如何更新界面截图、定位失败原因。

## 1. 前置条件

| 项目 | 要求 |
| --- | --- |
| Node.js | ≥ 22.13（推荐 24 LTS；项目使用内置 `node:sqlite`，22.5–22.12 需加 `NODE_OPTIONS=--experimental-sqlite`） |
| pnpm | ≥ 9 |
| 浏览器 | Playwright Chromium（`pnpm exec playwright install chromium`） |
| 服务 | 测试不负责启动服务，需先运行 `pnpm dev`（默认 `http://127.0.0.1:3000`） |

> **不要同时运行 `pnpm dev` 与 `pnpm build`**：两者共用 `.next` 目录，会互相覆盖。需要验证生产构建时先停掉 dev server。
> E2E 配置刻意不启用 Playwright `webServer`，就是为了避免测试运行时自动启动/重建 `.next`。

## 2. 命令速查

```bash
# 安装依赖与浏览器
pnpm install
pnpm exec playwright install chromium

# 启动服务（另一个终端）
pnpm dev

# 类型检查
pnpm typecheck          # 等价于 pnpm exec tsc --noEmit

# API 烟测（默认 http://127.0.0.1:3000）
pnpm smoke              # 等价于 node scripts/smoke.mjs
node scripts/smoke.mjs http://127.0.0.1:3001   # 指定地址

# 浏览器 E2E
pnpm test:e2e           # 等价于 playwright test
pnpm test:e2e:ui        # Playwright UI 模式
BASE_URL=http://127.0.0.1:3001 pnpm test:e2e   # 指定地址

# 只跑某个文件 / 某个用例
pnpm exec playwright test tests/review.spec.ts
pnpm exec playwright test -g "复习中心"
```

## 3. API 烟测（`scripts/smoke.mjs`）

覆盖主链路接口，只依赖运行中的服务：

- `GET /api/dashboard`、`GET /api/goals`、`GET /api/goals/[id]`
- `POST /api/goals/analyze`
- `POST /api/goals/[id]/chat`（SSE 事件完整性：meta / delta / done）与 `GET` 历史
- `POST /api/goals/[id]/quiz` 与 `POST .../quiz/grade`
- `POST /api/goals/[id]/flashcards/generate`、`GET /api/flashcards/due`、`POST /api/flashcards/[id]/review`
- `POST /api/goals/[id]/notes`
- `GET /api/settings`、`POST /api/settings/test`

演示模式（未配置 API Key）下全部用例应通过；配置真实模型后走真实调用，耗时取决于服务商。

## 4. E2E 用例矩阵（`tests/`）

共 12 个用例、7 个文件（`wizard.spec.ts` 含 3 个用例），串行执行。

| 文件 | 覆盖内容 | 依赖 / 说明 |
| --- | --- | --- |
| `dashboard.spec.ts` | 品牌、主标题、seed 目标卡片、统计/痛点/动态区块、复习入口；截 `dashboard.png` | CRUD API + seed 数据 |
| `goal-detail.spec.ts` | 从仪表盘点击进入详情、头部信息、五个 tab 切换（知识地图/资料/痛点/笔记）；截 `goal-detail.png`、`goal-knowledge.png` | 同上 |
| `wizard.spec.ts` | 向导步骤 1→2→3、痛点选择与自定义输入、AI 分析结果（知识地图/学习路径/诊断问题）、完整创建链路（创建→跳转目标详情→清理测试目标）；截 `wizard.png`、`wizard-analysis.png` | AI 分析接口；不可用时会跳过结果断言并记录 `wizard-analysis-error.png` |
| `review.spec.ts` | 到期卡片翻面、四档评分可用、评分后到期数减少；截 `review.png` | 无到期卡片时自动调用生成接口补卡 |
| `settings.spec.ts` | Provider 预设（DeepSeek/OpenAI/Ollama/自定义）、连接参数表单、演示模式提示、测试连接；截 `settings.png` | 不保存任何配置；演示模式下测试连接走内置 mock |
| `tutor.spec.ts` | 导师页框架、六种模式、历史消息、发送一条消息并等待流式回复结束；截 `tutor.png`、`tutor-chat.png` | SSE 对话接口；不可用时会跳过并记录 `tutor-error.png` |
| `responsive.spec.ts` | 390×844 移动端视口：无横向溢出、抽屉导航开合、目标详情/导师/设置可用；截 `mobile-*.png` | 布局检查不依赖具体数据 |

通用断言：

- 每个用例都会监听 `pageerror` 与 `console.error`，出现即失败。仅忽略 favicon 404、React DevTools 提示等与产品无关的噪声。
- 桌面用例额外断言页面无横向溢出（`document.scrollWidth ≤ 视口宽度`）。
- 定位优先使用 role / label / 可见文案，不使用易碎的结构选择器（抽屉遮罩等无 role 的结构除外）。

## 5. 截图

- 规格：桌面 1440×900 视口，`fullPage: true`；移动端 390×844。
- 输出目录：`docs/screenshots/`（每次执行测试会覆盖同名文件）。
- 截图为 `page.screenshot({ path, fullPage: true })`。
- 截图辅助函数（`tests/helpers.ts` 的 `capture`）会：等待字体就绪、等待 toast 消失（Toast 4.2s 自动关闭）、隐藏 Next.js 开发指示器 `nextjs-portal`、滚回顶部后再截图。隐藏开发指示器只影响截图，不影响测试断言。
- 当前截图清单：

```
docs/screenshots/
├─ dashboard.png          仪表盘（桌面）
├─ goal-detail.png        目标详情·概览
├─ goal-knowledge.png     目标详情·知识地图
├─ wizard.png             新建目标向导·步骤 2
├─ wizard-analysis.png    新建目标向导·AI 分析结果
├─ review.png             复习中心·卡片正面
├─ settings.png           模型与设置
├─ tutor.png              AI 导师·历史对话
├─ tutor-chat.png         AI 导师·流式回复完成
├─ mobile-dashboard.png   390px 仪表盘
├─ mobile-goal-detail.png 390px 目标详情
├─ mobile-tutor.png       390px AI 导师
└─ mobile-settings.png    390px 设置
```

## 6. 结果与失败排查

运行产物：

- `playwright-report/index.html`：HTML 报告，`pnpm exec playwright show-report` 打开。
- `test-results/`：失败用例截图、`error-context.md`（可访问性快照）、`trace.zip`（仅在重试时记录，`pnpm exec playwright show-trace <trace.zip>` 查看）。

失败分类建议：

1. **产品 bug**：页面/接口行为与预期不符。保留截图 + trace + 复现步骤，报告给负责人，不要为通过而改应用代码。
2. **等待中**：页面热更新、接口尚未实现导致 404/500。稍等重跑；本仓库开发期多人并行，dev server 会即时重编译。
3. **测试脆弱**：定位器歧义或时序问题。优先修测试（改用 role/name 精确定位、`expect.poll` 等待异步状态），不要削弱对产品行为的断言。

## 7. 测试对演示数据的影响

- 烟测与 E2E 都会写入本地演示数据库 `data/zhijing.db`：烟测会生成测验、复习卡、笔记与对话；E2E 会复习 1 张卡片并向导师发送消息。
- 需要恢复初始演示数据：设置页「清空并恢复演示数据」，或 `POST /api/settings/reset`。
- `review.spec.ts` 在到期卡片为 0 时会调用 `POST /api/goals/[id]/flashcards/generate` 补卡，因此可以重复执行。
- 截图反映的是运行时的真实数据状态；如果刚跑过烟测/其他测试，仪表盘统计会包含这些记录。
