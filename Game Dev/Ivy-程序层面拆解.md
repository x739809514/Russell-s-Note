# Ivy's Dream Cafe 程序层面拆解

## 1. 工程概况

项目主体是 Unity 工程：

- 主工程路径：`Ivy-s-Dream-Cafe/GameProjectFolder`
- 音频工程路径：`Ivy-s-Dream-Cafe/FMOD`
- 核心脚本路径：`Assets/Scripts`
- 表格配置路径：`Table/csv`、`Assets/Data/Script/Cfg/Demo`
- 运行时配置资源：`Assets/Resources/Cfg`
- 运行时 UI/音频/Timeline/字体资源：`Assets/Resources`
- 场景路径：`Assets/Scenes`

脚本规模约 730 个 C# 文件。项目使用 Unity 2D/URP、UGUI、Input System、Cinemachine、Timeline、Spine、FMOD、DOTween、Newtonsoft Json，并包含自研表格导出和配置读取系统。

美术话语权：

## 2. 依赖和技术栈

`Packages/manifest.json` 中的关键依赖：

- Unity Universal Render Pipeline 17.0.4
- Unity 2D Sprite / 2D Feature
- Unity Input System 1.18.0
- Cinemachine 2.10.4
- Timeline 1.8.10
- UGUI 2.0.0
- Visual Effect Graph 17.0.4
- Newtonsoft Json 3.2.1
- Spine Unity Runtime 4.2
- FMOD Unity 集成
- DOTween 配置资源存在于 `Resources/ConfigAssets/DOTweenSettings.asset`

项目是典型 Unity 运行时架构：大量 `MonoBehaviour` 负责场景对象和 UI，大量 `Singleton`/`SingletonMono` 负责跨场景系统。

## 3. 顶层代码组织

`Assets/Scripts` 的主要目录职责：

- `Manager`：全局管理器，包含 GameManager、UIManager、PlayerInfoManager、Save/Load、Progress、Story、Resource、Prop 等。
- `Levels`：场景/关卡生命周期基类和各场景脚本。
- `UI/UIWindow`：主要窗口，每个玩法面板一个目录。
- `SceneModel`：场景内模型和场景专属交互逻辑，例如植物、咖啡馆、卧室。
- `Character(New)`：玩家、NPC、人类、猫、Heimdale 及 FSM。
- `Collection`：收集物和线索系统。
- `FirstDream`：第一梦境玩法，含蝴蝶、镜路、门、裂隙。
- `Timeline`：自定义 Timeline Track/Clip/Behavior。
- `StoryEditor`：基于 XNode 的故事图节点。
- `Utilis`：事件、输入、时间、UI 组件、基础类、存档数据结构、本地化、对象池等。
- `Audio`：FMOD 和 Resources 音频播放管理。
- `Feature`：功能解锁逻辑。
- `Editor`：编辑器工具，生成进度节点、扫描 Spine、导出推理布局、校验配置等。
- `Generated`：自动生成的 Spine 动画名、皮肤名、StoryGraph 文件名等常量。

`Assets/Data/Script` 是表格代码生成区：

- `Cfg/Demo/*Cfg.cs`：每张表对应的数据结构。
- `Cfg/Demo/*CfgHelper.cs`：每张表对应的访问器。
- `Table/BaseCfgHelper.cs`：统一配置加载和解析。

## ==4. 启动和主生命周期==

核心入口是 `GameManager`，它继承 `SingletonMono<GameManager>`。

启动时执行：

- 初始化协程工具 `CoroutineHelper`
- 初始化剧情进度 `ProgressManager`
- 初始化功能解锁 `FeatureSystem`
- 初始化网络客户端 `Client`
- 编辑器下打开 HUD
- 启动 `NpcSender`
- 创建猫生成器 `CatSpawner`
- 初始化装饰模块 `DecoModule`
- 从 `PlayerInfoManager.statusData` 恢复游戏状态

`GameManager.Update()` 每帧驱动：

- `NpcSender.Instance.OnUpdate()`
- `NpcService.OnUpdate()`
- `HeimdaleSales.Instance.OnUpdate()`
- `MailModule.Instance.Update()`

这说明项目的主循环不是集中在一个 gameplay loop，而是由多个单例系统在 `GameManager` 下松散更新。

