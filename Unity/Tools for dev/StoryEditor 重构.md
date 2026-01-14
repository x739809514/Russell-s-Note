方案一：节点导航器 + 策略模式（推荐）

  核心思想

  用字典映射替代类型判断链，用策略模式替代switch-case

  具体设计

  1. 创建 NodeNavigator（节点导航器）
  职责：统一管理节点类型识别和导航
  位置：新建 Assets/Scripts/StoryEditor/Core/NodeNavigator.cs

  核心功能：
  - 维护 Dictionary<Type, Current> 映射表
  - 提供 Navigate(node, out current) 方法
  - 支持动态注册新节点类型

  好处：
  ✅ 所有Node的MoveNext方法从85行 → 3行
  ✅ 1,360行重复代码 → 50行映射表
  ✅ 新增节点只需注册一次

  2. 重构所有Node的MoveNext
  // 之前：85行重复代码
  public Node MoveNext(out Current current)
  {
      Node temp = this;
      current = Current.XXX;

      var exitPort = GetOutputPort("output");
      if (exitPort.IsConnected) {
          var nextNode = exitPort.Connection.node;

          if (nextNode is DialogNode dialogNode) { ... }
          else if (nextNode is SelectNode) { ... }
          else if (nextNode is BackGroundNode) { ... }
          // ... 15个else if
      }
      return temp;
  }

  // 之后：3行代码
  public Node MoveNext(out Current current)
  {
      current = Current.XXX; // 只需标记当前节点类型
      return NodeNavigator.Navigate(this, out current);
  }

  3. 拆分 StorySystem（733行 → 4个类）

  StoryFlowController (150行)
  ├── 职责：流程控制、状态管理
  ├── 字段：curNode, curState, graphList
  └── 方法：StartStory(), PauseStory(), ResumeStory()

  NodeExecutorManager (200行)
  ├── 职责：节点执行调度
  ├── 使用策略模式：Dictionary<Current, INodeExecutor>
  └── 方法：ExecuteNode(node, current)

  StoryRecorder (100行)
  ├── 职责：剧情记录管理
  ├── 方法：RecordDialog(), SaveRecord(), LoadRecord()
  └── 数据：storyRecord, dialogHistory

  StoryEventBridge (80行)
  ├── 职责：事件桥接
  └── 方法：监听/发送Command事件

  4. 节点执行器接口
  public interface INodeExecutor
  {
      void Execute(Node node, StoryContext context);
  }

  // 每个节点类型一个执行器
  public class DialogNodeExecutor : INodeExecutor { ... }
  public class AudioNodeExecutor : INodeExecutor { ... }
  // ... 17个执行器

  // 注册执行器
  ExecutorRegistry.Register(Current.Dialog, new DialogNodeExecutor());
  ExecutorRegistry.Register(Current.Audio, new AudioNodeExecutor());

  ---
