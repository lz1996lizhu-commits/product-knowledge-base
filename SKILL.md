# 产品业务知识库

## 概述

这是一个基于本地 Markdown 文件的产品业务知识库，支持通过 Git 多人协作维护，在 QoderWork 中提供对话式智能问答。

**架构说明**：本 Skill 采用"指令层与数据层分离"架构：
- **Skill 仓库**（当前目录）：仅包含 SKILL.md 指令文件
- **知识库仓库**（独立数据仓库）：存储知识条目、索引和图片

## 数据初始化（每次调用前执行）

每次调用本 Skill 时，首先执行数据同步：

```bash
KB_PATH="$HOME/.product-knowledge-base"

if [ ! -d "$KB_PATH" ]; then
  # 首次使用：浅克隆知识库（不下载 LFS 图片二进制）
  GIT_LFS_SKIP_SMUDGE=1 git clone --depth 1 \
    https://github.com/lz1996lizhu-commits/product-knowledge-base.git \
    "$KB_PATH"
else
  # 增量同步：静默拉取最新内容
  cd "$KB_PATH" && git pull origin master --depth 1 --no-edit 2>/dev/null || true
fi
```

如果 clone 或 pull 失败（网络问题），检查本地是否已有数据：
- 有数据 → 使用本地缓存继续工作，提示"当前使用离线缓存，联网后将自动同步"
- 无数据 → 提示用户检查网络连接，无法执行问答

## 迁移兼容逻辑（仅旧版升级时执行一次）

如果检测到当前 Skill 目录下存在 `knowledge/` 子目录，说明是从旧版升级而来，执行迁移：

```bash
SKILL_DIR="{当前Skill目录的绝对路径}"
KB_PATH="$HOME/.product-knowledge-base"
MIGRATE_MARKER="$SKILL_DIR/.migrated"

if [ -d "$SKILL_DIR/knowledge" ] && [ ! -f "$MIGRATE_MARKER" ]; then
  echo "检测到旧版数据，执行自动迁移..."
  
  # 1. 确保知识库数据存在于新的数据路径
  if [ ! -d "$KB_PATH" ]; then
    # 数据路径由 git clone 自动创建
    # 从当前远程仓库浅克隆最新知识库数据
    GIT_LFS_SKIP_SMUDGE=1 git clone --depth 1 \
      https://github.com/lz1996lizhu-commits/product-knowledge-base.git \
      "$KB_PATH"
  fi
  
  # 2. 清理 Skill 目录中的旧知识数据
  rm -rf "$SKILL_DIR/knowledge"
  rm -f "$SKILL_DIR/CONTRIBUTING.md"
  
  # 3. 切换 Skill 仓库 remote 到 Skill 专用仓库，并重置本地分支
  cd "$SKILL_DIR"
  git remote set-url origin https://github.com/lz1996lizhu-commits/product-kb-skill.git
  git fetch origin master
  git checkout -B master origin/master --force
  
  # 4. 标记迁移完成
  touch "$SKILL_DIR/.migrated"
  
  echo "✓ 迁移完成！Skill 已升级为新架构。"
  echo "  - 知识库数据路径: $KB_PATH"
  echo "  - Skill 仓库已切换到: product-kb-skill"
fi
```

## 知识库目录结构

知识库文件存储在 `~/.product-knowledge-base/`：

```
product-knowledge-base/
├── _index.md              # 主索引（分类+标题+标签+日期）
├── _tags_index.md         # 标签倒排索引（检索性能关键）
├── CONTRIBUTING.md        # 协作贡献指南
├── .gitattributes         # Git LFS 配置
├── product/               # 产品功能
│   ├── feature-xxx.md
│   └── ...
├── business/              # 业务流程
│   ├── process-xxx.md
│   └── ...
├── faq/                   # 常见问题
│   ├── faq-xxx.md
│   └── ...
├── guide/                 # 操作指南
│   ├── guide-xxx.md
│   └── ...
└── images/                # 图片资源（Git LFS 管理）
    ├── product/
    ├── business/
    ├── faq/
    └── guide/
```

## 问答工作流

当用户提出产品/业务相关问题时，按以下优化流程执行：

### Step 1：提取关键词

从用户问题中提取 2-5 个核心关键词/标签词。优先匹配产品术语、功能名称、业务概念。

### Step 2：标签索引检索（优先）

使用 Grep 工具在 `{KB_PATH}/_tags_index.md` 中搜索关键词：

