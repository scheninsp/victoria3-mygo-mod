# mygo 1.13.9 调试记录

最后更新：2026-10-12

## 目标与边界

当前排查的是 Victoria 3 1.13.9 中“义务批量交易”运行到阶段 6 后无法继续的问题。用户自行启动和验证游戏；不要由调试者启动游戏或操作游戏进程。按用户要求只查 mod、游戏随附脚本和 GUI 中实际使用的接口，不检查或逆向游戏可执行文件。本记录用于跨对话保留状态。

## 用户已报告的运行现象

- 用户没有以 `debug_mode` 启动游戏；`debug.log` 中没有 mod 添加的 `debug_log` 输出是符合当前测试条件的，不能据此推断相关脚本没有执行。
- 界面能显示 mod 的运行状态以及原版“添加资金转让条款”窗口，说明 GUI 覆盖和打开窗口链路已执行到阶段 6。
- 当前一例界面数据：`phase=6 qty=100 min=100 max=144 AI-accept=-2`。国家和条约上下文显示正确，条款 source/right、target/left 也正确；原版“添加”按钮可点击，游戏仍暂停且没有卡死。
- 用户把原版金额滑杆拖到最右端后，批量进度仍没有变化。尚未拿到拖动后诊断行中的 `qty=max` 状态，因此要区分“拖动未改变 ArticleDraft.GetQuantity”和“数量已达上限但自动添加状态没有触发”。

## 1.13 原版文件确认的接口

本机 `game/gui/right_click_menu.gui` 中，原版资金输入控件位于 `article_input_select_money`：

- 滑杆绑定 `value = "[ArticleDraft.GetQuantity]"`，上下限分别绑定 `GetMinValidQuantity` 和 `GetMaxValidQuantity`。
- 原版以 `onvaluechanged = "[ArticleDraft.OnQuantityChangedSlider]"` 绑定滑杆变化事件；原版 GUI 中没有显式调用 `OnQuantityChangedSlider(value)` 的例子。
- 原版金额编辑框绑定 `OnInputTextChanged`，在 `oneditingfinished` 调用 `OnQuantityChanged`。
- 原版添加按钮以 `TreatyDraft.CanIncludeArticleDraft(ArticleDraft.Self)` 作为 `enabled`，点击时调用 `TreatyDraft.AddArticleDraft(ArticleDraft.Self)`。

因此，`CanIncludeArticleDraft` 是添加按钮当前是否可用的原版判断；AI 接受度不是这个按钮的前置条件。`AI-accept=-2` 是显示信息，当前 mod 阶段 6 的自动添加条件没有读取它。

本机 `game/common/treaty_articles/05_transfer_money.txt` 中，资金转让的有效最小值为 100，普通上限按 `fixed_income / 4` 计算（另有相对支出达到阈值时上限 100 的规则）。界面中的 min 100、max 144 与这套规则相符。

## 当前 mod 阶段链与故障点

主要文件：

- `gui/vob_widgets.gui`：阶段 4 时调用 `ToggleArticleTypeInput(..., 'vob_money_input')` 打开输入窗口，并清理本轮诊断变量。
- `gui/right_click_menu.gui`：自定义 `vob_money_input` 继承原版输入模板；阶段 5 状态尝试设置金额，再调到阶段 6；阶段 6 自动添加状态要求数量等于当前上限且 `CanIncludeArticleDraft` 为真，随后调用 `AddArticleDraft`、进到阶段 7并关闭菜单。
- `common/scripted_guis/vob_guis.txt`：阶段变量转换及跳过/失败处理。

目前工作树的阶段 5 设置写法是 `ArticleDraft.OnQuantityChangedSlider(IntToFixedPoint(ArticleDraft.GetMaxValidQuantity))`。`cb` 诊断变量只证明阶段 5 的隐藏 widget `on_start` 已进入；它不证明 `ArticleDraft.GetQuantity` 被改写。当前诊断行还显示数量是否等于上限、`CanIncludeArticleDraft`、是否记录了本轮上限、条约上下文与 source/target 匹配。

先前备份中的写法是 `ArticleDraft.SetQuantity(IntToFixedPoint(ArticleDraft.GetMaxValidQuantity))`。可见原版 GUI 没有直接使用这个方法；先前运行观察到金额仍为 100，因此不能把它当作已验证的 1.13 可用写入接口。对 `OnQuantityChangedSlider(...)` 的直接调用同样不是原版 GUI 展示的调用形式；现有静态证据只确认它作为滑杆 `onvaluechanged` 回调暴露。

