# 协作贡献指南

## 概述

本知识库通过 Git 进行版本管理，支持团队多人协作维护。所有成员都可以添加、更新知识条目。

- **远程仓库**：https://github.com/lz1996lizhu-commits/product-knowledge-base
- **主分支**：`master`
- **权限问题**：如遇推送/拉取权限不足，请联系 **李铸** 处理

## 快速开始

### 1. 克隆仓库

```bash
git clone https://github.com/lz1996lizhu-commits/product-knowledge-base.git
cd product-knowledge-base
```

### 2. 拉取最新内容

每次操作前先同步远程 master 分支：

```bash
git fetch origin master
git merge origin/master --no-edit
```

### 3. 添加/修改知识条目

在 `knowledge/` 目录下对应分类中添加或修改 Markdown 文件。

### 4. 更新索引

修改 `knowledge/_index.md`，在对应分类表格中添加/更新条目信息。

### 5. 提交变更

```bash
git add .
git commit -m "add(product): 新增XXX功能说明"
```

### 6. 创建分支并推送

分支命名规则：`task_{git用户名}_{日期}`

```bash
GIT_USER=$(git config user.name | tr ' ' '_')
TODAY=$(date +%Y%m%d)
BRANCH="task_${GIT_USER}_${TODAY}"

git checkout -b "$BRANCH"
git push -u origin "$BRANCH"
```

### 7. 创建 Pull Request

提交 PR 到 `master` 分支，等待团队 review 后合并：

```bash
gh pr create --base master --head "$BRANCH" --title "$BRANCH" --body "知识库更新"
```

> ⚠️ 如果推送失败报 403/401 权限错误，请联系 **李铸** 处理仓库权限。

## 文件命名规范

- 产品功能: `product/feature-功能名.md`
- 业务流程: `business/process-流程名.md`
- 常见问题: `faq/faq-问题简述.md`
- 操作指南: `guide/guide-指南名.md`

使用小写英文、连字符分隔，保持简短清晰。

## 条目编写规范

### 必须包含

- 完整的 YAML frontmatter（title, category, tags, author, created, updated）
- 清晰的标题和结构化内容
- 相关条目的链接

### 建议包含

- 具体的示例或截图说明
- 常见问题或注意事项
- 参考链接

### 注意事项

- 每个条目聚焦一个主题，避免过长
- 使用简洁明确的语言
- 及时更新 `updated` 日期
- 添加相关条目的交叉引用

## 提交消息格式

```
动作(分类): 描述

动作: add / update / remove / fix
分类: product / business / faq / guide
```

示例：
- `add(product): 新增支付功能说明`
- `update(faq): 更新密码重置步骤`
- `fix(business): 修正订单流程中的审批条件`

## Review 要求

- 内容准确性：信息是否正确、最新
- 格式规范：是否遵循模板格式
- 索引同步：`_index.md` 是否已更新
- 链接有效：相关条目链接是否正确

## 联系方式

- 仓库权限问题：联系 **李铸**
- 内容疑问：请在团队群中讨论