## ==5. 场景和关卡结构==

场景生命周期的核心基类是 `Levels.LevelBase`。

`LevelBase` 负责：

- 注册场景加载后事件 `CallAfterLoadScene`
- 注册场景卸载前事件 `CallBeforeUnloadScene`
- 设置主相机位置、正交大小、裁剪面、LayerMask 和 URP Camera RenderType
- 同步第一视角相机和 UI 相机
- 刷新 HUD
- 调用子类的 `OnStart`
- 调用子类的 `TryPlayGameSequence`
- 场景卸载前销毁猫 NPC、释放气泡

子类必须实现：

- `OnCallAfterSceneLoad()`
- `TryPlayGameSequence()`
- `OnCallBeforeSceneUnLoad()`

`LevelManager` 保存每个关卡的状态：

- `Init`：新手/引导阶段
- `Normal`：正常阶段
- `Null`：未登记

关卡状态会写入 `PlayerInfoManager`，并通过 `OnLevelStateChange` 事件通知其他系统。

## 6. 进度系统

进度由 `ProgressManager` 驱动。它读取 `GameMainCfg` 配表，为每个检查点创建一个 `ProgressBase` 子类。

关键字段：

- `CurCheckPoint`：当前正在等待完成的检查点。
- `completedCheckPoints`：已完成检查点列表。
- `checkPoints`：检查点 ID 到 `ProgressBase` 实例的字典。

关键流程：

1. `Initialize()` 清理并创建所有检查点。
2. `TryPlayGameSequence(node)` 调用对应检查点的 `Check()`。
3. 检查点内部完成条件后调用 `completeHandle`。
4. `CompleteCurCheckPoint(node)` 调用 `Finish()`、`Dispose()`，并把当前节点推进到 `next`。
5. 写入进度存档，必要时触发 `SaveManager.SaveAllData()`。

进度系统带一个 5 分钟 CD，用于剧情检查点之间留出自由活动时间。

设计上，进度节点是“配表 + 代码类”的混合结构：配表提供 ID、前置、后继、描述；代码类实现具体检查逻辑。

## ==7. 事件系统==

项目使用自研事件中心 `CMDCenter`。

接口：

- `RegisListener(IReceive receive, string cmd)`
- `UnRegisListener(IReceive receive, string cmd)`
- `DispatichReceiveCmd(string cmd, params object[] args)`

监听者实现 `IReceive.OnHandleCmd`。事件名集中在 `CmdDefine`。

特点：

- 事件按字符串分发。
- 分发前会复制监听列表，避免遍历时被修改。
- 老的回调列表 `cmdDic` 已标记废弃。
- 系统间通信大量依赖事件，例如 UI 打开关闭、剧情点击、推理选择、咖啡按钮、场景加载、收集提示、功能解锁。

优点是模块间耦合较低；缺点是字符串事件缺少编译期约束，参数类型也不安全。

## ==8. UI 系统==

UI 基类是 `UIBase`。

`UIBase` 负责：

- 确保窗口有 `Canvas` 和 `GraphicRaycaster`
- 设置 `overrideSorting`
- 提供 `ShowView`、`CloseView`、`CloseSelfView`
- 提供清理子物体、创建子物体、滚动到底等辅助方法
- 支持实现 `IReceive`

`UIManager` 是窗口管理器。它通过 `UIViewCfg` 配表加载 UI：

1. 调用 `ShowUIView(uiName, args)`。
2. 从 `UIViewCfgHelper` 取 UI 路径、脚本类型和挂载层级。
3. 从 `Resources` 加载 Prefab。
4. 挂到 `UICanvas/NormalRoot` 或 `UICanvas/TipsRoot`。
5. 如果 Prefab 没有对应 `UIBase`，按配置的 `scriptType` 动态 AddComponent。
6. 设置 Sorting Order。
7. 调用窗口 `ShowView(args)`。
8. 分发 `OnViewOpen` 和 `RefreshHud`。

UI 栈由 `showUI` 维护，同名窗口打开前会先关闭旧实例。

## ==9. 配置系统==

配置数据来自 Excel/CSV 导出。运行时使用二进制 TextAsset 形式放在 `Resources/Cfg`。

`BaseCfgHelper<T, U>` 负责：

