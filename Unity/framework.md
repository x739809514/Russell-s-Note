# LunamiPuzzle Code Framework

本文档整理当前 `Assets/Scripts` 下的代码架构。项目整体分为三层：

- `Core`: 可复用基础设施，例如事件、存档、场景切换、UI、音频、输入、资源加载。
- `GamePlay`: 通用玩法系统，例如背包、交互物、关卡、小游戏、角色状态、条件状态。
- `Repo`: 项目具体内容层，例如具体关卡、道具、交互脚本、小游戏实现、UI 面板、常量和事件定义。

## 目录总览

```text
Assets/
  Scripts/
    Core/        基础框架和服务
    GamePlay/    通用玩法模块
    Repo/        当前游戏内容实现
```

## Core 层

### Singleton

位置：

- `Core/Singleton.cs`
- `Core/SingletonMono.cs`

用途：

- `Singleton<T>` 用于普通 C# 单例。
- `SingletonMono<T>` 用于 Unity `MonoBehaviour` 单例，常见于全局管理器。

典型使用：

- `GameManager`
- `SaveLoadManager`
- `TransitionManager`
- `AudioManager`
- `MiniGameController`
- `ItemManager`

### EventModule

位置：

- `Core/Event/EventModule.cs`
- `Core/Event/EventSender.cs`
- `Repo/Event/EventName.cs`
- `Repo/Event/EventData.cs`

职责：

- 全局事件中心。
- 使用 `EventName` 枚举作为事件 key。
- 数据通过 `object` 传递，监听端自行判断类型。

常用事件：

- `EvtStartGameEvent`: 开始指定周目。
- `EvtBeforeUnloadScene`: 场景卸载前。
- `EvtAfterLoadScene`: 场景加载后。
- `EvtUpdateItem`: 修改场景道具显示状态。
- `EvtItemUse`: 消耗背包物品。
- `EvtUpdateObject`: 修改交互物完成状态。
- `EvtUpdateRoleState`: 修改角色状态。
- `EvtFinishMiniGameStage`: 小游戏完成一个阶段。
- `EvtPassGameEvent`: 小游戏整体通过。
- `EvtRefreshBag`: 刷新背包 UI。
- `EvtRefreshScene`: 刷新场景表现。

事件流示例：

```text
交互脚本完成逻辑
  -> EventModule.Dispatch(EvtUpdateObject, EvtObjectUpdateData)
  -> ObjectManager 更新状态
  -> SaveLoadManager 自动保存
```

### SaveLoad

位置：

- `Core/SaveLoad/ISaveable.cs`
- `Core/SaveLoad/SaveLoadManager.cs`
- `GamePlay/SaveData/*.cs`

职责：

- 所有需要存档的系统实现 `ISaveable`。
- `ISaveable.Register()` 会注册到 `SaveLoadManager`。
- `SaveLoadManager` 将每个系统生成的 `SaveData` 汇总为字典并序列化。
- 存档 key 使用类型 FullName，同时兼容旧版类型名 key。

存储位置：

- 非 WebGL: `Application.persistentDataPath/SAVE/data.sav`
- WebGL: `PlayerPrefs` key `LunamiPuzzle.SaveData`

自动保存触发：

- 使用物品。
- 更新物品显示。
- 更新交互物状态。
- 更新角色状态。
- 完成小游戏阶段。
- 小游戏通过。
- 卸载场景前。
- 手动调用 `SaveLoadManager.Instance.RequestAutoSave()`。

`SaveData` 是 partial class。不同模块在独立文件里扩展字段，例如：

- `GameManagerData.cs`: 周目、语言。
- `TransitionData.cs`: 当前场景。
- `ItemSaveData.cs`: 背包。
- `ObjectData.cs`: 物品/交互物状态。
- `MiniGameData.cs`: 小游戏通过状态。
- `PlantMiniGameData.cs`: 小游戏阶段进度和 Plant 点击状态。
- `ClockMiniGameSaveData.cs`: Clock 指针角度。
- `FlowerMiniGameSaveData.cs`: Flower 当前阶段点击序列。

### Transition

位置：

- `Core/Transition/TransitionManager.cs`
- `Core/Transition/Translate.cs`

职责：

- 管理 additive scene loading。
- 通过黑幕 `CanvasGroup` 做淡入淡出。
- 转场前派发 `EvtBeforeUnloadScene`。
- 加载后派发 `EvtAfterLoadScene`。
- 持久化当前可保存场景。

