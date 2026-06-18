---
name: "ai_lineage_analysis"
description: "Traces data lineage of stored procedures to source tables with field-level detail. Invoke when user needs blood lineage analysis or data flow tracing for a stored procedure or indicator."
---

# AI Lineage Analysis (血缘分析与算法)

## 概述
对指定的存储过程进行血缘分析，追溯到最源头的表和字段，生成字段级血缘关系文档。

## 触发条件
- 用户提供存储过程名称，要求追溯血缘
- 用户需要了解指标的数据来源和流转关系
- 用户要求分析数据流向

## 前置条件
- 已通过 db_connect_sqlserver 技能连接到 SQL Server 数据库

## 执行步骤

### 1. 提取存储过程定义
```python
# 使用 pyodbc 连接数据库
# 查询 sys.sql_modules 获取存储过程定义
cursor.execute("SELECT definition FROM sys.sql_modules WHERE object_id = OBJECT_ID(?)", sp_name)
```

### 2. 递归提取依赖对象
- 从存储过程代码中识别所有被调用的子存储过程、视图、函数
- 对每个子对象递归提取其定义
- 直到所有叶子节点都是基础表

### 3. 提取所有源表结构
- 对每个源表查询 sys.columns 获取完整字段信息
- 对于跨库表（如 link_u8if.reportnew.dbo.xxx），使用 SELECT TOP 1 方式获取字段结构
- 跨库表字段获取方法：
```python
cursor.execute("SELECT TOP 1 * FROM [linked_server].[database].[schema].[table]")
columns = [desc[0] for desc in cursor.description]
```

### 4. 分析字段级血缘
- 追踪每个输出字段的数据来源
- 识别字段转换逻辑（直接映射、计算派生、聚合等）
- 标注关键转换节点

### 5. 生成血缘文档
输出 Markdown 文档，包含：
- **整体血缘图**：存储过程 → 子存储过程/视图 → 源表
- **表级血缘**：每张源表到目标表的流转关系
- **字段级血缘**：每个输出字段对应的源字段和转换逻辑
- **跨库依赖**：标注跨数据库的表依赖关系

### 6. 生成血缘思维导图
同步生成 Mermaid 格式的思维导图文件，包含4种可视化视图：

#### 视图1：思维导图（Mermaid Mindmap）
从指标出发，按层级展开：
```
root((指标名称))
  外部系统数据源
    OA系统 → 各OA源表
    EBS系统 → 各EBS源表
    阿里商旅 → 各阿里源表
    HRM系统 → HRM源表
    MueTec系统 → MueTec源表
    其他系统 → 其他源表
  配置映射维度表
    各配置表
  存储过程计算
    主存储过程
    各子存储过程
  输出目标表
    各输出表（含字段数）
```

#### 视图2：数据流向图（Mermaid Flowchart）
从外部系统 → 中间层源表 → 配置映射维度表 → 存储过程 → 输出目标表，用 subgraph 分组，用颜色区分层级：
- 外部系统：默认色
- 中间层源表：浅蓝
- 跨库源表：浅绿
- 配置映射维度表：虚线连接
- 存储过程：深蓝填充
- 输出目标表：红色填充

#### 视图3：数据来源构成（Mermaid Pie）
费用明细表各数据源的占比饼图，每个 UNION ALL 分支作为一项。

#### 视图4：分摊层级结构（Mermaid Flowchart TD）
如存在分摊逻辑，展示分摊层级树：
- 原始费用 → 直接综费 / Level1分摊 / Level2分摊 / Level3分摊
- 各层级汇总关系

## 输出格式

### 文件1：`{指标名}血缘分析与指标算法.md`
1. 概述（存储过程功能说明）
2. 整体数据流向图
3. 表级血缘关系
4. 字段级血缘关系（按目标表分组）
5. 跨库依赖说明
6. 关键转换节点说明

### 文件2：`{指标名}血缘思维导图.md`
1. 思维导图（Mermaid Mindmap）
2. 数据流向图（Mermaid Flowchart）
3. 数据来源构成（Mermaid Pie）
4. 分摊层级结构（Mermaid Flowchart TD）

## 注意事项
- 跨库表需要通过链接服务器访问，注意连接方式
- 视图需要展开到基础表
- UNION ALL 操作需要分别追溯每个分支
- 临时表和表变量需要追溯其数据来源
- 加密存储过程无法获取定义，需标注
- 思维导图中的表名需与血缘文档保持一致
- Mermaid 语法中节点文字含特殊字符时需用引号包裹
- 思维导图需在支持 Mermaid 渲染的编辑器中查看（VS Code + Mermaid插件、Typora、GitHub等）
