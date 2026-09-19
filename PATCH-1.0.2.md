# 1.0.2 补丁说明

在现有 Electron + TypeScript 项目上修改，共用原有状态机和动画播放器，无新增运行时依赖。按最后确认保留普通／最上层／最底层选项，并继续保存层级。每次启动普通散步、仓鼠球均关闭，散步方式恢复左右。

## 修改文件

- `electron/main.ts`：缩放保持中心、菜单固定、拒绝旧位置消息、缩放存档、仓鼠球资源列表。
- `electron/preload.ts`、`src/bridge.d.ts`：缩放状态和位置版本通信。
- `src/menu.ts`、`src/index.html`、`src/styles.css`：稳定滑杆事件、像素菜单及仓鼠球开关。
- `src/renderer.ts`、`src/animationManager.ts`：单播放器继续当前帧、分别读取两套资源。
- `src/animations.ts`：三种球动画的帧数、尺寸和速度配置。
- `src/stateMachine.ts`、`src/freeWalk.ts`：独立球行为、完整动画边界、休息、拖动、睡眠隔离和边界约束。
- `scripts/split-ball.py`、`assets/hamster_ball/`：固定网格切帧脚本、原图及三张输出图。
- `tests/state.test.cjs`、`tests/electron-smoke.cjs`、`tests/ball-resize-smoke.cjs`：逻辑及真实窗口验证。
- `package.json`、`package-lock.json`、`README.md`、本文：版本、切帧命令、打包排除原始合集、使用说明。

## 素材与切帧

原图放在 `assets/hamster_ball/hamster_ball_atlas.png`。现有原图为 1132×916 RGBA，按照用户确认，仅在处理时向底部补 4px 透明画布到 920px；原文件保持不变。按 x=16、y=32/348/664、每格 128×128、每行最多 8 格，依次抽取 10/10/12 帧。没有物体识别、透明边界裁剪、重新绘画或改变帧顺序。

切帧需要 Python 3 + Pillow，仅用于重新生成素材，应用运行无需 Python。已有生成结果可以直接启动。首次准备与重新切帧：

```sh
python3 -m venv .venv
.venv/bin/python -m pip install Pillow
.venv/bin/python scripts/split-ball.py
# 若系统 python3 已安装 Pillow，也可以：
npm run cut:ball
```

Windows 虚拟环境对应 `.venv\Scripts\python.exe`。默认补 4px；新的完整图可传 `--pad-bottom 0`。输出：

| 文件 | 帧数 | 尺寸 |
| --- | --- | --- |
| `assets/hamster_ball/move_right.png` | 10 | 1280×128 |
| `assets/hamster_ball/move_left.png` | 10 | 1280×128 |
| `assets/hamster_ball/idle_breathe.png` | 12 | 1536×128 |

三张图均保留 RGBA 透明背景和完整 128×128 帧画布。所有帧四边均经过透明度检查，球体没有接触裁切边。读取与写出均输出帧数、尺寸、文件存在及透明度结果。越界或可见像素接触边缘会报出动作、帧号、图片尺寸，提示检查 startX/startY/frameWidth/frameHeight；脚本不会猜新坐标。固定坐标在脚本 `GROUPS`、`FRAME_WIDTH`、`FRAME_HEIGHT` 中修改。运行时也严格检查球图尺寸，错误会在控制台及错误提示中显示。应用只加载三个输出文件，原合集不进入应用包。

## 启动与验证

```sh
npm install
npm start
npm test
```

1. 普通形态右键打开菜单，连续快速往返拖动大小滑杆。菜单应一直打开且固定，仓鼠中心稳定（屏幕边缘会正常约束），呼吸继续，不能触发震惊或窗口拖拽。球形态重复一次。关闭重开应用应保留最终大小。
2. 开启仓鼠球模式，应在当前动画结束后切入球形态；窗口位置及大小不变。左右／全屏控制球的移动范围，普通散步开关不影响球自动移动。手动关闭球模式后恢复此前普通散步选择。
3. 关闭菜单，不点击或拖动，等待超过 3 分钟。球仍应滚动与休息，不出现普通睡眠动画。切回普通形态后，无互动 3 分钟仍会睡觉。
4. 球移动时按住拖拽、改变左右方向并释放，窗口应跟随且只使用球动画，转向不重播第一帧。正在呼吸时，窗口立即跟随，呼吸播完再切行走；释放后短暂停顿并恢复自动移动。普通形态仍保留上一版立即用行走打断动作、释放续播的逻辑。
5. 球形态左键停止位移并呼吸；右键在当前移动循环结束后开菜单，菜单期间不移动。选择吃瓜子／洗脸会退出球形态，动作后安静待机；选择睡觉会退出球形态并正常入睡。
6. 关闭再启动，恢复位置、大小、层级，球模式和普通散步关闭，散步方式左右。

真实窗口自动验证需 Playwright，可用 `PLAYWRIGHT_PATH` 指向现有安装，不是应用运行依赖：

```sh
node tests/electron-smoke.cjs
node tests/ball-resize-smoke.cjs
HAMSTER_LONG_TEST=1 node tests/ball-resize-smoke.cjs
```

长测试实际观察 215 秒，验证超过 180 秒仍有位移。测试使用独立临时存档。逻辑测试覆盖 29 项，包含模拟 240 秒的球模式行为。

## 调整速度

`src/freeWalk.ts` 的 `ballSpeed.min/max` 默认为 45/85 px/s，同文件控制距离及休息概率。`src/animations.ts` 独立控制帧时长：球行走 80ms、呼吸 150ms；不要通过改变帧时长调整位移速度。所有球帧保持 128×128，显示大小使用与普通形态共用的 scale，固定绘制留白使两种素材的视觉大小接近。

## 交付

`release/v1.0.2/` 提供 Mac Apple Silicon 应用、ZIP 及 Windows x64 免安装 EXE。请先从旧版菜单退出，再打开新版本，避免单实例机制继续显示旧版。Mac 未签名公证；Windows 为交叉打包，尚未在 Windows 实机验证。
