---
name: "ai_data_dictionary"
description: "Generates comprehensive data dictionary in Excel with Chinese field names and lowercase English. Invoke when user needs data dictionary for tables involved in an indicator or stored procedure."
---

# AI Data Dictionary (数据字典)

## 概述
生成完整的数据字典Excel文件，包含血缘分析中涉及的所有表的字段信息。每个字段必须补充中文名，所有英文字母使用小写。

## 触发条件
- 用户要求生成数据字典
- 用户要求梳理表结构信息
- 其他子技能需要数据字典作为输入

## 前置条件
- 已完成血缘分析（ai_lineage_analysis），掌握所有相关表清单
- 已连接数据库

## 执行步骤

### 1. 提取所有表的字段信息
对每张表查询 sys.columns 获取：
- 字段英文名、数据类型、长度、是否可空

```python
cursor.execute("""
    SELECT c.name, t.name, c.max_length, c.is_nullable
    FROM sys.columns c
    JOIN sys.types t ON c.user_type_id = t.user_type_id
    WHERE c.object_id = OBJECT_ID(?)
""", table_name)
```

### 2. 处理跨库表
对于跨库表，使用 SELECT TOP 1 获取字段结构：
```python
cursor.execute("SELECT TOP 1 * FROM [linked_server].[database].[schema].[table]")
columns = [(desc[0], desc[1], desc[3], desc[5], desc[6]) for desc in cursor.description]
```

### 3. 补充字段中文名
**关键步骤**，必须为每个字段补充准确的中文名：
- **优先级1**：数据库字段说明（sys.extended_properties 中的 MS_Description）
- **优先级2**：存储过程代码中的注释
- **优先级3**：字段英文名的中文翻译（如 xmh→项目号, fylb→费用类别）
- **优先级4**：采样数据分析（SELECT TOP 10 查看实际数据内容推断含义）

常见英文字段缩写映射：
- xmh/xmmc → 项目号/项目名称
- fylb/costtype → 费用类别
- sybmc → 事业部名称
- ssywmk → 所属业务模块
- sszx/ssywzx → 所属中心/所属业务中心
- cwshrq/fidate → 财务审核日期
- pjno/pjname → 项目号/项目名称
- cpx → 产品线

### 4. 英文字母全部小写
所有字段英文名、表英文名中的大写字母全部转为小写。

### 5. 标记字段使用情况
标注每个字段是否在存储过程计算中使用。

### 6. 生成Excel
使用 openpyxl 生成 Excel 文件。

## Excel列定义
| 列名 | 说明 |
|------|------|
| 表分类 | 核心输出表/中间层源表/跨库源表/配置映射维度表 |
| 表中文名 | 表的中文描述 |
| 表英文名 | 表的英文名（小写） |
| 字段英文名 | 字段英文名（小写） |
| 字段中文名 | 字段中文描述（必须补充完整） |
| 字段说明 | 字段的详细说明 |
| 数据类型 | 字段数据类型 |
| 长度 | 字段最大长度 |
| 是否可空 | YES/NO |
| 是否在计算中使用 | 是/否 |
| 表数据来源 | 数据来源说明 |

## 输出格式
文件名：`{指标名}数据字典.xlsx`
同时生成：`{指标名}数据字典.md`

## 注意事项
- 未在计算中使用的字段也必须包含在数据字典中
- 跨库表必须通过链接服务器获取完整字段信息
- 字段中文名不能为空，必须根据上下文补充
- 所有英文字母必须小写
- 表分类根据血缘分析中的角色确定
