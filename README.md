# 小仓鼠桌面宠物

Electron + TypeScript + 原生 HTML/CSS，Mac / Windows 共用一份代码。不使用 React、联网字体或服务器。普通素材从桌面「仓鼠计划/previews」原样复制；仓鼠球按指定固定网格切帧。

## 安装与启动

需要 Node.js 22 或更高版本。在本目录打开终端：

```sh
npm install
npm start
```

如果依赖安装工具阻止了 Electron 的安装脚本，执行一次 `node node_modules/electron/install.js`，然后重新启动。第一次安装需要下载 Electron。

## 使用

- 左键：震惊；连续快速点击只触发一次。
- 按住拖动：清醒时跟随鼠标；睡着时保持原位，松开后先醒来，再慢慢跑到释放位置。
- 右键：打开粉色像素菜单。待机时立即打开，其他动作结束后打开；睡眠先唤醒。
- 菜单提供吃瓜子、洗脸、自由散步、散步方式、仓鼠球、窗口层级、大小、睡觉及退出确认。窗口层级依次切换普通／最上层／最底层，并保存选择。
- 普通模式 3 分钟无互动自动睡觉；仓鼠球模式持续移动，不自动睡觉。自由散步和鼠标悬停不算互动。
- 大小默认 2×（128×128），可调整至 1×～4×。退出保存位置及大小。每次启动散步关闭、模式为左右。
- 键盘辅助：仓鼠获得焦点后 Enter/空格互动，Shift+F10 打开菜单；菜单 Tab 切换、Esc 关闭。

普通形态拖拽具有最高动画优先级：清醒时立即暂停当前动作并切换行走，松开后从暂停位置续播。拖动转向不重置帧进度。睡眠循环中仍保持原位，释放后先醒来，再以 110px/s 跑向释放位置。选择菜单选项后菜单保持打开，点击外部、关闭按钮或 Esc 关闭；菜单打开时暂停自动移动，关闭后恢复。仓鼠限制在显示器工作区内，留 28px 边距；拔掉显示器后自动约束到现有屏幕。

## 打包

```sh
# Mac：Apple Silicon 和 Intel，分别输出 DMG 和 ZIP
npm run pack:mac

# Windows：x64 安装程序及免安装 EXE，建议在 Windows 上运行
npm run pack:win

# 只生成本机应用目录
npm run pack:local
```

输出在 `release/`。Mac 本机应用位于 `release/mac-arm64/小仓鼠.app`（Intel 为 `mac/`）；Windows 位于 `release/win-unpacked/`。未配置开发者签名或 Apple 公证；正式分发时需自行提供证书。此项目没有自动上传或发布操作。

项目包含 `.github/workflows/build.yml`，把本目录作为 GitHub 仓库根目录后，可以在 Actions 手动构建两个平台并下载产物。

## 动画素材

把文件放进 `assets/hamster/`，名称固定为：

```text
idle_breathe_1   idle_breathe_2   sleep_enter   sleep_loop   wake_up
eat              bathe            move_left     move_right    shocked
```

同名文件按 `.webp` → `.gif` → `.png` 优先读取。WebP/GIF 使用 Chromium 的 ImageDecoder 解码全部帧，再由 canvas 按配置速度播放，不使用浏览器默认动画计时。PNG 作为横向、纵向或多行图集，默认每格 64×64、按行排列；更大单帧请修改 `src/animations.ts` 的 `assetConfig.frameWidth/frameHeight`。每个 PNG 的帧数由该动作配置决定，多余尾格忽略并警告。

现有 10 个 WebP 均为 **256×256**，帧数与需求一致；画面仍按逻辑 64px × scale 显示。程序会明确输出尺寸差异警告，原素材不变。动画帧数不同会警告并按实际全部帧播放；缺素材、解码失败或图集无法切帧会弹出明确错误，不会静默空白。

## 修改位置

