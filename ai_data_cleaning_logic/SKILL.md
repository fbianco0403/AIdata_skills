---
name: "ai_data_cleaning_logic"
description: "Generates data cleaning logic with executable SQL scripts. Invoke when user needs data cleaning procedures or ETL logic for indicator data."
---

# AI Data Cleaning Logic (数据清洗逻辑)

## 概述
基于血缘分析和数据字典生成可直接执行的数据清洗逻辑，每条清洗规则提供SQL脚本。

## 触发条件
- 用户要求生成数据清洗逻辑
- 用户需要对数据进行清洗操作
- 用户要求制定ETL清洗规则

## 前置条件
- 已完成血缘分析（ai_lineage_analysis）
- 已完成数据字典（ai_data_dictionary）
- 已连接数据库

## 执行步骤

### 1. 分析数据质量问题
基于数据质量检验规则的结果，识别需要清洗的问题：
- 空值/缺失值
- 格式不统一
- 异常值/离群值
- 重复数据
- 关联不一致

### 2. 生成清洗规则
按以下类型生成清洗逻辑：

#### 空值处理
- 关键字段空值填充默认值
- 非关键字段空值标记
- 空值记录过滤

#### 格式标准化
- 日期格式统一（yyyy-mm-dd）
- 编码格式统一（去除前后空格、统一大小写）
- 金额精度统一（保留2位小数）
- 项目号格式标准化

#### 异常值处理
- 金额异常值识别和处理（负数、超大值）
- 日期异常值处理（未来日期、过早日期）
- 编码异常值处理（不符合规范的编码）

#### 重复数据处理
- 完全重复记录去重
- 业务键重复处理（保留最新/最完整记录）

#### 关联校验与修复
- 外键关联缺失数据标记
- 编码与名称不一致修复
- 跨表数据一致性修复

### 3. 编写SQL清洗脚本
每条清洗规则提供可直接执行的SQL：
```sql
-- 清洗规则：项目号格式标准化
UPDATE table_name
SET project_number = LTRIM(RTRIM(UPPER(project_number)))
WHERE project_number IS NOT NULL
  AND project_number != UPPER(LTRIM(RTRIM(project_number)))
```

## 输出格式
文件名：`{指标名}数据清洗逻辑.md`

### 文档结构
1. 清洗规则总览
2. 空值处理规则
3. 格式标准化规则
4. 异常值处理规则
5. 重复数据处理规则
6. 关联校验与修复规则
7. 清洗执行顺序建议

每条规则包含：
- 规则编号
- 规则名称
- 问题描述
- 清洗逻辑说明
- 影响范围（表.字段）
- SQL清洗脚本
- 回滚方案

## 注意事项
- 清洗脚本必须提供回滚方案
- 建议先在测试环境执行验证
- 金额类清洗要注意精度问题
- 清洗顺序很重要：先格式标准化→再空值处理→再异常值处理→最后去重
