# 知径桌面版（Windows）构建与分发

> 目标：把 Next.js 15 应用打包成 Windows 可安装的 `.exe`，双击即用；数据库只写用户目录，**绝不写安装目录**。

## 1. 产物

| 产物 | 路径 | 说明 |
| --- | --- | --- |
| NSIS 安装包 | `release/知径-1.0.0-Setup.exe` | 向导式安装，可选择安装目录，创建桌面/开始菜单快捷方式 |
| 免安装目录 | `release/win-unpacked/` | 绿色版，双击 `知径.exe` 直接运行 |
| 服务器运行时 | `dist-server/` | Next standalone 产物（`server.js` + `.next` + `node_modules`），构建中间产物 |

最终实测（2026-10-22 20:16 构建）：

| 项目 | 大小 |
| --- | --- |
| NSIS 安装包 | **93.7 MB（98,215,144 字节）**，SHA256 `BE2DC33B141A5B13510895C4FC1EE8965924CFFA94EA033C4D9EE43EBEC43663` |
| `win-unpacked/` | **358.4 MB / 1,796 文件**（Electron 约 320.9 MB + dist-server 约 37.5 MB） |
| `dist-server/` | **37.5 MB / 1,776 文件**（node_modules 32.0 / `.next` 5.5 / public 2.5 KB；无 `.map`、无 `.d.ts`、无 `data/`） |
| 压缩率 | 358.4 → 93.7 MB（约 0.26，`compression: maximum`） |

> 上述体积/哈希对应 2026-10-22 的最终构建；重新构建后哈希会变化，请以 `release/` 实际文件为准。
> 独立的体积审计（Electron 二进制拆分、locales、裁剪收益、压缩率、构建可靠性）见
> [`docs/SIZE-AUDIT.md`](./SIZE-AUDIT.md)，与本文数字已交叉复核。

## 2. 快速构建

```powershell
# 仓库根目录
node scripts/build-desktop.mjs
```

脚本依次执行：

1. `next build`（`next.config.ts` 中 `ZHIJING_STANDALONE=1` 时才启用 `output: "standalone"`）；
2. 组装 `dist-server/`：`.next/standalone` + `.next/static` + `public`；
3. 清理 `dist-server/data`、`backups`、`*.db`、日志等（**不会把用户数据库打进安装包**）；
4. 检查/安装 `desktop/` 的 Electron 依赖；
5. `electron-builder --win nsis --x64` 输出到 `release/`。

常用参数：

| 参数 | 作用 |
| --- | --- |
| `--skip-build` | 跳过 `next build`，复用现有 `.next` |
| `--skip-install` | 跳过 desktop 依赖安装 |
| `--dir` | 只打 `release/win-unpacked`（快速验证，不生成安装包） |

网络受限时脚本默认走 npmmirror 镜像：

```powershell
$env:ELECTRON_MIRROR = "https://npmmirror.com/mirrors/electron/"
$env:ELECTRON_BUILDER_BINARIES_MIRROR = "https://npmmirror.com/mirrors/electron-builder-binaries/"
```

## 3. 桌面壳架构

```text
desktop/
├── package.json            # 独立 Electron 壳工程（不含 Next 依赖）
├── main.cjs                # 主进程：单实例锁、选端口、spawn 服务器、开窗口、退出清理
├── preload.cjs             # 只暴露只读信息，contextIsolation + sandbox
├── electron-builder.yml    # 打包配置（asar/compression/languages/NSIS）
└── build/
    ├── icon.ico            # 多尺寸图标
    ├── gen-icon.mjs        # 纯 Node 图标生成器
    ├── symlink-fallback.cjs# 本地无符号链接权限时的构建补丁
    ├── verify-node-sqlite.cjs
    └── smoke-dist-server.cjs
```

启动流程：

1. 请求单实例锁；已有实例时聚焦旧窗口，新实例退出；
2. 在 `127.0.0.1` 上动态选一个空闲端口；
3. 用 **Electron 自带 Node**（`ELECTRON_RUN_AS_NODE=1`）spawn
   `resources/dist-server/server.js`，环境变量：
   - `NODE_ENV=production`、`HOSTNAME=127.0.0.1`、`PORT=<空闲端口>`；
   - `ZHIJING_DB_PATH=<userData>/zhijing.db`（关键：数据库只写用户目录）；
4. 轮询 `GET /api/dashboard`，最长 60 秒；就绪后才创建 `BrowserWindow` 加载 `http://127.0.0.1:<port>`；
5. 外部链接交给系统浏览器；应用内路由留在窗口内；
6. 窗口关闭/应用退出时 kill 服务器子进程（Windows 用 `taskkill /T /F` 兜底进程树）。

## 4. 数据与迁移

| 内容 | 位置 |
| --- | --- |
| 数据库 | `%APPDATA%\知径\zhijing.db`（首次启动自动建库，含示例数据） |
| 日志 | `%APPDATA%\知径\logs\desktop.log`（启动失败弹窗会显示该路径） |
| 临时/缓存 | 同上 userData 目录，卸载时不会删除 |