- 根据配置类名加载 `Resources.Load<TextAsset>("Cfg/" + cfgName)`
- 用 `BinaryReader` 读取行数和每行内容
- 用 `AutoParse(strcontent.Split('\t'))` 解析字段
- 调用 `cfg.Init()`
- 建立 `key -> cfg` 字典

每张表都有对应：

- `XxxCfg`：数据结构和解析逻辑。
- `XxxCfgHelper`：继承 `BaseCfgHelper`，提供 `GetCfg()` 和 `GetAllCfg()`。

项目重要配置表包括：

- `GameMain`：主线检查点。
- `UIView`：UI 路径和脚本绑定。
- `Dialogue`、`Language`：文本与本地化。
- `Prop`、`Reward`：道具和奖励。
- `Plant`、`Seed`、`PlantEfficacy`：植物和材料功效。
- `CoffeeOrder`、`Npc`、`NpcExperience`：NPC 和订单。
- `Induction`、`InductionItem`、`InductionPage`、`DeductionQues`：线索推理。
- `Spell`、`MagicDraft`：魔咒。
- `LakeHouse`、`LakeHouseWall`、`LakeHouseQuest`：湖屋谜题。
- `Feature`、`Tutorial`、`Task`：功能、教学和任务。

## ==10. 存档系统==

存档由 `SaveManager`、`LoadManager` 和 `PlayerInfoManager` 共同承担。

`AllSaveData` 是总存档结构，字段非常多，几乎每个系统一个字符串：

- 金币
- 背包
- 装饰
- 日记
- 留言
- 星星
- 植物和花盆
- 图鉴
- 设置
- 魔咒
- 收集物
- 关卡状态
- 进度
- 游戏状态
- NPC 咖啡订单
- 户型图
- 已获得道具
- 剧情记录
- 乌鸦线索
- 推理问题
- 草稿纸
- 待办任务
- 教学
- 邮件
- NPC 解锁
- 章节信息

`SaveManager.SaveAllData()` 流程：

1. 从 `PlayerInfoManager.GetAllSaveData()` 汇总数据。
2. 用 Newtonsoft Json 序列化为字符串。
3. 用 `AesCrypt.Encrypt` 加密。
4. 写入临时文件。
5. 读取并解密临时文件做有效性验证。
6. 用 `File.Replace` 原子替换主存档，并保留 backup。

`LoadManager` 读取主存档失败后会尝试备份存档，两个都失败则创建空存档。

`PlayerInfoManager` 是最大的状态聚合器。它保存背包、装饰、植物、日记、线索、进度等大量内存数据，并提供 Init/Save/Get/Set 方法。

## ==11. 剧情系统==

剧情运行时是 `StorySystem`，编辑侧基于 `StoryEditor` 和 XNode 图。

`StorySystem` 持有：

- StoryGraph 列表
- 当前节点
- 当前播放状态
- 当前对话索引
- 当前图名
- 对话记录
- NPC 对话记录
- 选项选择记录
- UI 更新回调

它在 `Update()` 中按当前节点类型分发：

- Start
- Dialog
- Select
- Background
- Audio
- Animation
- StoryType
- Bubble
- TodoTask
- CompleteTask
- GetProp
- Collect
- NpcSkin
- SimplePop
- UnlockNpc
- ReasoningQues
- CompareItem
- CheckLanguageState
- UnlockNpcExp
- End

`StoryPanel` 是剧情 UI。它订阅 `StorySystem` 的回调，负责：

- 展示左右角色头像和名字
- 打字机文字
- 选项按钮
- 背景图
- 鼠标/F 键推进
- 锁定玩家移动
- 切换 UI 输入模式

剧情系统是跨玩法调度器，不只是对话播放器。

## ==12. 输入系统==

项目使用 Unity Input System，封装在 `Utilis.InputModule`。

它创建 `PlayerActionsMap`，并把所有 action 放进 `actionMaps` 字典。

主要能力：

- `ActivePlayerControl()`：启用玩家输入，关闭 UI 输入。
- `ActiveUIControl()`：启用 UI 输入，关闭玩家输入。
- `IsKeyPressed(name)`：判断本帧按下，并用一帧冷却避免连续检测。
- `IsKeyHeld(name)`：判断持续按住。
- `GetValue<T>(name)`：读取输入值。
- `GetControlType()`：判断键鼠或手柄。