转场流程：

```text
Transition(from, to)
  -> EvtBeforeUnloadScene
  -> Fade out
  -> Unload old scene
  -> Load new scene additive
  -> SetActiveScene
  -> EvtAfterLoadScene
  -> Fade in
```

### UI

位置：

- `Core/UI/UIModule.cs`
- `Core/UI/UIBase.cs`
- `Core/UI/CanvasControl.cs`
- `Repo/UI/PanelPath.cs`
- `Repo/UI/Panels/**`

职责：

- `UIModule` 是普通 C# 单例，负责打开、弹出、关闭 UI。
- UI prefab 从 `Resources` 加载。
- `PanelPath` 维护 `PanelName -> Resources path`。
- `UIBase` 提供统一生命周期钩子：
  - `DoStart`
  - `DoEnable`
  - `DoUpdate`
  - `DoFixUpdate`
  - `DoDisable`
  - `DoDestroy`

UI 管理模式：

- `OpenPanel<T>`: 替换当前顶层 UI。
- `PopPanel<T>`: 压栈显示新 UI。
- `CloseUI`: 关闭栈顶 UI。
- 关闭的 UI 会进入销毁列表，`UIModule.Update` 定期清理。

### Audio

位置：

- `Core/Audio/AudioManager.cs`
- `Core/Audio/AudioCatalog.cs`
- `Core/Audio/AudioEntry.cs`
- `Core/Audio/AudioId.cs`
- `Core/Audio/AudioBus.cs`
- `Core/Audio/AudioMusicEmitter.cs`
- `Core/Audio/AudioSfxEmitter.cs`
- `Core/Audio/AudioPointerClickSfx.cs`

职责：

- `AudioManager` 是全局音频服务。
- 支持音乐交叉淡入淡出。
- 支持 2D/3D SFX source pool。
- 支持 Master/Music/Sfx bus 音量和静音。
- Audio 配置由 `AudioCatalog` 提供。

常用接口：

- `PlayMusic(AudioId)`
- `StopMusic()`
- `PlaySfx(AudioId)`
- `PlaySfxAsync(AudioId)`
- `PlaySfxAt(AudioId, Vector3)`
- `StopAllSfx()`

### ResourceManager

位置：

- `Core/ResourceManager.cs`

职责：

- 对 `Resources.Load<T>` 和 `Resources.LoadAsync<T>` 的轻包装。
- 空路径会写 warning 并返回 null。

### Input / Cursor / Logger / CSV

位置：

- `Core/Input/**`
- `Core/Cursor/**`
- `Core/Logger/**`
- `Core/Csv/**`
- `Repo/#Generated/CSV/**`

职责：

- `InputReader` / `InputModule`: 输入读取和广播。
- `SmartCursor`: 根据可交互对象更新光标表现。
- `GameLogger` / `GameNetworkClient`: 日志和网络日志客户端。
- `CsvLoader` / `CsvBase`: CSV 基础加载。
- `Repo/#Generated/CSV`: 由工具生成的项目配置访问层，例如 `Csv.ItemCfgStore`、`Csv.InteractionCfgStore`。

## GamePlay 层

### GameManager

位置：

- `GamePlay/GameManager.cs`

职责：

- 游戏主入口管理器。
- 初始化 CSV。
- 初始化网络日志。
- 播放主 BGM。
- 打开菜单 UI。
- 监听开始游戏和场景加载事件。
- 保存当前周目和语言。

关键流程：

```text
Start
  -> Register save
  -> ApplyLanguageFromSave
  -> PlayMusic(BgmMain)
  -> Open MenuPanel

EvtStartGameEvent
  -> 设置 _gameWeek
  -> MiniGameController.ClearMiniGameState()

EvtAfterLoadScene
  -> MiniGameController.SetMiniGameStateInScene(_gameWeek)
```

### Bag

位置：

- `GamePlay/Bag/ItemManager.cs`
- `GamePlay/Bag/Data/**`
- `GamePlay/Bag/Logic/Item.cs`
- `Repo/UI/Panels/Bag/**`
- `Repo/UI/Panels/BagRd/**`

职责：

