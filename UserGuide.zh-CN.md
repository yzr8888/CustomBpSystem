# CBP 3.0 插件使用指南（0.5.14）

本指南适用于 UE 5.5。插件目录为 `Plugins/CustomBlueprintRuntime`。

![完整编辑器：创建、函数、图表和变量面板](Images/cbp-editor-overview.png)

## 1. 最小接入

1. 在保存图表的 Actor 上添加 `Custom Blueprint Runtime` 组件。
2. 建议开启该 Actor 的 Replication。组件在服务器上也会自动保证所属 Actor 可复制。
3. 创建一个继承自 `CBPGraphEditorWidget` 的 Widget Blueprint。
4. 创建控件后调用 `Initialize Editor From Actor`，传入保存图表的 Actor。
5. 将控件 `Add to Viewport`，并按项目需要切换输入模式和显示鼠标。

如果要自由布局，请分别放置以下四个独立控件，再调用一次
`Initialize Independent Editor Panels From Actor`：

- `CBPGraphWidget`
- `CBPCreatePanel`
- `CBPVariableDetailsPanel`
- `CBPFunctionDetailsPanel`（原生类名仍是 `CBPFunctionToolbarPanel`）

顶部操作栏也是独立控件 `CBPGraphToolbarPanel`。自由布局时把它放到任意
UMG 容器，并在图表初始化后调用 `Initialize Toolbar`，传入同一个
`CBPGraphWidget`。完整的 `CBPGraphEditorWidget` 会自动创建并初始化它。

操作栏提供四个实际功能：`RUN` 内部先验证图表，再通过服务器权威路径启动
运行会话；`STOP` 停止整个运行会话并恢复运行前状态；`SAVE` 使用旁边可编辑的
Slot Name 调用服务器权威保存；`LOAD` 从同一 Slot 读取并同步完整图表。运行中
读取会先停止并恢复运行前状态，再载入存档。解释执行不需要单独生成代码，因此不再显示
重复的 `COMPILE` 按钮。操作完成后
可绑定 `On Toolbar Action Finished` 获取动作、成功状态和提示文本。

三个预制事件只在点击 `RUN` 后执行，不再跟随 Actor/组件 BeginPlay 自动启动。
服务器会先执行 `Event Construction`，完成后执行 `Event Begin Play`，最后才
启用 `Event Tick` 的间隔调度；图表不需要 `Start` 节点。客户端提交 RUN 后，
顶部状态会继续显示服务器实际返回的开始、失败、停止和 Print 信息。

每个面板都有独立的 `Appearance`，可以修改背景、标题、强调色、选择色、文字、间距和删除按钮颜色。需要完全自定义时，可继承对应控件并关闭 `Use Built-in Layout`。

如果只初始化函数面板，调用新的 `Initialize Function Panel` 并只传已初始化的 `CBPGraphWidget`；它会自动取得同一个 Runtime Component，不再需要手工 `Get Component By Class`。也可直接调用 `Initialize Function Panel From Actor`。旧的 `Initialize Function Toolbar` 已隐藏并仅保留兼容。

## 2. 创建节点与变量

- Create 面板中的普通节点来自图表的 `Graph Style > Available Nodes`；启用
  `Use All Registered Nodes` 时显示全部。新建或修改节点 DataAsset 后会自动重扫注册表，
  无需重启编辑器。
- 点击 `+ ADD VARIABLE` 创建变量；先选类型，再输入名称和默认值，最后点击 `Save Variable`。
- 变量保存成功后，Create 面板会自动出现 `Get 变量名` 和 `Set 变量名`，不需要手工维护条目。
- Create 面板创建的节点会落在最后一次鼠标/右键所在的图表坐标；尚未移动鼠标时使用视图中心。
- 自定义右键菜单可以先调用 `Set Node Spawn Graph Position`，再调用创建函数。
- 右键按住拖动仍用于平移画布；只有没有发生拖动的右键单击才触发统一的 `On Graph Pointer Context`。插件不会自动打开菜单。
- 对事件中的 `Context` 枚举执行 Switch：`Canvas` 调用 `Open Create Node Menu`，`Node` 调用 `Open Node Context Menu`，`Pin` 调用 `Open Pin Context Menu`。
- 创建菜单支持搜索、折叠分类和 `Context Sensitive`；节点菜单提供删除选择、折叠为函数、断开选择节点的全部引脚；引脚菜单提供断开该引脚和断开该节点全部连线。
- 绑定 `On Graph Menu Item Selected` 可在任何菜单条目被点击后收到动作类型、节点/引脚 ID、变量/函数来源 ID、创建节点/函数 ID、图表坐标和是否成功。
- 三种菜单使用紧凑宽度、最大高度和屏幕工作区约束，不再向屏幕底部无限延伸。
- 引脚右键和连线落点默认检测可见插口中心的 16 px 半径；连线松到节点主体但没有直接命中插口时，会自动选择该节点上距离最近且方向、类型兼容的引脚。在多选节点状态下右键任何所选节点的引脚会返回 `Node` 上下文，而不是破坏多选后返回 `Pin`。
- 拖线按下使用当前鼠标事件的实时几何，不依赖上一帧缓存；即使多人状态刷新临时转移鼠标捕获，图表也会用稳定的节点/引脚 ID 继续拖动并在松开时完成连接。