UI 面板打开时常调用 `ActiveUIControl()`，关闭时恢复 `ActivePlayerControl()`。

## 13. 角色和 NPC 状态机

角色系统在 `Character(New)` 下，使用自研 FSM。

`FSM.Fsm` 提供：

- 状态注册
- `SwitchState`
- `OnCheck`
- `OnEnter`
- `OnExit`
- `OnUpdate`
- `OnFixUpdate`

玩家：

- `PlayerController` 是主入口。
- `PlayerAI` 驱动状态机。
- `PlayerBoardData` 是状态共享数据。
- 状态包括 Idle、Walk、Run、SitIdle、SitCoffeeIdle、Spelling、OnFloor 等。
- 使用 Spine SkeletonAnimation 控制角色动画。
- 使用 Cinemachine 跟随玩家。
- 支持锁定移动、隐藏/显示、气泡、面向目标 NPC。

NPC：

- 人类 NPC、猫、Heimdale 都有自己的 Controller、BoardData、State。
- `NpcFactory<T>` 创建/销毁 NPC。
- `NpcService.OnUpdate()` 做全局 NPC 更新。
- `NpcSender` 负责 NPC 投递/队列/剧情 NPC。

## 14. 咖啡制作程序结构

咖啡制作分 UI、控制器、动画、顾客和材料多个类。

关键类：

- `CoffeeMakingController`：制作状态机和按钮流程。
- `CoffeeMakingView`：材料选择 UI。
- `CoffeeMakingAnimationController`：制作动画。
- `Customer`：当前顾客和订单处理。
- `CoffeeMakingWareItem`：材料列表项。
- `BottleItem`、`TableItem`、`MakingButton`：交互元素。

`CoffeeMakingController` 使用两个状态：

- `CoffeeMakingState`：制作流程状态，例如 Start、Plate、Grind、Boil、Pour、Stir。
- `CoffeeMakeState`：整体 Playing/Pause。

按钮事件通过 `CMDCenter` 接收：

- 咖啡按钮点击
- 按钮点击成功
- 动画结束
- 下一个咖啡
- 咖啡等级变化
- 鼠标进入/退出按钮
- 清理 NPC
- 剧情结束

教学阶段使用 `guideDic` 强制按钮顺序。

## 15. 植物系统程序结构

植物主控制类是 `PlantControBase`。

职责：

- 初始化植物和花盆数据。
- 加载植物 Spine Prefab。
- 设置植物等级和状态表现。
- 注册玩家操作事件。
- 根据时间系统推进植物状态。
- 处理种植、浇水、施肥、除虫、收获、铲除、标签、水滴加速。
- 收获时解锁图鉴、增加背包道具、更新网络会话、显示 PopWindow。

相关数据：

- `PlantData`
- `PlantSaveData`
- `FlowerPotData`
- `PlantLevelData`
- `PlantCfg`
- `SeedCfg`
- `PlantEfficacyCfg`

植物成长依赖 `TimeSystem.AddTimeCallBack`，说明这是一个离散回调式时间系统，而不是每株植物每帧自己累计。

## 16. 收集和推理程序结构

收集系统核心类是 `CollectionSystem`。

它用字典保存：

`sceneName -> List<CollectData>`

`CollectData` 包含：

- `id`
- `collectTypes`
- `sceneName`

收集后：

- 避免重复。
- 写入 `PlayerInfoManager`。
- 检查当前推理问题线索是否已齐。
- 线索齐全时分发 `ActiveReasoningHint`。

推理 UI 是 `ReasoningPanel`，使用 partial class 拆分到多个文件：

- `ReasoningPanel.cs`
- `ReasoningField.cs`
- `ReasoningDeduction.cs`
- `ReasoningCrowClue.cs`
- `ReasoningTutorial.cs`

它负责：

- 从 `UIBoxConfigList` 和 `InductionCfg` 合成 UI 数据。
- 按 `InductionPageCfg` 展示页面。
- 只显示已收集线索。
- 查询当前问题 `QueryControl.Instance.GetCurQuestion`。
- 使用 `DeductionQuesCfg` 判断关键/辅助线索。
- 管理线索选择、数量限制、高亮、弹窗、翻页、教程、乌鸦线索。