- `ItemManager` 管理背包列表和手中物品。
- 场景中的可拾取物一般继承或使用 `Item`。
- 使用物品通过 `EvtItemUse` 消耗。
- 添加物品会刷新背包 UI 并触发自动保存。

关键接口：

- `AddItemToBag(int itemId)`
- `UseItem(object obj)` 监听 `EvtItemUse`
- `SelectItemInHand(ItemDetail)`
- `ReleaseHand()`
- `IsBagContain(int itemId)`

### Interaction

位置：

- `GamePlay/Interaction/Interaction.cs`
- `GamePlay/Interaction/ZeroTargetInteraction.cs`
- `GamePlay/Interaction/SingleTargetInteraction.cs`
- `GamePlay/Interaction/MultiTargetInteraction.cs`
- `GamePlay/Interaction/MultiResultInteraction.cs`
- `Repo/Interacts/**`

职责：

- `Interaction` 是场景可交互物基类。
- 使用 `id` 关联 CSV 中的交互配置。
- 点击时根据配置目标数量分发到：
  - `OnZeroTargetInteraction`
  - `OnSingleTargetInteraction`
  - `OnMultiTargetInteraction`
- 具体关卡逻辑在 `Repo/Interacts` 中实现。

完成交互：

```text
CompleteInteraction()
  -> OnItemClick()
  -> MarkDone()
  -> Dispatch EvtUpdateObject
```

可扩展状态：

- `ExportExtraState()`
- `ImportExtraState(string state)`

这些由 `ObjectManager` 捕获和恢复，用于比 `isDone` 更复杂的交互状态。

### ObjectManager

位置：

- `GamePlay/ObjectManager/ObjectManager.cs`

职责：

- 管理场景物品可见性。
- 管理交互物完成状态。
- 管理交互物额外状态。
- 在场景卸载前捕获状态，在场景加载后恢复状态。

主要数据：

- `objectDic`: item id -> 是否显示。
- `interactionDic`: interaction id -> 是否完成。
- `interactionExtraStateDic`: 当前场景捕获的扩展状态。
- `interactionStateDic`: 跨场景保存的扩展状态。

### RoleState

位置：

- `GamePlay/RoleState/RoleStateManager.cs`
- `GamePlay/RoleState/RoleStateActor.cs`
- `GamePlay/RoleState/RoleStateType.cs`
- `Repo/Role/**`

职责：

- 管理角色状态，例如 Normal、Sick、Crazy 等。
- 监听 `EvtUpdateRoleState`。
- 场景切换时保存和恢复当前场景中的 `RoleStateActor`。

### Level

位置：

- `GamePlay/Level/LevelManager.cs`
- `GamePlay/Level/LevelBase.cs`
- `Repo/Levels/**`

职责：

- `LevelManager` 保存当前章节和已完成章节。
- `LevelBase` 监听场景刷新、加载后、卸载前三个事件，供具体关卡覆写。
- `Repo/Levels` 存放具体关卡脚本和执行逻辑。

### GameCondition

位置：

- `GamePlay/GameCondition/GameCondition.cs`
- `GamePlay/GameCondition/ConditionId.cs`

职责：

- 全局条件状态表。
- 使用 `ConditionId` 作为 key，int 作为值。
- 提供 bool/int 快捷访问。
- 条件变化时可触发 `ConditionChanged`。
- 保存到 `SaveData.conditionDic`。
- 保留旧存档字段迁移逻辑。

### MiniGame Framework

位置：

- `GamePlay/MiniGame/MiniGame.cs`
- `GamePlay/MiniGame/MiniGameController.cs`
- `GamePlay/Interfaces/IMiniGame.cs`
- `Repo/MiniGame/**`

#### MiniGame 基类

`MiniGame` 提供：

- `miniGameName`: 小游戏唯一名称。
- `isPass`: 是否通过。
- `finishGame`: 通过后触发的 UnityEvent。
- `InitMiniGame()`
- `ResetMiniGame()`
- `ChooseGameData(int week)`
- `UpdateMiniGameState()`

生命周期：

```text
Start
  -> MiniGameController.RegisterMiniGame(miniGameName)
  -> InitMiniGame()

EvtFinishMiniGameStage
  -> CheckGameStateEvent()
  -> OnCheckGameStageEvent()
```

#### MiniGameController

职责：

