# 任务四 · 控制台报错捕获与 ReAct 闭环自愈记录

> 实验者：胡奥（6003226005） · 页面：`starter_code/03_buggy_practice/index.html`
> 环境：Windows 11 + Microsoft Edge（headless + CDP，等价于 F12 DevTools Console）

## 1. 故障现象（Observation 原始证据）

打开演练页，点击「重置摆动」按钮，摆球没有任何复位动作，Console 立即抛出一行红字：

```
Uncaught ReferenceError: resetAnimation is not defined
    at onclick (file:///D:/3/week05_pendulum_sim/starter_code/03_buggy_practice/index.html:34:56)
    at (anonymous) (:1:37)
```

点击「暂停」按钮同样报错：

```
Uncaught ReferenceError: togglePause is not defined
    at onclick (file:///D:/3/week05_pendulum_sim/starter_code/03_buggy_practice/index.html:35:67)
```

完整捕获文本见 [`react-console-error.txt`](./react-console-error.txt)，故障现场截图见
[`screenshots/07_practice_error.png`](./screenshots/07_practice_error.png)。

关键观察：**动画循环一直正常，只有按钮回调失效**。这说明问题不在物理公式或 Canvas 渲染，
而是按钮点击时找不到函数——错误范围被锁定在“事件绑定 / 作用域”这一层。

## 2. ReAct 闭环过程

### Round 1｜Thought → Action → Observation

- **Thought**：先不要重写页面。报错是 `ReferenceError`，说明代码执行到按钮回调时，
  标识符 `resetAnimation` 在查找链上不存在；应确认它到底定义在哪里、以什么形式暴露。
- **Action**：打开 DevTools → Console，点击重置按钮，复制完整红色堆栈；
  再查看 `index.html` 第 34 行与 `<script>` 的函数定义位置。
- **Observation**：
  - 第 34 行是 `<button onclick="resetAnimation()">`；
  - `resetAnimation` 定义在 `<script>` 的 `(function () { ... })();` 内部；
  - 页面其余逻辑（`frame`、`step`、`draw`）都在同一 IIFE 内正常运行。

### Round 2｜Thought → Action → Observation

- **Thought**：行内 `onclick` 的属性值会被浏览器当作**全局作用域**下的代码执行，
  而 `resetAnimation` 是 IIFE 的局部函数，没有挂到 `window` 上。二者不在同一个作用域链，
  所以点击时全局查找失败。这不是拼写错误，也不该靠 `window.resetAnimation = ...` 污染全局来绕过。
- **Action**：检查修复方案的作用域边界：
  1. 删除两个按钮的行内 `onclick`；
  2. 在 IIFE 内部用 `addEventListener` 绑定，让回调与函数处于同一闭包；
  3. 保留 IIFE 封装，不向 `window` 暴露任何调试接口。
- **Observation**：修改点只有按钮标签与 3 行绑定代码，物理与绘制逻辑零改动，符合“外科手术式修复”。

### Round 3｜Thought → Action → Observation

- **Thought**：修复后必须做回归验证，而不是“看起来好了”。验收标准应是：
  Console 0 红字 + 重置精确回到 45° + 暂停/继续恢复正常。
- **Action**：用 Edge headless 重新加载页面，依次点击暂停、重置，并从 Canvas 像素反推摆角。
- **Observation**：
  - 未捕获异常数：**0**
  - 暂停按钮文案变为「继续」，再次点击恢复「暂停」
  - 像素反推摆角：**44.99°**（目标 45°，误差来自抗锯齿采样）
  - 截图：[`screenshots/08_practice_fixed.png`](./screenshots/08_practice_fixed.png)

## 3. 根因定位（Reflection）

| 层次 | 结论 |
| --- | --- |
| 直接原因 | 行内 `onclick` 在全局作用域解析 `resetAnimation`，而函数被 IIFE 封装为局部变量 |
| 深层原因 | 事件绑定方式与代码封装方式不匹配：新增按钮时复制了旧式 `onclick` 写法，但没有同步导出函数 |
| 为什么动画没坏 | 动画由 IIFE 内的 `requestAnimationFrame(frame)` 驱动，不经过全局查找 |
| 为什么不能 `window.resetAnimation = resetAnimation` | 能消除报错，但会把内部实现暴露为全局副作用，且同类问题会继续出现；应统一改用 `addEventListener` |

## 4. 修复 Diff

```diff
- <button id="resetBtn" onclick="resetAnimation()">重置摆动</button>
- <button id="pauseBtn" onclick="togglePause()">暂停</button>
+ <button id="resetBtn" type="button">重置摆动</button>
+ <button id="pauseBtn" type="button">暂停</button>

  // ... IIFE 内部，resetAnimation / togglePause 定义之后
+ document.getElementById('resetBtn').addEventListener('click', resetAnimation);
+ document.getElementById('pauseBtn').addEventListener('click', togglePause);
+
  window.addEventListener('resize', resize);
  resize();
  requestAnimationFrame(frame);
```

对应提交：`fix: resolve reset button runtime error`。

## 5. 反馈质量对照

| 低效反馈（情绪化噪声） | 本次使用的证据驱动反馈 |
| --- | --- |
| “重置按钮坏了，帮我重写一遍” | 粘贴完整堆栈 + 行号 `34:56` + 复现步骤（点击重置） |
| AI 只能盲目重构，容易引入新 Bug | 错误范围被压缩到“事件绑定/作用域”，修复仅 5 行 |
| 无法判断是否真的修好 | 用 0 异常 + 44.99° 像素反推 + 截图做回归验收 |

结论：**报错堆栈不是“给 AI 看的抱怨”，而是可验证的现场证据。**
人负责定义验收标准与判断根因是否成立，AI 负责在证据约束下完成最小修复。