## 17. ~~魔咒系统程序结构~~

魔咒面板是 `SpellPanel`。

启动时：

- 切换 UI 输入模式。
- 从 `SpellBag` 获取已拥有魔咒。
- 创建 6 x 16 的格子。
- 加载草稿存档。
- 为每个魔咒生成 `SpellItem`。
- 为每个魔法字符生成可拖动/可交换的 `SingleGridMono`。

保存内容：

- 草稿 InputField 内容。
- 魔法字符所在格子和位置。

成功后：

- `SpellBag.Instance.UpdateSpell(id)`
- 分发 `UnLockSpellSuccess`
- 播放 `magicspellcorrectgraph`

这个系统程序上是“UI 拖放格子 + 存档 + 配表验证”的组合。

## 18. 湖屋系统程序结构(横版卷轴)

湖屋 UI 主类是 `LakeHousePanel`。

职责：

- 初始化 `LakeHouseUtil`
- 生成家具图标和已摆放家具
- 显示/隐藏家具名称
- 根据收集物显示墙面照片提示
- 判断墙面组合是否正确
- 成功后锁定家具、改变图标、播放音效
- 关闭时尝试解锁户型图功能

相关配置：

- `LakeHouseCfg`
- `LakeHouseWallCfg`
- `LakeHouseQuestCfg`

该系统是 UI 层解谜，核心状态由 `LakeHouseUtil` 和存档数据维护。

## 19. 第一梦境系统程序结构（拆散）

第一梦境是较复杂的演出交互模块。

代表类：

- `WingController`
- `WingBase` 和四个具体翅膀类
- `WingModMono`
- `MirrorRoadCore`
- `FDMirrorRoad`
- `Mirror`
- `FdCharacterMove`
- `FdCharacterAnim`
- `DoorPre`

`WingController` 使用：

- Spine SkeletonAnimation
- PlayableDirector
- TimelineAsset
- CinemachineVirtualCamera
- SmartCursor
- FMODFirstDream
- CMDCenter 事件

流程包括：

- 初始化翅膀阶段。
- 玩家选择圆形/方形/自我选择。
- 设置 `GameManager.status.windMode`。
- 启动翅膀 UI 和 Timeline。
- 检查鼠标是否悬停在交互区。
- 所有翅膀完成后播放后续演出。
- 超时或未完成时进入自我裁剪演出。

这是项目中“叙事演出 + 状态机 + Timeline + Spine + 摄像机”的典型高表现模块。

## 20. 音频系统

项目同时使用：

- FMOD：主要事件音频、音乐、VCA。
- `Resources/Audio`：部分直接资源音效，通过 `AudioReUtility`/`AudioReManager` 播放。

`AudioManager` 负责：

- FMOD one-shot
- 循环音效
- 音乐事件
- VCA 音量控制
- 停止音乐总线
- 监听 Spine 动画事件，并根据动画名/事件名/参数播放咖啡制作音效

咖啡制作音效和动画高度绑定，例如杯子、咖啡机、倒液体、搅拌、出餐铃、吞吞吃东西等。

## 21. 资源加载

`ResourcesManager` 是对 Unity `Resources` 的薄封装。

能力：

- `Load<T>(path)`
- `LoadAll<T>(path)`
- `LoadAsync<T>(path, callback)`
- `UnloadUnusedAssets(callback)`
- 图集 Sprite 缓存加载

Sprite 加载特殊处理：如果请求类型是 `Sprite`，会从路径中拆分 atlas 路径和 sprite 名，并缓存 `Resources.LoadAll<Sprite>` 的结果。

项目大量 UI、Timeline、图片、音频、配置都走 `Resources`，开发方便，但长期可能有内存和依赖管理压力。

## 22. 编辑器工具和自动生成

项目包含多个编辑器工具：

- 进度节点生成器
- 进度节点名生成器
- 推理布局导出器
- 推理笔记编辑器
- 实验笔记编辑器
- NPC 解锁/投递工具
- Spine 资源扫描
- StoryGraph 校验
- Progress/Induction CI 校验
- 资源引用查找
- Texture 导入处理器

自动生成内容包括：

- Spine 动画名
- Spine 皮肤名
- StoryGraph 文件名
- 若干场景/角色动画常量

