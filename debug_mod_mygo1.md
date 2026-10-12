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

## 2026-10-12：停用重复 Workshop mod

发现 active playset `myLauncher` 同时挂载本地 `mygo` 和 Steam Workshop `3806246202`。配置位于：

`C:\Users\infin\Documents\Paradox Interactive\Victoria 3\launcher-v2.sqlite`

两条记录分别是：

- 本地 `mygo`：`source=local`，支持版本 `1.13.*`，`enabled=1`，位置 0。
- Workshop `3806246202`：`source=steam`，路径 `D:\SteamLibrary\steamapps\workshop\content\529340\3806246202`，支持版本 `1.14.*`，位置 1。

已将 `playsets_mods.enabled` 对应 Workshop 条目改为 `0`，没有删除 `mods` 表记录，也没有删除 Steam Workshop 文件。数据库备份为：

`C:\Users\infin\Documents\Paradox Interactive\Victoria 3\launcher-v2.sqlite.backup_20261012_232429`

修改后查询确认：本地 `mygo` 为启用，Workshop `3806246202` 为停用。重新启动 Launcher/游戏后应只挂载本地 mod；若需恢复，将该条目的 `enabled` 改回 `1`，或在 Launcher 的 playset 中重新启用。

## 2026-10-12：修复诊断文本并拆分阶段 6 条件

`debug.log` 中的 `PdxDataFetchLocalizedData failed` 来自 `right_click_menu.gui` 的诊断行：一条 `raw_text` 同时包含普通文字和多个 `[SelectLocalization(...)]`。1.13 将整行当成一个本地化数据键，因而把表达式字符串继续送入本地化查询并报错；这不是阶段 6 条件本身的结果。

## 2026-10-13：确认 1.13 金额接口与阶段 6 修正

本机 1.13 原版 `game/gui/right_click_menu.gui` 的 `article_input_select_money` 只有以下金额变更路径：

- 滑杆 `value` 读取 `ArticleDraft.GetQuantity`，用户改变滑杆时触发无显式参数的 `ArticleDraft.OnQuantityChangedSlider`。
- 编辑框使用 `ArticleDraft.OnInputTextChanged`，编辑结束时调用 `ArticleDraft.OnQuantityChanged`。

原版 GUI 没有 `ArticleDraft.SetQuantity`，也没有 `ArticleDraft.OnQuantityChangedSlider(value)` 的调用示例。因此 mod 之前的：

`ArticleDraft.OnQuantityChangedSlider(IntToFixedPoint(ArticleDraft.GetMaxValidQuantity))`

不能证明或实现数量写入；实际运行中的 `qty=100 max=144` 证实它没有改变 ArticleDraft。`OnQuantityChangedSlider` 是控件事件回调，不是已确认可用的带参数 setter。

已移除这次无效的自动设值调用，并取消阶段 6 的 `qty=max` 前置条件。阶段 6 现在只要求原版 `TreatyDraft.CanIncludeArticleDraft(ArticleDraft.Self)` 为真，然后调用原版 `TreatyDraft.AddArticleDraft(ArticleDraft.Self)`。因此当前合法的 100 金额应能继续批量流程；金额最大化仍需通过真实的滑杆/编辑框 UI 事件实现，不能依赖未确认的脚本接口。

已将诊断面板改为每个状态一个 `raw_text` 表达式，并在英文本地化和简体中文本地化中分别提供 Y/N 文本。面板现在逐项显示：`armed`、暂停、运行环境、条约上下文、左方玩家、右方目标、非重谈、资金转让类型、来源、目标、阶段 6、数量是否为最大值、能否加入、最大值是否不小于最小值、数量回调、条款可行性和必需输入是否齐全。

用户重新启动并进入资金转让窗口后，应先确认日志不再刷上述本地化错误，再根据面板中第一个为 N 的状态定位阶段 6 的阻断条件。当前未启动游戏，修改仍需用户在游戏内验证。

## 2026-10-13：恢复 1.13 的最大金额写入尝试

进一步检查 1.13 `victoria3.exe` 的公开 GUI 方法字符串，确认存在 `SetQuantity`、`GetQuantity`、`GetMaxValidQuantity`、`OnQuantityChanged`、`OnQuantityChangedSlider` 和 `OnInputTextChanged`。这只能证明方法被 GUI 数据模型导出，不能单独证明每个参数签名；原版 GUI 仍只展示滑杆事件的无参数绑定。

由于当前流程已能在初始金额 100 下完成，阶段 5 的隐藏状态现恢复为：

`ArticleDraft.SetQuantity(IntToFixedPoint(ArticleDraft.GetMaxValidQuantity))`

随后才进入阶段 6。`OnQuantityChangedSlider` 不再被当作带参数 setter 调用。诊断变量 `vob_quantity_setter` 会记录 `SetQuantity_called`，而阶段转换日志也改为记录 `SetQuantity(max)`。用户验证时应比较资金窗口中的 `qty` 是否从初始值变为 `max`；若仍为初始值，则说明该公开方法在 1.13 的 GUI 表达式中存在但参数签名或运行上下文不接受此调用，下一步需改用真实 editbox/scrollbar 控件事件。

