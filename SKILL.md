---
name: product-knowledge-base
description: 产品业务知识库问答系统。基于本地Markdown知识库文件回答产品功能、业务流程、常见问题等。当用户提问产品相关问题、查询业务知识、咨询操作方法、或提到"知识库"、"查一下"、"怎么操作"时触发。支持知识条目的增删改查和多人协作维护。
install_method: share
share_id: share_1778312425125_NHzofZAD
---

# 产品业务知识库

## 概述

这是一个基于本地 Markdown 文件的产品业务知识库，支持通过 Git 多人协作维护，在 QoderWork 中提供对话式智能问答。

## 环境初始化（首次调用时执行）

首次调用本 Skill 时，检查并安装 GitHub CLI（gh），用于支持仓库推送和团队协作：

```bash
# 检查 gh 是否已安装
gh --version 2>/dev/null
```

如果 gh 未安装，按以下步骤自动安装：

**第一步：获取最新版本号**
```bash
# 通过 GitHub API 获取最新版本
curl -sL "https://api.github.com/repos/cli/cli/releases/latest" -o gh_release.json
# 用 node 解析获取版本号和下载 URL（示例输出: 2.92.0）
node -e "const r=JSON.parse(require('fs').readFileSync('gh_release.json','utf8')); const v=r.tag_name.replace('v',''); console.log(v);"
```

**第二步：按平台下载安装**

Windows 环境：
```bash
# 替换 {VERSION} 为上一步获取的版本号
curl -sL "https://github.com/cli/cli/releases/download/v{VERSION}/gh_{VERSION}_windows_amd64.zip" -o gh.zip
mkdir -p "$USERPROFILE/.local/bin"
unzip -o gh.zip -d gh_extracted
cp gh_extracted/bin/gh.exe "$USERPROFILE/.local/bin/gh.exe"
export PATH="$PATH:$USERPROFILE/.local/bin"
gh --version
```

macOS 环境：
```bash
brew install gh
```

Linux 环境：
```bash
curl -sL "https://github.com/cli/cli/releases/download/v{VERSION}/gh_{VERSION}_linux_amd64.tar.gz" -o gh.tar.gz
tar -xzf gh.tar.gz -C /tmp
cp /tmp/gh_*/bin/gh /usr/local/bin/gh
gh --version
```

**第三步：认证登录**

安装完成后，如果用户尚未登录 GitHub，提示执行：
```bash
gh auth login
```

> 参考：https://cli.github.com/

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

### 远程仓库配置

- **远程仓库**：`https://github.com/lz1996lizhu-commits/product-knowledge-base`
- **主分支**：`master`（受保护，所有变更需通过 PR 合并）
- **权限问题**：如果遇到推送或拉取权限不足（403/401错误），请联系 **李铸** 处理权限配置

### 推送流程（每次知识变更后执行）

每次推送需创建独立分支并提交合并请求（PR），分支和 PR 命名规则：

- 分支名：`task_{git用户名}_{日期}`，例如 `task_zhangsan_20260509`
- PR 标题：与分支名一致

操作步骤：
```bash
# 1. 确保远程仓库已配置
git remote get-url origin || git remote add origin https://github.com/lz1996lizhu-commits/product-knowledge-base.git

# 2. 获取当前 git 用户名和日期，创建分支
GIT_USER=$(git config user.name | tr ' ' '_')
TODAY=$(date +%Y%m%d)
BRANCH="task_${GIT_USER}_${TODAY}"

# 3. 基于本地最新提交创建推送分支
git checkout -b "$BRANCH"

# 4. 推送分支到远程
git push -u origin "$BRANCH"

# 5. 创建 PR 到 master 分支
gh pr create --base master --head "$BRANCH" --title "$BRANCH" --body "知识库更新"

# 6. 推送完成后切回 main 分支
git checkout main
```

如果推送失败报权限错误，输出提示：
> ⚠️ 推送/拉取权限不足，请联系 **李铸** 处理仓库权限配置。
> 仓库地址：https://github.com/lz1996lizhu-commits/product-knowledge-base

### 拉取更新流程（同步远程最新内容）

每次使用知识库前，应从远程 master 分支拉取最新内容：

```bash
# 1. 确保远程仓库已配置
git remote get-url origin || git remote add origin https://github.com/lz1996lizhu-commits/product-knowledge-base.git

# 2. 拉取远程 master 分支最新内容到本地 main
git fetch origin master
git merge origin/master --no-edit

# 如果本地分支名为 main，也可以：
# git pull origin master
```

如果拉取失败报权限错误，输出提示：
> ⚠️ 拉取权限不足，请联系 **李铸** 处理仓库权限配置。
> 仓库地址：https://github.com/lz1996lizhu-commits/product-knowledge-base

### 提交规范

```
知识库提交消息格式:
- add(category): 新增XXX条目
- update(category): 更新XXX条目
- remove(category): 移除XXX条目
- fix(category): 修正XXX条目错误
```

### 协作流程总结

1. 拉取远程 master 最新内容
2. 在本地 main 分支上添加/修改知识条目
3. 更新索引文件，提交变更
4. 创建 `task_{用户名}_{日期}` 分支，推送并提 PR 到 master
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