- 保存每个小游戏是否通过。
- 保存每个小游戏当前 stage index。
- 监听 `EvtPassGameEvent` 标记通过。
- 监听 `EvtFinishMiniGameStage` 推进 stage。
- 场景加载后由 `GameManager` 调用 `SetMiniGameStateInScene`，把存档状态应用到当前场景小游戏。

保存字段：

- `miniGameStateDic`: `miniGameName -> bool`
- `plantMiniGameStageDic`: `miniGameName -> int`

注意：`plantMiniGameStageDic` 名字历史上来自 Plant，但现在实际承担通用小游戏 stage progress。

## Repo 层

### 常量和事件

位置：

- `Repo/GlobalConst.cs`
- `Repo/Event/EventName.cs`
- `Repo/Event/EventData.cs`

职责：

- `GlobalConst` 存放资源路径、道具 ID、角色 ID、小游戏名称等常量。
- `EventName` 是全局事件枚举。
- `EventData` 定义事件载荷类型。

### 具体关卡和场景脚本

位置：

- `Repo/Levels/**`
- `Repo/Interacts/**`
- `Repo/Mono/**`
- `Repo/Role/**`
- `Repo/Items/**`
- `Repo/Statue/**`

职责：

- `Repo/Levels`: 具体关卡状态刷新和剧情执行。
- `Repo/Interacts`: 具体交互物逻辑。
- `Repo/Mono`: 非通用框架的场景 Mono 行为。
- `Repo/Role`: 具体角色表现脚本。
- `Repo/Items`: 具体道具脚本。
- `Repo/Statue`: 雕像相关状态管理。

## MiniGame 实现

### PlantMiniGame

位置：

- `Repo/MiniGame/PlantMiniGame.cs`
- `Repo/MiniGame/PlantMiniGameConfig.cs`
- `Repo/MiniGame/PlantMiniGameTarget.cs`
- `Repo/MiniGame/PlantMiniGameStateStore.cs`

特征：

- 多阶段小游戏。
- config 使用 `stages`，每个 stage 有 target id 列表。
- 点击正确目标后隐藏目标。
- 阶段完成派发 `EvtFinishMiniGameStage`。
- 最后一阶段完成派发 `EvtPassGameEvent`。
- 点击状态保存到 `PlantMiniGameStateStore`。

### ClockMiniGame

位置：

- `Repo/MiniGame/ClockMiniGame/**`

特征：

- 拖动时针/分针到配置角度。
- 指针角度通过 `ClockMiniGameStateStore` 保存。
- 通过后派发阶段完成和小游戏通过事件。

### FlowerMiniGame

位置：

- `Repo/MiniGame/FlowerMiniGame/**`

特征：

- 多阶段点击序列小游戏。
- `FlowerMiniGameConfig.stages` 每阶段配置：
  - `clickSequence`
  - `exampleScaledPetalIds`
- 玩家花瓣使用 `FlowerMiniGamePetal`。
- 示例花瓣使用 `FlowerMiniGameExamplePetal`。
- 正确点击花瓣 scale down。
- 错误点击花瓣也 scale down，然后 1 秒后 reset。
- 当前阶段点击序列通过 `FlowerMiniGameStateStore` 保存。

### SnakeMiniGame

位置：

- `Repo/MiniGame/SnakeMiniGame/**`

特征：

- 点击蛇的顺序小游戏。
- `SnakeMiniGameConfig.clickSequence` 使用 `List<int>`。
- 每条蛇用 `SnakeMiniGameTarget` 配置 `snakeId`。
- 蛇身路径动画由 `SnakePathFollower` 控制。
- 正确点击时蛇移动到目标点。
- 错误点击时所有蛇沿路径回到 origin。
- 蛇运行状态不保存；未通过时再次进入会重新初始化。

`SnakePathFollower` 约定：

- `waypoints[0]`: 最终目标点。
- `waypoints[1]`: 初始蛇头。
- 后续点：身体拐点直到初始蛇尾。
- `LineRenderer` 使用 world space。
- `EdgeCollider2D.points` 使用 local space，所以脚本会把 world positions 转回本地坐标。

## 推荐扩展模式

### 添加新存档系统

1. 创建管理器并实现 `ISaveable`。
2. 在 `Start` 中调用：

```csharp
ISaveable saveable = this;
saveable.Register();
```

3. 在 `GamePlay/SaveData` 中给 `SaveData` 增加 partial 字段。
4. 实现：

