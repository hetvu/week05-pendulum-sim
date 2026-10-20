# 自动化验收记录（Edge headless + CDP）

> 验收对象：`index.html`（单文件应用）
> 视口：1440 × 900，devicePixelRatio = 1
> 方式：Microsoft Edge `--headless=new` 启动，通过 Chrome DevTools Protocol 监听
> `Runtime.exceptionThrown` / `Runtime.consoleAPICalled` / `Log.entryAdded`，
> 并用 `Input.dispatchMouseEvent` 发送真实指针事件。

## 1. 主应用功能验收

| # | 验收项 | 操作 / 断言 | 结果 |
| --- | --- | --- | --- |
| 1 | 首屏渲染 | 打开 `index.html`，Canvas 有发光摆球与网格 | ✅ 通过 |
| 2 | 单摆运行 | 摆角持续变化，FPS 稳定 | ✅ 60 FPS |
| 3 | 滑块联动 | L=330、g=21.5、k=0.006，标签与周期同步 | ✅ 通过（T=2.46 s） |
| 4 | 重置精度 | 点击重置，HUD 立即显示 45.0° | ✅ 45.0° |
| 5 | 暂停 / 继续 | 暂停 500 ms 内摆角不变，按钮文案切换 | ✅ 通过 |
| 6 | 摆球拖拽 | 真实鼠标按下摆球并拖动，状态变 DRAGGING，释放后恢复 | ✅ 通过 |
| 7 | 双摆 RK4 | 切换双摆，连续运行 2 s 以上无数值发散 | ✅ θ₁/θ₂ 与能量正常 |
| 8 | 双摆拖拽 | 拖动紫色摆球，状态 DRAGGING，释放后继续演化 | ✅ 通过 |
| 9 | 天体预设 | 点击月球，重力平滑过渡到 1.62 并联动滑块 | ✅ 通过 |
| 10 | 本地持久化 | L=280、g=15.5、k=0.002、双摆模式，刷新后完整恢复 | ✅ 通过 |
| 11 | 控制台 | 上述全流程异常 / console.error / Log error 数量 | ✅ **0** |

持久化缓存实际内容：

```json
{"v":1,"mode":"double","L":280,"g":15.5,"k":0.002,"preset":"custom"}
```

## 2. ReAct 故障演练页回归

| 验收项 | 操作 | 结果 |
| --- | --- | --- |
| 修复前复现 | 点击重置按钮 | ❌ `ReferenceError: resetAnimation is not defined`（index.html:34:56） |
| 修复后异常数 | 暂停 → 重置 | ✅ 0 |
| 重置角度 | 像素反推（白心摆球质心） | ✅ **44.99°**（目标 45°） |
| 暂停按钮 | 点击一次 | ✅ 文案变为「继续」 |

故障证据：[`react-console-error.txt`](./react-console-error.txt)
修复分析：[`react-debug.md`](./react-debug.md)

## 3. 复现方式

1. 双击打开 `index.html`；
2. 按 F12 打开 Console，确认无红色报错；
3. 拖动三个滑块，摆球实时变长 / 变快 / 衰减；
4. 点击「重置摆动」回到 45°；
5. 点击顶栏「双摆」，观察 RK4 混沌与彩虹拖尾；
6. 点击「月球 / 木星 / 太空」，观察重力平滑过渡；
7. 修改参数后刷新页面，确认 localStorage 恢复上次配置。

## 4. 原始报告

`docs/verification-report.json` 为自动化脚本的原始输出，包含每一步的
模式、参数、读数、FPS 与异常列表。
