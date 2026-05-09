---
name: product-knowledge-base
description: 产品业务知识库问答系统。基于本地Markdown知识库文件回答产品功能、业务流程、常见问题等。当用户提问产品相关问题、查询业务知识、咨询操作方法、或提到"知识库"、"查一下"、"怎么操作"时触发。支持知识条目的增删改查和多人协作维护。
---

# 产品业务知识库

## 概述

这是一个基于本地 Markdown 文件的产品业务知识库，支持通过 Git 多人协作维护，在 QoderWork 中提供对话式智能问答。

## 知识库目录

知识库文件存储在与本 Skill 同级的 `knowledge/` 目录中：

```
product-knowledge-base/
├── SKILL.md
├── CONTRIBUTING.md
└── knowledge/
    ├── _index.md              # 知识库索引（自动维护）
    ├── product/               # 产品功能
    │   ├── feature-xxx.md
    │   └── ...
    ├── business/              # 业务流程
    │   ├── process-xxx.md
    │   └── ...
    ├── faq/                   # 常见问题
    │   ├── faq-xxx.md
    │   └── ...
    └── guide/                 # 操作指南
        ├── guide-xxx.md
        └── ...
```

## 问答工作流

当用户提出产品/业务相关问题时：

1. **读取索引**: 读取 `knowledge/_index.md` 获取所有知识条目概览
2. **定位相关条目**: 根据用户问题的关键词匹配相关条目
3. **读取详细内容**: 读取匹配到的知识条目文件
4. **综合回答**: 基于知识条目内容组织回答，标注来源条目

### 回答规范

- 优先基于知识库内容回答，引用具体条目
- 如果知识库中无相关内容，明确告知用户并建议添加
- 回答末尾标注来源: `[来源: knowledge/xxx/yyy.md]`
- 保持回答简洁、准确、可操作

## 知识条目管理

### 添加新条目

当用户说"添加知识"、"记录一下"、"新增条目"时：

1. 确认分类（product/business/faq/guide）
2. 使用模板创建新 Markdown 文件
3. 更新 `_index.md` 索引

### 条目文件模板

每个知识条目文件遵循以下格式：

```markdown
---
title: 条目标题
category: product | business | faq | guide
tags: [标签1, 标签2]
author: 作者
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

# 条目标题

## 内容

[具体内容]

## 相关条目

- [相关条目1](../xxx/yyy.md)
```

### 更新条目

当用户说"更新知识"、"修改条目"时：

1. 定位目标条目
2. 修改内容并更新 `updated` 日期
3. 同步更新 `_index.md`

### 删除条目

当用户说"删除条目"、"移除知识"时：

1. 确认要删除的条目
2. 将文件移至回收站（不永久删除）
3. 从 `_index.md` 移除条目

## Git 协作规范

### 分支策略

- `main`: 正式知识库，受保护
- `update/xxx`: 知识更新分支，合并前需 review

### 提交规范

```
知识库提交消息格式:
- add(category): 新增XXX条目
- update(category): 更新XXX条目
- remove(category): 移除XXX条目
- fix(category): 修正XXX条目错误
```

### 协作流程

1. 从 `main` 创建更新分支
2. 添加/修改知识条目
3. 更新索引文件
4. 提交并创建 PR
5. 团队 review 后合并

## 索引维护

`_index.md` 文件格式如下，每次增删改条目后自动更新：

```markdown
# 知识库索引

## 产品功能 (product)
| 文件 | 标题 | 标签 | 更新日期 |
|------|------|------|----------|
| product/feature-xxx.md | XXX功能说明 | 标签1,标签2 | 2025-01-01 |

## 业务流程 (business)
...

## 常见问题 (faq)
...

## 操作指南 (guide)
...
```