| 内容 | 文件 |
| --- | --- |
| 帧数、速度、循环说明、资源目录和后缀 | `src/animations.ts` |
| 状态机、队列、3 分钟睡眠、唤醒、拖拽、点击冷却 | `src/stateMachine.ts` |
| 距离分层、随机目标、休息分层、屏幕边距 | `src/freeWalk.ts` |
| 素材解码与 canvas 播放 | `src/animationManager.ts` |
| 菜单按钮及滑杆 | `src/menu.ts` |
| 菜单布局、颜色、像素边框 | `src/index.html`、`src/styles.css` |
| 窗口、显示器、位置与大小存档、退出 | `electron/main.ts` |

状态机是动画唯一控制者，`tick()` 分别推进移动距离与动画周期，普通操作入队，右键菜单在安全动画边界优先处理。菜单打开时允许待机和主动选择的动作，自动移动暂停；睡眠循环直到互动才退出。散步在休息与完整移动循环之间切换，连续 2～4 次移动后强制普通或长休息。

存档是 Electron 用户数据目录内的 `position.json`，保存 x、y、scale、layer，使用临时文件原子替换；写入失败会提示且不直接退出。

## 验证

```sh
npm test
```

已在当前 Mac 上通过 30 项 Node 内置测试，无测试框架，覆盖动作不可打断、点击去重、菜单等待、睡眠隔离、睡眠拖拽、散步睡眠计时、多屏边界等。

`tests/electron-smoke.cjs` 另提供真实 Electron 窗口验证，需要已安装的 Playwright（可通过 `PLAYWRIGHT_PATH` 指定模块路径），不作为普通启动依赖。测试使用独立临时存档，不改用户的实际位置。

技术参考：[Electron BrowserWindow](https://www.electronjs.org/docs/latest/api/browser-window)、[Electron screen](https://www.electronjs.org/docs/latest/api/screen)、[ImageDecoder](https://developer.mozilla.org/en-US/docs/Web/API/ImageDecoder)。

已完成开发版与打包 Mac 应用的真实窗口验证：逐帧显示、菜单尺寸、动作等待、睡眠右键唤醒、真实睡眠拖拽、缩放、退出确认、存档恢复和重启散步默认值。Windows 产物为交叉打包，尚未在 Windows 实机运行。

## 1.0.1 更新

修复开启散步后菜单仍打开导致一直暂停的问题；开启时自动收起菜单并开始位移。拖拽即时暂停动作并播放行走，释放后续播。睡醒追赶速度调整至 110px/s。新增普通、最上层和最底层选项。

最底层指普通应用窗口下面、桌面背景上面。Mac 使用原生窗口层级；Windows 使用系统自带 PowerShell 调用 SetWindowPos 保持底层（约 300ms 校正一次），没有新增运行库。切换模式及退出会停止辅助进程。Windows 层级效果仍需实机验证。

## 1.0.2 更新

修复大小滑杆抖动，新增仓鼠球独立动画及移动模式，保留上一版普通／最上层／最底层选项及其保存行为。完整素材说明、修改清单和验收步骤见 [PATCH-1.0.2.md](PATCH-1.0.2.md)。本次产物单独放在 `release/v1.0.2/`，不会覆盖旧版。

## 1.0.3 更新

仓鼠球由画布的 75% 放大至 100%（整体约增大 33%），使球内仓鼠本体接近普通形态的大小；不改素材、scale 或窗口尺寸，完整保留球体。菜单选择后保持打开，可连续切换形态、散步、层级和大小；吃瓜子、洗脸、睡觉仍正常执行。点击外部会关闭菜单，取消退出确认则返回菜单。菜单打开期间不会自动移动或自动睡觉。

本次修改 `src/animations.ts`、`src/stateMachine.ts`、`electron/main.ts` 及现有测试、版本号和说明。输出在 `release/v1.0.3/`；先退出旧版，再启动新版。Mac 已实测，Windows 仅交叉打包。