变量详情中的修改只有点击 `Save Variable` 后才会提交到服务器。名称、类型和默认值都会复制给其他客户端；如果当前值仍等于旧默认值，保存新默认值时当前值也会一起更新。

## 3. 折叠为函数

折叠是图表上的主动操作，不是函数面板自动完成的操作：

1. 在图表中选择需要折叠的非事件节点。
2. 从自己的右键菜单、快捷键或按钮调用 `Collapse Selection To Function`。
3. 输入函数名称。选中子图会进入复制的函数存储区，原位置生成函数调用节点。
4. 在 Functions 面板选择该函数，编辑名称，并用 `+ INPUT` / `+ OUTPUT` 增加任意数量的接口引脚。
5. 双击主画布上的函数调用节点或 Functions 列表条目，会让当前 `CBPGraphWidget` 直接切换到函数画布；其中节点的创建、移动、删除、默认值与连线修改都提交给服务器并复制给其他客户端。点击函数面板顶部的 `< EVENT GRAPH`，或调用 `Return To Main Graph` 返回事件图表。关闭 `Switch Main Graph On Double Click` 后，仍可选择旧的独立窗口模式或监听 `On Function Editor Opened` 自定义承载方式。

接口引脚使用稳定 ID 更新所有已有调用节点。函数图固定显示一个包含全部公开输入的 `Function Entry` 节点和一个包含全部公开输出的 `Return` 节点，布局与 UE 蓝图函数一致；旧存档中的分散接口节点会在服务器加载时自动合并。未连线的输入使用调用节点默认值，未连线的输出保持下游默认值；执行展开时边界节点会被安全穿透，不作为普通运行节点执行。入口事件和已有函数调用不允许再次折叠，以避免递归定义。

## 4. 三个预制事件

- `Event Construction`：点击 `RUN` 后在服务器首先触发一次。
- `Event Begin Play`：Construction 执行完成后在服务器触发一次；即使图中没有 Tick 事件也会正常执行。
- `Event Tick`：Begin Play 完成后才由服务器调度。`Interval Seconds = 0` 表示每个服务器帧；大于 0 表示按指定秒数触发。`Delta Seconds` 是本 Tick 节点距离上次触发的时间。

`STOP` 会取消当前所有节点执行和 Tick 调度，清除节点高亮与连线动画，并把
运行时变量值恢复为点击 RUN 前的快照。已经对世界造成的外部副作用（例如已
生成 Actor、发送网络请求或写文件）无法通用回滚，需要对应节点自行提供撤销逻辑。

`Print` 必须连接到实际执行链。执行后内容会写入服务器 Output Log，通过
`On Execution Event` 的 `DebugMessage` 同步，并默认显示在客户端游戏画面和
顶部操作栏。可在 Runtime Component 上关闭 `Print Debug Messages To Screen`
或修改显示时长与颜色。

只有下游节点实际取得并消费执行令牌时，对应连接才显示流动效果；
运行快照中的函数边界引脚会先解析为当前画布中的可见执行连线，避免图表重建后
因为引脚 ID 映射不同而丢失动画。
未执行到的节点和连线不会动画。即使没有指定 `GraphStyle` 也使用 Moving Dots
默认样式，不需要额外设置。默认脉冲为高亮青色并保留约 1.25 秒，点击 STOP 会立即清除。

同一图表中每种预制事件最多一个。所有执行与生命周期事件以服务器为准。