```
Grep pattern="关键词" path="{KB_PATH}/_tags_index.md"
```

从匹配结果中获取关联文件路径列表。

### Step 3：读取相关文件

仅读取匹配到的 **最相关的 1-3 个文件**（不要贪多）：

```
Read file_path="{KB_PATH}/{matched_file_path}"
```

### Step 4：Fallback（仅在 Step 2 未命中时）

如果标签索引未命中任何结果，再读取 `{KB_PATH}/_index.md` 全量索引进行扫描匹配。

### Step 5：综合回答

基于读取到的知识条目内容：
- 组织清晰、准确的回答
- 标注引用来源（文件名）
- 如果知识库中无相关内容，明确告知用户并建议添加

### Step 6：展示来源

回答末尾使用 `present_files` 工具展示引用的知识条目文件，让用户可以直接点击查看原始文档：

```
present_files files=[{"file_path": "{KB_PATH}/product/feature-xxx.md"}]
```

### 回答规范

- 优先基于知识库内容回答，引用具体条目
- 如果知识库中无相关内容，明确告知并建议添加
- 保持回答简洁、准确、可操作

## 知识条目管理

### 添加新条目

当用户说"添加知识"、"记录一下"、"新增条目"时：

1. 确认分类（product/business/faq/guide）
2. 使用模板创建新 Markdown 文件到 `{KB_PATH}/{category}/`
3. 更新 `{KB_PATH}/_index.md` 索引
4. 更新 `{KB_PATH}/_tags_index.md` 标签索引
5. 执行 git commit

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

1. 通过标签索引定位目标条目
2. 修改内容并更新 `updated` 日期
3. 同步更新 `_index.md` 和 `_tags_index.md`
4. 执行 git commit

### 删除条目

当用户说"删除条目"、"移除知识"时：

1. 确认要删除的条目
2. 将文件移至回收站（不永久删除）
3. 从 `_index.md` 和 `_tags_index.md` 移除条目
4. 执行 git commit

### 标签索引维护

每次增删改条目后，自动更新 `_tags_index.md`：

1. 读取被修改文件的 YAML frontmatter 中的 `tags` 字段
2. 在 `_tags_index.md` 中添加/移除/更新对应标签下的文件路径
3. 保持标签按首字符分组、文件路径按字母排序

## Git 协作规范

### 远程仓库配置

- **知识库仓库**：`https://github.com/lz1996lizhu-commits/product-knowledge-base`
- **Skill 仓库**：`https://github.com/lz1996lizhu-commits/product-kb-skill`
- **主分支**：`master`（受保护，所有变更需通过 PR 合并）
- **权限问题**：如遇推送或拉取权限不足（403/401错误），请联系 **李铸** 处理权限配置

### 推送流程（每次知识变更后执行）

操作对象为**知识库仓库**（`{KB_PATH}`），不是 Skill 目录：

```bash
KB_PATH="$HOME/.product-knowledge-base"
cd "$KB_PATH"

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

# 6. 推送完成后切回 main/master 分支
git checkout master || git checkout main
```

如果推送失败报权限错误，输出提示：
> ⚠️ 推送/拉取权限不足，请联系 **李铸** 处理仓库权限配置。
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

1. Skill 调用时自动 git pull 同步知识库最新
2. 在本地知识库中添加/修改知识条目
3. 更新双索引文件，提交变更
4. 创建 `task_{用户名}_{日期}` 分支，推送并提 PR 到 master
5. 团队 review 后合并
6. 其他人下次调用 Skill 时自动 pull 到最新

## 索引维护

### 主索引 `_index.md`

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

### 标签倒排索引 `_tags_index.md`

```markdown
# 标签倒排索引

> 自动生成，请勿手动编辑。

## K
### KPI
- product/feature-indicator-setting.md
- guide/guide-performance-indicators.md

## 人
### 人才盘点
- product/feature-talent-inventory-overview.md
- guide/guide-talent-inventory.md
...
```

## 环境依赖

本 Skill 需要以下工具（首次使用时检查）：

- **git**：版本管理（通常已预装）
- **gh**（GitHub CLI）：用于创建 PR

如果 `gh` 未安装，提示用户：
> 需要安装 GitHub CLI 以支持 PR 创建。请访问 https://cli.github.com/ 安装，或运行：
> - Windows: `winget install GitHub.cli`
> - macOS: `brew install gh`
> - 安装后执行 `gh auth login` 完成认证
