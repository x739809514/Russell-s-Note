
# 🚀 **Personal RPG System — MVP 开发任务列表（P0 / P1 / P2）**

MVP 的目标非常明确：  
➡️ **能创建角色 → 定义技能 → 写日记 → AI 分析 → 更新技能 → 本地保存 → 可视化展示。**

你不需要登录、支付、技能树可视化、成就、主题等高级功能。

下面是完整拆解。

---

# 🎯 **P0 基础（必须完成，才能跑 MVP）**

## **P0.1 — 项目初始化**

-  初始化 Next.js (App Router + TS)
    
-  装 TailwindCSS
    
-  装 shadcn/ui 并配置
    
-  装 Dexie.js
    
-  配置 ESLint + Prettier
    
-  创建 GitHub 仓库（可选）
    

---

## **P0.2 — 数据库 / 本地存储（Dexie）**

### 实现 Dexie 数据库 schema

-  创建数据库 `prs-db`
    
-  创建表：
    
    -  config
        
    -  characters
        
    -  skillTrees
        
    -  dailyLogs
        

### 实现 Dexie client 封装

-  `dexieClient.ts`
    
-  增/删/查/改基础 API
    

---

## **P0.3 — 配置页（Settings）**

用户必须能输入：

### **必需功能：**

-  输入 OpenAI API Key
    
-  保存 API Key 到 IndexedDB
    
-  测试 API Key 是否可用（可选）
    

UI 内容：

- API Key 输入框
    
- 保存按钮
    
- 提示用户 “数据仅存本地，不上传到服务器”
    

---

## **P0.4 — 创建角色（Character Creation）**

实现角色初始化向导：

-  输入角色名称
    
-  输入基础信息（年龄、职业、目标）
    
-  选择角色模板（Creator / Engineer / Hybrid）
    
-  保存角色信息到 IndexedDB
    

**成功进入角色 → 打开角色主页（Dashboard）。**

---

## **P0.5 — 技能定义（Skill Setup）**

用户能为角色创建技能点：

-  技能列表页（新增、删除、编辑）
    
-  创建一级技能：
    
    - 名称
        
    - 初始等级（默认 Lv.1）
        
    - 标签（tags）
        
    - 关键词（keywords）
        
-  保存技能到 `skillTrees`
    

UI：

- 添加技能弹窗
    
- 列表显示（名称、当前等级、XP）
    

> MVP 不需要二级技能（可在 v1.0 再做）。

---

## **P0.6 — AI 分析模块（核心逻辑）**

封装一个 AI 解析函数：

### 输入：

- 用户的日记原文
    
- 技能列表（含 tags 与 keywords）
    
- 用户 API Key
    

### 输出：

- parsedActions（行为列表）
    
- skillDeltas（技能经验变化）
    

任务：

-  实现 `analyzeDailyLog()` 函数
    
-  定义默认 Prompt 模板
    
-  确保能返回结构化 JSON
    
-  处理异常（模型错误/格式错误）
    

---

## **P0.7 — 日记输入页（Daily Log）**

用户输入日记 → AI 解析 → 更新技能。

任务：

-  创建日记输入页面
    
-  文本输入框
    
-  “AI 分析并记录”按钮
    
-  调用 `analyzeDailyLog()`
    
-  显示解析结果（行为列表）
    
-  更新技能经验（XP）
    
-  写入 dailyLogs 表
    

错误处理：

- API Key 无效 → 弹提示
    
- AI 返回格式异常 → 用户可重试
    

---

## **P0.8 — 技能经验更新逻辑（XP System）**

实现基本 XP 体系：

-  加 XP
    
-  检测是否升级
    
-  升级后 level +1 & 重设 xpToNextLevel
    
-  写回 skillTrees
    

XP 曲线：

-  MVP 用线性或简单的固定 XP（如 100 XP 升级）
    

---

## **P0.9 — 仪表盘（Dashboard）**

初步 Dashboard 显示：

-  技能列表（名称、等级、XP 条）
    
-  今日成长总结
    
-  雷达图（ECharts）
    
    - 显示一级技能等级数据
        

**这就是 MVP 最大亮点。**

---

## **P0.10 — 导出 / 导入数据（基础）**

-  导出所有角色相关 JSON（打包成一个 zip 或单文件 JSON）
    
-  导入 JSON → 覆写本地数据
    

MVP 不需要 GitHub 同步。

---

# 🏆 **P1 功能**

## **P1.1 — 多角色支持**

-  角色列表页
    
-  切换角色
    
-  删除角色
    

## **P1.2 — 日志列表页**

-  显示每日日记列表
    
-  点击可查看当日解析结果
    

## **P1.3 — 经验动画（Framer Motion）**

-  XP 条动态变动
    
-  升级时“升级动画”
    

## **P1.4 — 技能编辑（非创建）**

-  修改某技能名称
    
-  修改 tags、keywords
    

## **P1.5 — 日志重解析**

-  用户可以对某天重新运行 AI 分析
    
-  覆盖旧 skillDeltas
    

---

# 💡 **P2**

## **P2.1 — AI 周报 / 月报（超强卖点）**

-  每周：自动生成总结
    
-  每月：成长曲线、趋势与建议
    

## **P2.2 — 二级技能系统**

技能可分层：

- Unity
    
    - FSM
        
    - NavMesh
        
    - BehaviorTree
        

→ XP 会同时加给一级技能和对应子技能。

## **P2.3 — 成就系统（Achievement）**

- 第一次升级
    
- 连续记录 7 天
    
- 完成技能 Lv.10
    
- 完成 20 次日记
    

## **P2.4 — 手动调整 XP**

- 用户可以微调 AI 判断（避免误判）
    

## **P2.5 — 主题 / UI 主题切换**

- Dark / Light
    
- Fantasy / Tech / Minimal RPG 风
    