## 5. 多人协作与冲突提示

![两名玩家选择同一节点：玩家状态、警告条和亮色边框](Images/cbp-collaboration-selection.png)

`CBPGraphWidget` 默认开启 `Enable Collaboration Presence`。本地选择、编辑引脚、移动节点和连接引脚时，会把玩家状态发送到服务器，再复制给其他客户端。

- `Local Player Avatar`：可选头像资源；未设置时显示玩家名首字母色块。
- `Local Player Presence Color`：Alpha 为 0 时自动按玩家生成稳定颜色。
- 选中节点默认使用 4 px 亮青色边框和浅色填充；可在自定义 `CBPNodeWidget` 子类中修改 `Selection Outline Color`、`Selection Fill Color` 和 `Selection Outline Thickness`。
- 同一节点出现多名玩家时，节点顶部会列出姓名与 `editing`、`moving`、`connecting` 状态，并显示橙色共享警告。
- 玩家条现在从节点原坐标向上延伸，显示/隐藏协作信息不会改变节点本体的位置。
- 变量详情面板会显示正在编辑所选变量的玩家；多人打开同一变量时显示 `SHARED EDITING`。
- 可绑定 `On Collaboration Conflict`，额外显示 Toast、声音或确认窗口。

每个本地玩家创建图表控件后调用一次 `Set Local Player Collaboration Profile`，传入自定义名称、头像纹理和颜色。也可以直接在目标 `CBPRuntimeComponent` 调用 `Set Local Collaboration Profile`。该状态经过服务器复制，但不写入图表存档。

冲突提示是协作提醒，不是硬锁。所有修改仍由服务器验证和排序，最后一个被服务器接受的修改成为最终状态。若项目需要禁止抢占，请重写 `Can Remote Player Edit`，加入队伍、距离、角色或所有权规则。

## 6. 网络设置

持久数据路径为：

`图表控件 -> 目标 CBP 组件 -> 玩家拥有的命令网关 -> 服务器 -> FastArray -> 所有客户端`

插件会在每个 PlayerController 上自动创建复制编辑网关。客户端过早发出的命令会排队，网关可用后按顺序发送。图表、变量、函数和函数体使用 FastArray，晚加入客户端也会收到完整当前状态。

多人拖动选中节点时，插件只发送一次批量 `Move Nodes` 请求并只增加一次图表版本，避免每个节点一条 RPC。玩家在线状态也使用 FastArray 增量复制，只重建状态发生变化的节点。

推荐 PIE 验证：将 Play Mode 设为至少 2 Players，Net Mode 使用 Listen Server。两个窗口打开同一图表，选择同一节点，然后在一个窗口移动或编辑；两个窗口都应看到玩家条与冲突警告。

## 7. 保存和加载

在目标 `CBPRuntimeComponent` 上调用：

- `Save Graph To Slot(SlotName, UserIndex)`
- `Load Graph From Slot(SlotName, UserIndex)`
- `Does Graph Save Exist`
- `Export Graph Save Data` / `Import Graph Save Data`

客户端的保存/加载请求会经过拥有权网关在服务器执行。快照包含节点、连线、变量定义与当前值、函数体、函数签名；玩家选择/头像/冲突状态属于临时协作状态，不写入存档。

## 8. 发布前检查

- Shipping 中保留服务器权限校验，并按项目重写 `Can Remote Player Edit`。
- 为图表 Actor 设置合适的网络相关性，不要无条件复制给不需要编辑器的玩家。
- 大型图表使用分类、搜索和节点白名单。
- 在 Listen Server 和 Dedicated Server 两种模式分别测试保存、函数签名修改、变量重命名和 Tick 间隔。

0.5.14 的工作区自动化验证覆盖显式运行会话、Construction/Begin Play/Tick 顺序、STOP 变量恢复、实际消费执行令牌后触发连线动画、可靠的多人运行可视化同步、连接层双路径帧刷新、可见执行连线解析、仅已遍历连线动画、无样式资产默认动画、顶部 SAVE/LOAD、运行中读取安全停止、节点 DataAsset 热刷新、图表可用节点白名单、批量节点位置、同节点冲突、变量编辑状态、玩家资料、任意数量函数接口、合并式函数入口/返回节点和函数体权威编辑。本阶段按开发要求不进行打包隔离验证。