**从旧版（源码/服务器版）迁移学习数据**：退出应用，把旧仓库的 `data/zhijing.db`
复制到 `%APPDATA%\知径\zhijing.db`（覆盖前先退出应用）。也可以使用
`--user-data-dir=<目录>` 或环境变量 `ZHIJING_DB_PATH=<完整 db 路径>` 指定其他位置。

```powershell
$env:ZHIJING_DB_PATH = "D:\我的数据\zhijing.db"
& "$env:LOCALAPPDATA\Programs\知径\知径.exe"
```

> 安装目录（默认 `%LOCALAPPDATA%\Programs\知径`）只放程序文件，**不要**把数据库写进去。

## 5. 体积构成与优化

优化手段（`electron-builder.yml`）：

- `compression: maximum`（NSIS 7z/LZMA 最大压缩）；
- `asar: true`（壳代码单文件）；
- `electronLanguages: [zh-CN, en-US]`（移除 40+ 语言包，安装后省约 47 MB）；
- `files` 只含 `main.cjs` / `preload.cjs` / `package.json`；
- `extraResources` 只含 `dist-server`，并排除 `*.map`、`*.d.ts`、`data/`、`*.db`、日志、缓存；
- 不打包 `node_modules` 开发依赖、`.next/cache`、测试与文档。

体积实测数据见 `docs/SIZE-AUDIT.md`。当前安装包约 93.7 MB；Electron 运行时占绝对大头
（压缩后约 88–89 MB），dist-server 压缩后约 10 MB。`dxcompiler/dxil` 本版保留以保
WebGPU 可靠性；如需再省约 7 MB，可按 `SIZE-AUDIT.md` §6 的 P1-7 做实验包验证。

## 6. 验证方法

构建后可用内置自检模式（无头跑一遍启动链路并输出 JSON）：

```powershell
# 用临时 userData，避免污染真实数据
& "release\win-unpacked\知径.exe" --smoke-test `
    --user-data-dir="$env:TEMP\zhijing-smoke" `
    --smoke-result="$env:TEMP\zhijing-smoke-result.json" `
    --smoke-screenshot="$env:TEMP\zhijing-smoke.png"
echo $LASTEXITCODE   # 0 = 通过
```

自检内容：窗口创建、页面加载、`document.title`/正文长度、`/api/dashboard` HTTP 200、
数据库文件在 userData 下生成、截图。结果同时写入 `%APPDATA%\知径\logs\desktop.log`。

手动验证清单：

1. `release\win-unpacked\知径.exe` 双击能打开仪表盘；
2. 新建目标、上传 PDF/DOCX、AI 对话、生成笔记/复习卡；
3. 关闭窗口后任务管理器无残留 `知径.exe`/Node 子进程；
4. 安装版：`Setup.exe /S /D=<临时目录>` 静默安装 → 运行安装目录内 `知径.exe --smoke-test`
   （退出码 0、dashboard 200、userData 生成 db）→ 卸载后目录清理。

> **重要验证陷阱**：必须在仓库外运行 `--smoke-test`（例如把 `win-unpacked` 整个拷到
> `%TEMP%`），否则 Node 会向上解析到仓库 `node_modules`，造成假通过。
>
> 若当前 shell 设置了 `ELECTRON_RUN_AS_NODE=1`，Electron 会以纯 Node 模式启动，
> 报 `bad option: --smoke-test`。测试前先 `Remove-Item Env:ELECTRON_RUN_AS_NODE`。

## 7. 常见问题

| 现象 | 原因/处理 |
| --- | --- |
| 双击安装包提示“未知发布者” | 安装包未做代码签名，Windows SmartScreen 正常提示；点“更多信息 → 仍要运行” |
| 首次启动黑屏 1–2 秒 | 正在建库/启动 Next 服务器；`ready-to-show` 后显示窗口 |
| 启动失败弹窗 | 查看 `%APPDATA%\知径\logs\desktop.log`，通常是端口/权限/杀软拦截 |
| 数据为空 | 桌面版使用 `%APPDATA%\知径\zhijing.db`；按 §4 迁移旧库 |
| 安装包体积大 | Electron 运行时约 320 MB 原始；安装包已用 LZMA 压到约 93.7 MB |
| 想改端口 | 桌面版自动选空闲端口；Web 版用 `pnpm start`（固定 127.0.0.1:3000） |

## 8. 构建可靠性

Next 的 standalone file trace 在 Windows/pnpm 环境偶发漏文件（本机两次构建分别漏
`styled-jsx/package.json`、`next/dist/lib/metadata/get-metadata-route.js`）。
`scripts/build-desktop.mjs` 已内置四步闸门：

1. 组装后真实启动 `dist-server/server.js`，轮询 `/api/dashboard` 必须 200；
2. 失败时按 `MODULE_NOT_FOUND` 从隔离构建目录补齐缺失模块；
3. 仍失败则整体替换为完整 `next` 包再 prune；
4. 裁剪后再次启动复检；未通过绝不进入 electron-builder。

同时显式断言 `next/dist/lib/metadata/get-metadata-route.js` 存在，并对 `dist-server`
做“无 data/、无 *.db”检查。**不要绕过这套闸门，也不要打包后手删 resources/dist-server**。