```csharp
public SaveData GenerateSaveData()
public void ReadGameData(SaveData gameData)
```

5. 状态变化后调用 `SaveLoadManager.Instance.RequestAutoSave()`，或派发已有自动保存事件。

### 添加新交互物

1. 继承 `Interaction` 或其具体子类。
2. 设置唯一 `id`，并在 CSV 中配置交互目标。
3. 覆写对应方法：
   - `OnZeroTargetInteraction`
   - `OnSingleTargetInteraction`
   - `OnMultiTargetInteraction`
   - `OnItemClick`
   - `OnInitSate`
4. 完成时调用 `CompleteInteraction()` 或直接派发 `EvtUpdateObject`。
5. 如有复杂状态，覆写 `ExportExtraState` / `ImportExtraState`。

### 添加新小游戏

1. 继承 `GamePlay.MiniGame.MiniGame`。
2. 添加 `ScriptableObject` config。
3. 在 `OnInitMiniGame` 中：
   - 读取 config。
   - 注册或构建子对象 lookup。
   - 根据 `MiniGameController.Instance.IsMiniGamePassed(miniGameName)` 初始化通过状态。
   - 如需多阶段，读取 `GetMiniGameStageProgress(miniGameName)`。
4. 完成阶段时派发：

```csharp
EventModule.Dispatch(EventName.EvtFinishMiniGameStage, new EvtMiniGameStageFinishData
{
    miniGameName = miniGameName,
    stageIndex = currentStageIndex
});
```

5. 整个小游戏通过时派发：

```csharp
EventModule.Dispatch(EventName.EvtPassGameEvent, miniGameName);
isPass = true;
UpdateMiniGameState();
```

6. 如果小游戏中间状态需要跨场景保存，使用独立 StateStore，并通过内部 `ISaveable` proxy 注册到 `SaveLoadManager`。

## 当前架构的数据流

### 新游戏

```text
MenuPanel
  -> Dispatch EvtStartGameEvent(week)
  -> SaveLoadManager 删除旧存档
  -> GameManager 记录 week 并清空小游戏状态
  -> TransitionManager 进入 startScene
```

### 加载场景

```text
TransitionManager LoadScene
  -> Dispatch EvtAfterLoadScene
  -> GameManager 应用小游戏状态
  -> ObjectManager 恢复物品和交互物
  -> RoleStateManager 恢复角色状态
  -> ItemManager 刷新背包 UI
  -> LevelBase 派生类执行 AfterLoadScene
```

### 交互获得物品

```text
Interaction / Repo.Interacts
  -> Dispatch EvtUpdateItem(itemId, visibility)
  -> ObjectManager 更新 item 可见性
  -> ItemManager.AddItemToBag(itemId)
  -> Dispatch EvtRefreshBag
  -> SaveLoadManager auto save
```

### 使用物品

```text
Bag UI 选择 item
  -> ItemManager.itemInHand
  -> Interaction 检查 CSV target
  -> ConsumeItemInHand(itemId)
  -> Dispatch EvtItemUse(itemId)
  -> ItemManager 从 bag 移除
  -> SaveLoadManager auto save
```

### 小游戏通过

```text
具体 MiniGame
  -> Dispatch EvtFinishMiniGameStage
  -> MiniGameController 推进 stage
  -> SaveLoadManager auto save
  -> Dispatch EvtPassGameEvent
  -> MiniGameController 标记 pass
  -> MiniGame.UpdateMiniGameState()
  -> finishGame UnityEvent
```

## 注意事项

- `EventModule` 的事件参数是 `object`，监听端必须做类型检查。
- `SaveData` 是 partial class，新增字段时应放到对应模块的 SaveData 文件中。
- `MiniGameController` 中的 `plantMiniGameStageDic` 名字有历史包袱，但当前实际是通用 stage progress。
- `ObjectManager` 会在场景卸载前扫描当前场景对象，所以交互物和道具需要正确设置唯一 id。
- UI prefab 路径必须登记在 `Repo/UI/PanelPath.cs`。
- `Resources` 路径常量集中在 `Repo/GlobalConst.cs`。
- DOTween 动画建议使用 `.SetLink(gameObject)`，避免对象销毁后 tween 悬挂。
- 新增脚本后，Unity 生成的 `.csproj` 可能需要重新生成，命令行 `dotnet build` 才能包含新文件。
