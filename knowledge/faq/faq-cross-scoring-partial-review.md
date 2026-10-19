---
title: 交叉评分如何实现部分指标评分后由专人复核
category: faq
tags: [交叉评分, 部分指标复核, 评分人设置, 绩效评估, 考核执行过程管理, 常见问题, 基础入门]
author: 绩效招聘
created: 2026-10-15
updated: 2026-10-15
source: https://vip.kingdee.com/knowledge/898524322405902336
product: 金蝶AI HR
module: 目标绩效云 > 个人绩效考核 > 考核执行过程管理
---

# 交叉评分如何实现部分指标评分后由专人复核

## 客户问题

绩效评估环节—交叉评分节点：评分人填完分数提交后，部分指标由部门负责人复核分数（非所有指标都要复核，只有其中几项指标需要填完再复核）。这种场景要怎么在系统中实现？

## 解决方式

交叉评分设置两个角色，评估时通过设置评分人进行控制。

### 交叉评分流程设置

交叉评分设置两个角色，并调整好角色评分的顺序、权重。如果复核不需要权重则权重为0。

![交叉评分角色设置](../images/faq-cross-scoring-partial-review-01.png)

### 绩效评估阶段操作

前提条件：被考核人的评估流程选择该交叉评分流程。

**步骤1：** 评估阶段，启动评分前，操作【设置评分人】。

![设置评分人入口](../images/faq-cross-scoring-partial-review-02.png)

**步骤2：** 对应的指标进行设置——不需要复核的指标关闭"启用"，并调整好权重，则这些指标最终在评分流程中不需要复核，而没有关闭"启用"的指标会按照交叉评分的角色设置走复核流程。

![指标启用与权重设置](../images/faq-cross-scoring-partial-review-03.png)

## 相关条目

- [绩效评估](../product/feature-performance-evaluation.md)
- [评估表单配置指南](../guide/guide-evaluation-form.md)
- [评估表单](../product/feature-evaluation-form.md)
- [评分节点为交叉评分，但是为什么加减分无法选择评分人？](faq-cross-scoring-bonus-scorer.md)