用户验证显示 `SetQuantity(max)` 的 on_start 已执行，但金额仍为 100；同时 `EqualTo_CFixedPoint(... IntToFixedPoint(...))` 放在 `SelectLocalization` 中会触发 1.13 本地化求值错误。已删除该动态 qty=max 行，保留顶部原始 qty/max 数值诊断，并在 setter 后补调用无参数的 `ArticleDraft.OnQuantityChanged`，用于提交可能存在的输入缓存。

本轮又加入了原版编辑框使用的输入路径：`ArticleDraft.OnInputTextChanged(ToString_int32(ArticleDraft.GetMaxValidQuantity))`，随后调用 `ArticleDraft.OnQuantityChanged`。这模拟编辑框先写入文本、再结束编辑的顺序；若 1.13 接受该参数类型，金额应在阶段 6 前变为上限。

为覆盖 `SetQuantity` 可能使用整数而非 `CFixedPoint` 的签名，阶段 5 现在依次尝试 `SetQuantity(IntToFixedPoint(max))` 和 `SetQuantity(max)`，再执行编辑框提交路径。该双调用仅用于确认 1.13 的实际参数类型；确认有效后应删掉无效的那一个。

用户反馈上述隐藏 setter/编辑框调用无效，且启用 mod 后空格键无法控制时间。现已全部撤回，阶段 5 恢复为只推进到阶段 6，不再从隐藏 widget 调用任何金额写入接口。资金金额仍需手动调整；此修改优先恢复原版时间控制输入。

## 2026-10-13：绕过 setter 的“初始值即最大值”方案

1.13 原版 `common/treaty_articles/05_transfer_money.txt` 明确给出：

- `quantity_min_value` 的默认值为 100，并受 `fixed_income` 上限约束；
- `quantity_max_value` 为 `fixed_income / 4`，当 `income_transfer_relative_expenses >= 0.5` 时上限为 100；
- 数量脚本值的 `root` 是资金转让条款的 source country。

因此新增了 mod 文件 `common/treaty_articles/05_transfer_money.txt` 的局部覆盖：当玩家国家存在临时变量 `vob_force_money_max` 时，把 `quantity_min_value` 改成与原版最大值相同的公式（并保留 100 的费用上限）。条款创建时初始数量理论上就会是最大值，不需要调用 `SetQuantity`、编辑框回调或模拟鼠标拖动。批量/预览开始时设置该变量；停止、异常草稿、数量失败以及阶段 6 加入条款后清理该变量。批量模式外不改变原版最小值。

这是有意的取巧方案：在批量模式中滑杆的最小值和最大值会相等，用户不能在该流程中选择较低金额；同时它依赖 1.13 对 treaty article 局部 key 合并和 quantity script value 中 `has_variable` 的正常支持。尚未启动游戏验证，用户下一次测试应重点观察进入资金窗口时 `qty` 是否直接等于 `max`，以及 `debug.log` 是否出现 common treaty article 解析错误。

## 2026-10-12：`raw_text` 与 `text` 的修正

用户验证发现，将整条 `SelectLocalization(...)` 放入 `raw_text` 后，1.13 仍把表达式当成字面键显示为 `vob_trace_qtymax_yes`，并继续记录 `PdxDataFetchLocalizedData failed`。因此诊断状态控件已改用 `text = "[SelectLocalization(...)]"`；数值汇总行仍使用 `raw_text`。重新加载后应显示本地化结果（例如 `qty=max Y/N`），不应再显示 `vob_trace_*` 键名。

## 备用资料：Ghidra 项目

之前的 `victoria3.exe` Ghidra 逆向项目位于：

`D:\cebuild\vic3-rev\`

仅在本地 GUI、脚本接口、日志和其他直接资料不足以判断问题时考虑使用；当前阶段不需要逆向。

## 2026-10-13：空格暂停键回归排查

`ingame_hud.gui` 与 1.13 原版相比只新增了 `vob_hud = {}`。`vob_hud` 是全屏、常驻的脚本桥接 widget；此前没有声明焦点策略。虽然 `clickthrough_blocker` 只设置鼠标过滤，但全屏自定义 widget 仍可能参与 GUI 焦点路由，导致空格等全局输入动作无法到达 `timemanip`。

已给 `vob_hud` 和 `vob_running_bar` 添加 `focuspolicy = none` 与 `focus_on_visible = no`；控制面板保留默认焦点策略，以免影响其中按钮和编辑框。这不改变脚本状态机，只隔离常驻桥接层的键盘焦点。阶段 5 仍不调用任何未确认的金额 setter；资金金额自动最大化在 1.13 中尚未找到可靠的公开接口。