基于已报告的 `qty=100/max=144`，目前最直接的阻断是阶段 6 的 `qty=max` 前置条件为假，因而 `AddArticleDraft` 状态不会进入。接下来需要用同一时刻的完整 UI 诊断行确认 `gate qty=max`。如果它为“是”且 `include` 也为“是”但阶段仍停 6，再查阶段状态转换和 `AddArticleDraft` 的 GUI 调用时机；若 `gate qty=max` 为“否”，继续只围绕 1.13 原生滑杆/编辑框回调验证数量写入，不应查二进制。

## 下一轮验证要点

加载当前 mod 文件后，在资金转让窗口暂停于阶段 6 时，记录完整诊断行，尤其是：

- `qty`、`min`、`max`；
- `gate qty=max`；
- `include`；
- `cb` 和 `max-same`；
- context 和 article 行。

先记录滑杆未手动拖动时的状态；再把滑杆拖到最右端并记录同一组值。若 `qty` 变成 144 但 `gate qty=max` 仍为“否”，比较 quantity 的格式/类型判断；若 gate 与 include 均为“是”但状态不前进，阻断点才转为阶段 6 状态调度或 `AddArticleDraft` 调用。

## 当前未验证事项

- 当前 `OnQuantityChangedSlider` 的显式参数调用能否在 1.13 修改条款数据：尚未确认；只有用户重新加载当前文件后的 UI 反馈才能确认。
- `AddArticleDraft` 的原版按钮接口形式在本机 1.13 GUI 中可见，但隐藏 widget 的 `on_start` 调用是否在当前运行窗口上下文中同样工作，仍需运行结果验证。
- 当前 UI 的 `debug_log` 只能在相应调试日志配置下作为额外证据；不应让用户为了这次排查切换启动方式。屏幕上的 VOBTRACE 更适合作为本次直接反馈通道。

## 2026-10-12 后续修正：阶段 6 自动流程被提前终止

用户反馈重新进入资金转让窗口后诊断文字消失。复核发现这是当前调试补丁造成的，不是 debug mode 的日志开关问题：

- `gui/right_click_menu.gui` 中 `vob_native_quantity_failed` 在阶段 6、数量尚未等于上限时立即清除 `vob_armed`，调用 `vob_quantity_failed` 把阶段重置为 0 并清空队列。金额自动回调没有改动数量时，这个条件会截断工作流；之后即使手动拖到最大值，添加状态也因要求 `vob_armed` 而无法继续。
- 同一文件的诊断面板原先用 `vob_armed` 作为 `visible` 条件，因此上面的失败分支清除变量后，仍打开的资金窗口会同时隐藏 VOBTRACE。
- 已删除这个阶段 6 的“数量失败即退出”隐藏 widget，令阶段 6 保持等待；诊断面板显示条件改为 `ArticleDraft.HasType('money_transfer')`，不再依赖运行开关。新增 `armed`、`ArticleDraft.IsPossible`、`ArticleDraft.HasAllRequiredInputsSet` 项，这些与自动添加状态要求一致。进度仍停滞的情况下，应能继续观察状态，且手动调整金额后自动添加触发器仍有效。

## 当前测试环境需注意的重复 mod

用户最新启动产生的 `logs/debug.log` 显示同一次运行同时挂载了本地 `C:\Users\infin\Documents\Paradox Interactive\Victoria 3\mod\mygo`（支持版本 `1.13.*`）和 Steam Workshop `D:\SteamLibrary\steamapps\workshop\content\529340\3806246202`（支持版本 `1.14.*`）。两份 mod 的名称和 ID 相同，且都定义 VOB GUI；并行启用会使文件覆盖/类型定义来源不明确。验证本地改动时应在启动器只启用本地 `mygo`，停用 Workshop 重复项。该结论来自日志中的两条 `Mounted Data` 和两条版本匹配记录；没有启动或操作游戏。

当前补丁只做了文本/结构验证，未启动游戏。下一次由用户测试时，记录面板五行（尤其 `armed`、`qty=max`、`possible`、`inputs`、`include`）并确认滑杆拖到上限后是否自动添加并进入阶段 7。