这说明项目有一定内容生产管线意识，尤其是剧情、推理和 Spine 内容。

## ==23. 网络/数据上报==

项目包含 `Network` 和 `KevinZonda.HerGamesSDK` 目录。

从调用点看，网络主要用于：

- 初始化客户端 `Client.Init(this)`
- 创建咖啡会话 `Client.CreateCoffeeSession()`
- 更新植物会话 `Client.UpdatePlantSession()`
- 暂停/退出时 flush/end session

它更像统计或外部服务同步，而不是核心联机逻辑。

## 24. 程序架构优点

- 系统边界在目录上较清晰：Manager、Levels、UIWindow、SceneModel、Character、Collection。
- 配表驱动程度高，适合内容型叙事游戏快速迭代。
- UI 通过 `UIViewCfg` 配置路径和脚本，减少硬编码入口。
- 进度检查点系统把章节推进集中管理。
- 存档做了加密、临时文件校验和备份替换，可靠性意识较好。
- Spine、Timeline、FMOD、Cinemachine 结合充分，适合强演出游戏。
- 编辑器工具较多，说明团队已在解决内容生产效率问题。

## 25. 程序架构风险

- `PlayerInfoManager` 过大，承担太多系统数据，容易形成“上帝对象”。
- `CMDCenter` 用字符串和 `object[]` 参数，事件错误难以编译期发现。
- 大量单例互相调用，系统初始化顺序和隐藏依赖较多。
- `Resources` 依赖广泛，缺少显式资源生命周期和依赖分析。
- UI 打开时动态 `Type.GetType` 加组件，脚本重命名或命名空间变化容易出错。
- 进度节点是配表 + 类名工厂，若生成和配置不同步容易运行时缺节点。
- 许多系统在 `OnEnable/OnDisable` 注册事件，若窗口销毁路径不一致可能出现残留监听。
- 存档字段大多是字符串二次序列化，灵活但类型边界弱，版本迁移成本会增长。
- 部分代码存在临时注释和 TODO，例如状态管理、条件系统、任务系统，说明一些核心抽象仍在演进。

## 26. 程序层改进建议

- 为 `CMDCenter` 建立类型安全事件封装，至少给高频事件定义结构体参数。
- 拆分 `PlayerInfoManager`，按系统建立独立 save service，再由总存档聚合。
- 给 `ProgressManager` 增加配置校验：检查节点类是否存在、前后继是否闭环、是否有孤儿节点。
- 将 `Resources` 路径集中生成常量或 Addressables 化，减少字符串路径错误。
- 为 UI 生命周期建立统一的打开、关闭、销毁、事件解绑规范。
- 为剧情、推理、咖啡、植物建立最小 PlayMode 测试或编辑器校验。
- 存档版本号已有 `CurrentSaveVersion`，建议尽快建立迁移器，避免后期字段变化损坏老档。
- 对玩法系统写一层“条件系统”，替代 FeatureSystem 中的临时 switch 和散落的进度判断。

## 27. 快速追代码路线

想理解项目运行，可以按这个顺序读：

1. `Assets/Scripts/Manager/GameManager.cs`
2. `Assets/Scripts/Manager/SaveLoad/LoadManager.cs`
3. `Assets/Scripts/Manager/PlayerInfoManager.cs`
4. `Assets/Scripts/Manager/GameProgress/ProgressManager.cs`
5. `Assets/Scripts/Levels/LevelBase.cs`
6. `Assets/Scripts/Manager/UIManager.cs`
7. `Assets/Scripts/Manager/Story/StorySystem.cs`
8. `Assets/Scripts/UI/UIWindow/Story/StoryPanel.cs`
9. `Assets/Scripts/Collection/CollectionSystem.cs`
10. `Assets/Scripts/UI/UIWindow/Induction/ReasoningPanel.cs`
11. `Assets/Scripts/UI/UIWindow/CoffeeMaking/CoffeeMakingController.cs`
12. `Assets/Scripts/SceneModel/PlantModel/PlantControBase.cs`
13. `Assets/Scripts/UI/UIWindow/Spelling/SpellPanel.cs`
14. `Assets/Scripts/UI/UIWindow/LakeHouse/LakeHousePanel.cs`
15. `Assets/Scripts/FirstDream/NewButterfly/WingController.cs`

