---
title: 人才对比卡片二开实操步骤（移动端）
category: guide
cloud: 人才发展云
tags: [人才对比, 二开, 移动端, 人才星图, 卡片开发, 苍穹, 开发指南]
aliases: [人才对比卡片移动端, 移动端对比卡片二开]
author: 金蝶云社区
created: 2026-09-28
updated: 2026-09-28
source: https://vip.kingdee.com/knowledge/892426100943899392
---

# 人才对比卡片二开实操步骤（移动端）

以「教育经历」区块为例，按实际操作顺序记录。**本文只讲移动端**，PC 端另见 PC 交付包。

配套代码：`sample/frontend/`（前端控件工程）、`sample/backend/`（后端插件 + 控件方案预置 SQL）、`figma/`（设计稿产物）。  
对接机制看 `common/doc/人才对比数据展示控件事件规范.md`，**必读**，  
四种 setData 指令、空数据、超长截断、pair 切换规范都在里面。

占位符 `<二开工程>`、`<你的包>`、`<你的控件目录>` 替换为实际值。

整体做法与 PC 端同构：**不新建表单、没有视图配置页**，直接扩展标品的移动端人才对比表单，  
在内容面板下加自定义控件，再挂一个扩展插件推数据。控件标识自己起名（一张表单挂十几个控件，不能重名）。

## 移动端与 PC 端的差别（先看这个）

两端是**两个独立控件**，控件标识、控件方案、插件、预置 SQL 全部各一份，不能复用。  
下发数据的字段也可以不同（本例移动端就少了 PC 的 211 / 985 标签）。

|  | PC 端 | 移动端 |
| --- | --- | --- |
| 展示形态 | 多列并排，横向滚动 | **两列并排 + pair 窗口滑动** |
| pair 切换 | 无 | **必须接 `usePairSync`**（window 事件通道，不经后端） |
| 一个人都没有 | 整块 `return null` | **保留标题与锚点** + 缺省态插画 |
| 超长文本提示 | antd `Tooltip`（hover） | **原生 `title`** + 点击委托（触屏无 hover） |
| `remove`/`move` 广播 | 只遍历 `contentflex` 直接子项 | 从 `contentflex` **递归**下发，可嵌套 |
| 控件放哪 | `contentflex` 直接子级 | **`scrollflex` 下**（否则不进锚点导航） |
| 插件父类 | `AbstractFormPlugin` | **`AbstractMobFormPlugin`** |
| 选人 F7 字段 | `multiemployee`（`MulEmployeeEdit`） | **`multiassignment`**（`MulBasedataEdit`，选组织分配） |
| 浮层不被裁 | 需元数据加 `$ { overflow: visible; }` | 浮层挂 body，**不需要**此配置 |
| 缺省态插画 | 全目录没有，别画 | **有**，公共外壳内置 |

四种 action 的 payload 结构、`employeeId` 主键、`times` 约定**两端完全一致**。

五步，前三步在 IDE 让 AI 做，后两步在平台配：

| 步骤 | 做什么 | 在哪 |
| --- | --- | --- |
| step1 | 触发技能生成前端控件与后端插件 | IDE |
| step2 | 生成控件方案预置脚本 | IDE |
| step3 | 换真实取数（step1 说清了可省） | IDE |
| step4 | 扩展 `hrti_talentcomparison_m`，加控件、改标识、绑方案、挂插件 | 苍穹设计器 |
| step5 | 从 HR 自助工作台-人才搜索发起对比，验证增删移 + 翻页 | 手机 / 移动端预览 |

* * *

## step1 触发技能生成前端控件与后端插件

### 1.1 准备设计稿产物

Figma 选中区块，导出 2x PNG 截图 + `Inspect` 面板的 CSS（存 txt），放一个目录（本例 `figma/`）。  
CSS 必须导，颜色字号字重行高间距才能按原值落地。

**不要照搬的三类值**（本例 `figma/教育经历CSS.txt` 里全都有，括号内是它的实际取值）：

*   **固定 `width`**（如 `343px` / `138px` / `120px`）：那是 Figma 在 390 画板下量的结果。  
    移动端两列宽度由公共外壳按 `flex: 1` 等分，写死会导致列宽不随屏幅变化
*   **固定 `height`**（如 `232px` / `144px`）：高度由内容决定，写死会截断或留白
*   **Frame 层级与 `order` / `z-index`**：Figma 自动生成的结构层，与实现无关

要照搬的是**视觉属性**：颜色、字号、字重、行高、内边距、间距、圆角、边框。

**Figma 的 CSS 导出还有两类失真，照抄会做错**（本例的时间轴圆点与竖虚线都中了，  
其它形态遇到圆形、竖线元素时同理）：

*   **形状信息丢失**：Figma 里的 `Ellipse` 导出只有 `width/height/background`，  
    **没有 `border-radius`** —— 不补 `50%` 就是方块。凡是设计稿里看着是圆的元素都要检查
*   **用旋转模拟的元素**：竖线在 Figma 里常是「横线 + `transform: rotate(90deg)`」，  
    导出就是 `width: 63px; border: 1px dashed; transform: rotate(90deg)`。  
    直接画竖线（`width: 0; border-left: 1px dashed`）更稳，照抄 rotate 会因变换原点错位

### 1.2 触发技能

技能在 `skill/custom-control-full-stack-generic/`，**先按 `skill/README.md` 装到 AI 工具能加载的位置**。

```
/custom-control-full-stack-generic
根据 Figma 导出的图片和 CSS 为我生成自定义控件，严格还原颜色、间距、字体、字重等样式。
控件标识 secDevTalentComparisonEduMob，使用 React + antd。
自定义控件生成在 <你的控件目录> 下，后端插件写在 <你的插件目录> 下。

取数逻辑：
查询实体：<实体标识>
过滤条件：按 employeeIds 批量过滤（一次最多 20 人）
需要字段：<字段清单，基础资料字段说明取 name 还是 id>
排序分组：<排序规则，以及按 employeeId 分组>
派生数据：<需要计算/拼接的字段及规则，如起止年份拼成 2011 - 2014>
多语言模块标识：<你工程的常量或字符串>

请同时阅读 common/doc/人才对比数据展示控件事件规范.md，
按第二节补 update 的 action 分派与 store 的 handleAction，
按第三节落地空数据与超长截断规范，
按第五节接入 pair 切换（拷 shared/、用 PairCompareSection、store 里同步 pairIndex、
destoryed 里 dispose）。
```

不支持斜杠命令的工具，改成「请阅读 skill/custom-control-full-stack-generic/SKILL.md 并严格按其流程执行」+ 同样的需求。

**取数逻辑在这一步就说清，能省掉 step3。** 人才对比的取数有两点和普通卡片不同：  
**按 employeeIds 批量查再在内存分组**（不是单员工查询），  
**查不到数据的员工也要占一列**（下发 `entries: []`，详见 1.6）。

**最后那段（尤其 pair 切换）也必须说。** 技能模板不含增量处理、也不含 pair 机制，  
不说明生成出来的控件**首次加载正常、删人和排序没反应、翻页时本区块不跟着动**。

技能会确认：控件名（`secDevTalentComparisonEduMob`，后面预置 SQL 的 `FSCHEMAID` 必须一致）、  
框架（React + Antd）、ISV（`kingdee`）、MODULE\_ID（`hr`）、目标目录（必须给，技能不猜）。

> `sample/` 里的示例用的是静态模拟数据，只为让 demo 不依赖环境就能跑起来，  
> **不是推荐做法**，你的二开要按上面的模板把真实取数说清。

### 1.3 先把 shared/ 拷进来（移动端特有，第一步做）

移动端 pair 切换靠一组公共文件。**每个自定义控件独立打包、无法跨工程 import**，  
所以这组文件在各控件工程间是「逐字节一致地复制」的关系，不是依赖引用。

**本交付包已自带一份，直接从 `sample/frontend/src/shared/` 整个目录拷到你的工程**  
（与标品各控件工程里的那份内容一致，不用去找标品源码）：

| 文件 | 作用 |
| --- | --- |
| `pairSyncBus.ts` | window 事件通道传输层（频道 `tdcTalentComparison:pairSync`） |
| `usePairSync.ts` | 订阅方 hook，给出 `pairIndex` / `direction` / `slicePair()` |
| `usePairTransition.tsx` | 滑动过渡动效 |
| `PairCompareSection.tsx` | 对比区块通用外壳（切片、动效、锚点、空态、点击浮层全包） |
| `textTooltip.ts` | 触屏下点击省略文本弹完整内容 |

**拷进来就不要改。** 它们是跨工程一致的公共层，改了会与其余控件行为不一致  
（例如缺省态文案是外壳内置的硬编码，不要在自己工程里给它接词条）。

用外壳写业务只需给两个东西：

```
<PairCompareSection<EduPerson>
  moduleKey="secdevedu"          // 与元数据「自定义控件字段标识」一致，用作导航锚点
  className={Style.section}
  title={<div><span>{title}</span></div>}
  fullList={fullList}            // 从 ajaxData.data.list 取
  renderColumn={renderColumn}    // 渲染单个员工列
/>
```

外壳已处理：按 `pairIndex` 切当前两位、动效方向裁决、`data-tdc-anchor` 锚点、  
`scroll-margin-top: 46px`（避开 sticky 导航）、空列表缺省态、触屏点击浮层委托。

### 1.4 前端产出必须自查的四处

技能从离线模板复制，自动改好 `package.json` 的 `name`、`app.config.js` 的三项、  
`variable.less` 变量前缀，并跑 `npm install`。以下四处模板不带，**逐一确认 AI 补上了**：

**（1）`update` 按 action 分派**（`src/index.tsx`）。模板原样是无脑 `setAjaxData(props)`，  
会把增量指令当全量数据覆盖掉：

```
update(props: ReturnDataType) {
  saveConfig(props, this.zustandStore)
  const store = this.zustandStore.useGlobalStore.getState()
  const action = props?.data?.action
  if (action === 'insert' || action === 'remove' || action === 'move') {
    store.handleAction(props.data)   // 增量：合并进已有数据
  }
  else {
    store.setAjaxData(props)         // 全量：整体覆盖
  }
}
```

`src/indexDev.tsx`（ram 开发模式入口）**同样要改**，否则本地调不出增删移的效果。

**（2）store 里补 `handleAction`，且要同步调整 `pairIndex`**（`src/store/useGlobalStore.ts`）。  
移动端比 PC 端多了后半句——列表变了但可视窗口下标没跟着调，翻页后各控件显示的就不是同一对人。  
规则各控件必须一致：

`insert  → pairIndex = 0                         remove  → clampPairIndex(prev, 新列表长度) move    → clampPairIndex(prev, 新列表长度)`

clamp 范围是 `[0, max(0, len - 2)]`，与 `shared/usePairSync.ts` 内的规则一致。  
调整后还要 **只写快照、不广播事件**：

`const bus = getPairSyncBus() const snapshot = bus?.getSnapshot?.() bus?.writeSnapshot?.({   pairIndex: nextPairIndex,   employeeOrder: list.map(item => item.employeeId),   seq: snapshot?.seq ?? 0, })`

用 `publish` 广播会让其余控件误判成「用户翻页」而播放滑动动效——集合变更是 setData 的结果，  
不是翻页，规范里明确要求这时不播动效。

模板的 store 是 `create(set => ...)`，要改成 `create((set, get) => ...)` 才读得到当前状态。  
完整实现见 `sample/frontend/src/store/useGlobalStore.ts`。

漏了（1）（2）的表现：**首次加载正常，删人、拖拽排序后卡片不动**。

**（2.1）`setAjaxData`（全量分支）里也要同步 `employeeOrder` 快照。**  
这条最容易漏，且**只有二开控件会踩**——照抄标品控件的 store 不够。

`shared/usePairSync.ts` 的 `slicePair()` 在 `employeeOrder` 非空时以快照顺序为准关联  
`fullList`，**快照里没有的 `employeeId` 会被直接跳过**。而二开插件新增对比人时推的是  
全量（拿不到差集，只能按缓存顺序推全量，见 1.7-4），走 `setAjaxData` 分支、不经  
`handleAction`，快照就不会更新——`fullList` 里已有新人、快照里没有，新人被整个丢掉：

*   新增对比人后**本区块列数没变、看不到新人**（其它标品区块正常，它们走的是 `insert` 增量）
*   新增后翻到末尾时，本区块比其它区块少一列、横向对不齐

`setAjaxData: (data) => {   const prevList = get().ajaxData?.data?.[LIST_KEY] || []   set({ ajaxData: data })   const nextList = data?.data?.[LIST_KEY] || []   if (!nextList.length) return      const nextPairIndex = nextList.length > prevList.length     ? 0     : clampPairIndex(get().pairIndex, nextList.length)   set({ pairIndex: nextPairIndex })   const bus = getPairSyncBus()   const snapshot = bus?.getSnapshot?.()   bus?.writeSnapshot?.({     pairIndex: nextPairIndex,     employeeOrder: nextList.map(item => item.employeeId),     seq: snapshot?.seq ?? 0,   }) },`

**（3）`destoryed` 里清理 pair 通道**（`src/index.tsx` 与 `src/indexDev.tsx`）：

```
destoryed() {
  getPairSyncBus()?.dispose?.()
  // ... unmount
}
```

漏了的表现是**下次进入对比页时起手就停在最后一对**（沿用上次退出的 `pairIndex`），  
更麻烦的是废弃单例会让通道 A 静默失效，翻页 / 滑动全部无反应。

**（4）超长文本用原生 `title`，不引 antd Tooltip**。触屏没有 hover，  
antd Tooltip 在移动端点不出来；原生 title 点了也没反应，这一层由 `shared/textTooltip.ts`  
的事件委托统一补齐（用了外壳就自动生效）。业务代码只需正常写 `title`：

```
<span>
  {schoolName}
</span>
```

**`title` 必须挂在真正带省略号的那个元素本身**，不能挂到父节点。委托是  
「用 `closest('[title]')` 命中的那个元素量是否溢出」，挂在不截断的父节点上会被判成  
「没截断」而直接不弹。

截断样式三件套与 PC 一致，少一件就不截断：

`.entryText {   width: 100%;   min-width: 0;           overflow: hidden;   text-overflow: ellipsis;   white-space: nowrap; }`

### 1.5 列内布局：`renderColumn` 只管一个人，别跨列对齐

外壳把两列的宽度和切片都包了，`renderColumn(employee)` 的职责只有一个：  
**渲染这一个员工的内容**。列与列之间不要有任何耦合。

三条通用规则，与你的区块是什么形态无关：

**（1）不要试图让两列的第 N 条记录对齐。** 两列是各自独立的容器，同序号记录分属不同父节点，  
高度由各自内容决定（文本长短、是否换行）。想「第 1 条对齐第 1 条」只能靠 JS 量高度，  
会闪动且难维护。

确实需要逐条横向对齐的（如「每人多段卡片」逐段比对），外壳提供了 `rowMode`：  
把内容区改成一个两列 Grid 按行填充，同序号卡片落在同一 grid row，  
靠 Grid 的 `align-items: stretch` 纯 CSS 自动等高。用法是传 `rowMode` 而不是 `renderColumn`：

```
<PairCompareSection<IMyItem, IRecord>
  moduleKey="..."
  title={...}
  fullList={fullList}
  rowMode={{
    getRecords: emp => emp.records,          // 行级过滤在这里做完
    renderCell: (record, emp, rowIndex) => <MyCard record={record} />,
    rowGap: 8,
  }}
/>
```

两点限制：`getRecords` 里要**先剔除该隐藏的记录再返回**，否则空卡片仍占一个 grid 行、  
后续全错位；**有跨记录连续装饰的形态（时间轴连线）不要用 `rowMode`**，  
Grid 会把每条记录放进独立单元格，连线接不上——那类形态按下面（2）（3）做。

**（2）装饰性元素（时间轴、序号、连接线）要跟着记录走，不要抽成独立一列。**  
常见错法：左边一列画时间轴、右边一列排文字，再把时间轴按记录数等分高度——  
各记录高度不等，圆点就会逐条偏离它对应的那条记录。

正确做法是每条记录自成一行，行内自带装饰：

```
{records.map((record, index) => {
  const isLast = index === records.length - 1
  return (
    <div>
      <div>
        <span></span>
        {!isLast && <span></span>}   {/* 末条不再向下延伸 */}
      </div>
      <RecordItem record={record} />
    </div>
  )
})}
```

**（3）记录间距用非末行的 `padding-bottom`，不要用 `row-gap` / `gap`。**  
只要有跨记录连续的装饰（连接线、竖分隔线），`gap` 会在行与行之间留出断口，  
线就断成一截截。把间距放进行高之内，线才能一路连到下一条：

`.row { display: flex; gap: 12px; }          .rowNotLast { padding-bottom: 12px; }`      

没有连续装饰的形态（纯文本列表、标签云）不受此限，正常用 `gap` 即可。

### 1.6 数据契约

`src/types/edu.ts` 是前后端唯一口径：

`export interface EduEntry {   dateRange?: string          schoolName?: string   major?: string              degree?: string } export interface EduPerson {   employeeId: string          entries?: EduEntry[]      } export interface EduCompareData {   empty?: boolean             cardTitle?: string   list?: EduPerson[]          times?: number            }`

四条硬约定（与 PC 端一致）：

*   **`employeeId` 必须是字符串**，类型不一致会静默匹配不上
*   **`entries: []` ≠ 不下发这个人**。查不到数据的人也要占一列（渲染成空内容区），  
    跳过会导致各区块列数不一致、两列错位
*   **`times` 每条都带**（含全量）。内容相同的两次 `setData` 前端识别不到变化，靠它触发 update
*   日期由后端格式化好下发，前端不做日期处理

移动端设计稿没有 211 / 985 标签，所以本例契约里**没有 `tags` 字段**。  
展示什么字段由二开按业务定，两端可以不同，规范只统一 action 协议与主键。

### 1.7 后端插件要解决什么

标品对二开控件的覆盖范围**不全**，这是二开必须写插件的唯一原因：

| 指令 | 覆盖二开控件 | 二开要做什么 |
| --- | --- | --- |
| `remove` / `move` | 【emoji】 | 不用管，标品遍历面板广播（移动端递归） |
| `insert`（新增对比人） | 【emoji】 | **自己补推** |
| 全量（刷新 / F7 选完人） | 【emoji】 | **自己补推** |

漏了插件的表现：**删人、排序正常，新增对比人和刷新时不刷新**。

**不要继承或覆写标品对比插件**：二开不能改标品代码，且会绑死标品版本。  
只依赖平台 `kd.bos.*` 和标品公开的约定（控件标识、pageCache key、事件名）。

**pair 切换不经后端**，插件里不要为 `pairIndex` 加任何逻辑，只管推完整列表。

插件里有四处**错了不会报错、很难查**，必须照做。  
其中前两处**是移动端与 PC 端的实际差异，不能照抄 PC 版插件**：

**（1）父类是 `AbstractMobFormPlugin`**，不是 PC 的 `AbstractFormPlugin`。

**（2）选人 F7 字段是 `multiassignment`，不是 `multiemployee`。** 标品移动端裁掉了  
PC 的多选员工 F7，改为选「组织分配」（`hrpi_assignment`，控件类型 `MulBasedataEdit`），  
再由标品从 `assignment.employee.id` 解析员工。**照抄 PC 的 `multiemployee` 会静默失效**——  
`getControl` 返回 null、监听注册不上，表现是选完人二开控件不刷新且无任何报错：

`@Override public void registerListener(EventObject e) {     super.registerListener(e);     MulBasedataEdit multiAssignment = (MulBasedataEdit) this.getView().getControl("multiassignment");     if (multiAssignment != null) {         multiAssignment.addAfterF7SelectListener(this);     } }`

移动端标品在 `afterF7Select` 里是从 model 取 `multiassignment` 的值解析的，  
**不要判 `ListSelectedRowCollection` 是否为空**（PC 版那样判会误跳过），  
只需读缓存推全量。

**（3）员工顺序读标品的 pageCache**，key 是 `currentEmployeeIds`，别自己另存。

**（4）新增对比人时推全量，不推 `insert` 增量**。事件参数里的 ID 是前端原样传来的，  
标品处理时还会去重和 20 人上限截断，二开插件跑在标品之后拿不到「本次真正新增了谁」的差集——  
推 `insert` 会多出重复列。按缓存顺序推全量既正确又幂等。

要监听的事件名与标品一致：`insertEmployee`（新增）、`refreshData`（刷新）。  
`removeEmployee` / `employeeMoved` 不用管，标品会递归广播。  
页面初次加载也不经 customEvent——标品在自己的 `afterBindData` 里读 `customParams`  
的 `employeeIds` 写缓存，二开在 `afterBindData` 里读缓存即可（排在标品之后就已就绪）。

参考实现：`sample/backend/TalentComparisonMobSecDevExtPlugin.java`。

**做多个二开控件时只写一个插件。** 插件挂在表单上、不挂在控件上，  
`afterBindData` / `customEvent` / `afterF7Select` 都是表单级事件，挂 N 个就触发 N 次、  
查库翻 N 倍。按「控件标识清单 + 按 key 分派」组织，新增控件只加一个 case，  
写法见 PC 端文档 step1.4（两端一致）。

### 1.8 本地验

`npm run mock       npm run dev:ram`   

mock 业务数据放 `mock/data/init.js`，**每个人给不同的 `employeeId`**。  
本例还在 `mock/data/edu.js` 里开了四个接口（`eduFull` / `eduInsert` / `eduRemove` / `eduMove`），  
本地没有平台环境也能自己触发四种指令、验证前端合并逻辑。

至少过：两人并排（列宽是否等分）、某人 `entries: []`（是否渲染成空内容区而非整块消失）、  
条目数不同的人并排、超长学校名（省略号 + **点击**弹完整内容）、一个人都没有（是否保留标题与锚点）、  
多语言、主题色。

连真实环境用 `npm run dev`，测试环境 URL 后拼 `&kdcus_cdn=http://localhost:<DEV_RAM_PORT>`。

* * *

## step2 生成控件方案预置脚本

不做这步，平台里选不到控件。三件事按序：生成 ID → 查库校验 → 写 SQL。

**移动端要另一组 ID、另一个 `FSCHEMAID`，不能拷 PC 端那份 SQL 改名。**  
一个 `FSCHEMAID` 对应一个控件方案，复用会互相顶掉。

**生成 ID**：`FID` 是 19 位正 long，`FPKID` 是 12 位平台风格字符串，  
**必须用雪花算法工具生成，不能手写或复制改写**（编造的数字保证不了唯一，会覆盖数据或主键冲突）。  
技能内置零依赖工具，有 JDK 即可：

`javac -encoding UTF-8 SnowflakeIdGen.java java SnowflakeIdGen 3`

源文件含 UTF-8 中文注释，**编译必须带 `-encoding UTF-8`**，否则默认 GBK 的 Windows 环境会失败；  
且**必须在 `assets/` 下执行**（类无包名）。  
一组 ID 只用于一条记录；多语言表每种语言一行、各自一个 `FPKID`，本例需要 1 个 FID + 2 个 FPKID。

**查库校验**（元数据在 meta 库），两条都返回空才可落库：

`SELECT FID FROM T_META_CTLSCHEMA WHERE FID = <生成的FID> OR FSCHEMAID = '<你的控件标识>'; SELECT FPKID FROM T_META_CTLSCHEMA_L WHERE FPKID IN (<生成的FPKID列表>);`

**写 SQL**：DELETE + INSERT 幂等，完整脚本见  
`sample/backend/kd_secdev_meta_ctlschema_talentcomparisonedumob_preset.sql`。字段对应：

| 字段 | 来源 |
| --- | --- |
| `FSCHEMAID` | 前端 `app.config.js` 的 `APP_NAME`，**必须一致** |
| `FMODULEID` / `FISVID` | `MODULE_ID` / `ISV` |
| `FSCHEMANAME` | 控件方案名称，设计器里选方案时显示的名字 |

`FPKID` 里出现 `+` `/` `=` 是正常的，它是 long 主键的 base64 风格编码（字符集 `0-9A-Z+/=`），  
单引号包裹能正常入库，不用因此重新生成。

文件放到 `<二开工程>/datamodel/.../preinsdata/`，按工程版本号命名规范命名。

* * *

## step3 换真实取数

**step1 已说清取数逻辑、AI 生成了正确实现的话，这步可省。**

技能默认给静态数据（打通链路用）。换真实查询时这几条是硬性的：

*   `QueryServiceHelper.query` 返回 `DynamicObjectCollection`，不是 `DynamicObject[]`
*   基础资料字段不会自动带出动态对象，只能用点路径取值，如 `education.name`
*   多选基础资料按分录路径过滤（如 `xxx.fbasedataid.id`），结果展开成多行
*   **务必用 `QCP.in` 按 employeeIds 一次性批量查，再在内存里按员工分组**。  
    一次最多 20 人，循环查库会放大 20 倍
*   日期格式化用 `HRDateTimeUtils`，禁用 `SimpleDateFormat`（非线程安全）

**按员工组装时，查不到数据的人也要放进 list**，这是空框规范的后端一半：

`for (Long empId : employeeIds) {     Map<String, Object> personData = new HashMap<>(4);     personData.put("employeeId", String.valueOf(empId));     List<Map<String, Object>> entries = new ArrayList<>();     if (!CollectionUtils.isEmpty(eduList)) {              }     personData.put("entries", entries);        list.add(personData); }`

字段级空值用 `-` 或空串占位，不要下发 `null`。

**推送的字段名必须与前端 interface 逐一对应**，写之前先读一遍前端类型定义。  
对不上前端不报错、只显示空白，排查费时间。

多语言：中文串走 `ResManager.loadKDString("教育经历", "<插件类名>_0", <模块标识>)`，  
第 2 个参数是 `类名_序号`（从 0 递增），词条写进工程的 `resources/<工程名>_zh_CN.properties`。  
英文标识符、日志、纯数字编码（如 `211` / `985`）不用处理。

* * *

## step4 扩展移动端对比表单并加入控件

不新建表单，扩展标品的移动端人才对比表单 `hrti_talentcomparison_m`：

![1.0 扩展人才对比移动端页面.png](../images/01004d1949b9a48a42b3a4a4cfe4576d93f3.png)

然后在扩展页面的 `scrollflex` 面板下加控件、改标识、绑方案、挂插件：

![1.1 在扩展页面scrollflex面板下合适位置增加自定义控件，调整控件字段标识名称，设置控件方案，注册表单插件.png](../images/01001b21c5aefefe4c79b7610b635eea0694.png)

四个动作，缺一个就出不来、没数据、不进导航或样式对不上。

### 4.1 控件位置：放 `scrollflex` 下

移动端标品广播 `remove` / `move` 是从 `contentflex` **递归**下发的，  
所以控件嵌套在子容器里也能收到，不像 PC 端必须是 `contentflex` 的直接子级。

但**要进锚点导航就必须放在 `scrollflex` 下**：标品生成导航项时只遍历 `scrollflex` 的子控件  
（标品那 17 个业务展示控件都在这里），挂在 `contentflex` 直接子级的控件收得到广播、  
但不在导航里。按希望的展示位置放即可（区块按元数据顺序排列）。

### 4.2 两个「标识」别搞混，其中一个要三处一致

先分清：

|  | 控件方案标识 | 自定义控件字段标识 |
| --- | --- | --- |
| 本例取值 | `secDevTalentComparisonEduMob` | `secdevedu` |
| 是什么 | 「这个控件位用哪份前端代码渲染」 | 「后端 `getControl` 找哪个控件位」 |
| 定义在哪 | 前端 `app.config.js` 的 `APP_NAME`＝预置 SQL 的 `FSCHEMAID` | 你在扩展页面新增自定义控件时给它起的标识 |
| 怎么用 | 设计器里从控件方案下拉里选（4.3） | 插件 `CONTROL_KEY`、前端 `moduleKey` |
| 唯一性 | 全租户唯一 | 同一张表单内唯一 |

**要三处一致的是后者**（自定义控件字段标识）。

拖入自定义控件后，设计器会给一个默认标识（如 `customcontrolap`），  
**必须手工改成自己的名字**（见 step4 第二张截图里「调整控件字段标识名称」那步）——  
一张表单挂十几个控件，默认值会重名。改好后三处对齐：

| 位置 | 值 |
| --- | --- |
| 扩展元数据上的自定义控件字段标识 | `secdevedu` |
| 后端插件的 `CONTROL_KEY` | `secdevedu` |
| 前端 `EduCompare.tsx` 的 `moduleKey` | `secdevedu` |

前两处不一致：控件能加载但永远空白（插件 `getControl(...)` 拿到 null 直接 return）。  
**第三处不一致：区块能正常显示，但点锚点导航跳不到本区块**——`moduleKey` 就是外壳渲染的  
`data-tdc-anchor` 值，导航靠它 `querySelector` 定位。

### 4.3 绑定控件方案

选 step2 预置的那个，下拉里能看到 `人才对比教育经历二开demo（移动端）` 说明预置 SQL 生效了。  
选不到就回查 `FSCHEMAID` 与 `APP_NAME` 是否一致、SQL 是否执行成功。

### 4.4 样式与配置

*   **控件必须配中文名称**，否则不进 sticky 锚点导航（导航是运行时读面板子控件生成的，无名称的被跳过）
*   **不需要**加 PC 端那段 `$ { overflow: visible; }`：移动端浮层挂在 body 下，不会被宿主容器裁掉
*   外边距参考相邻标品控件（本例区块自身的 `padding: 20px 16px` 与底部虚线已在控件内实现，  
    取自 figma，不用在元数据上另配）

### 4.5 注册插件，且必须排在标品之后

扩展页面插件列表挂上你的插件，**顺序必须在标品对比插件之后**。

插件按元数据顺序串行执行。二开插件靠读标品写入 pageCache 的 `currentEmployeeIds` 拿员工顺序，  
标品在增删移时更新它。**排在标品前面会读到上一次的旧顺序**——表现是新增对比人后，  
你的区块比别人少一列或顺序不对。

**不管做了几个二开控件，这里只挂一个插件**（原因见 1.7）。

* * *

## step5 从对比入口验证

入口在 HR 自助工作台的「人才搜索」：搜到人才后发起人才对比，进移动端对比页看效果。

![2.0 打开HR自助工作台-人才搜索，搜索人才进行人才对比，验证实现效果.png](../images/0100c5dfa66049b640f7836d9ccf980be6be.png)

**六个动作都要测**，走的链路不同、失败原因也不同：

| 动作 | 验证 | 挂了往哪查 |
| --- | --- | --- |
| 页面初次加载 | `afterBindData` 全量推送 | 插件是否注册、`CONTROL_KEY` 是否一致 |
| 添加对比人（F7 选人） | `afterF7Select` + 自己补的全量推送 | `registerListener` 里是否注册了 F7 监听（1.7） |
| 移除某人 | 标品 `remove` 广播 + 前端 `handleAction` | store 有没有 `handleAction`、`update` 有没有分派 |
| 拖拽排序 | 标品 `move` 广播 + 前端 `handleAction` | 同上 |
| **左右滑动 / 点翻页箭头** | window 事件通道（通道 A） | 是否拷了 `shared/`、是否用了 `PairCompareSection` |
| **退出再进对比页** | `destoryed` 里是否 dispose | 起手停在最后一对 → 漏了 `dispose`（1.4-3） |

同时看视觉与交互：

*   翻页后**本区块与其它区块显示的是同一对人**（不同步 → `pairIndex` 没在 `handleAction` 里调整）
*   增删移后**不播滑动动效**，只有主动翻页 / 滑动才播（播了 → 用 `publish` 广播了，应只 `writeSnapshot`）
*   某人没数据时是**空内容区**（不是整块消失、也不是「暂无数据」文案）
*   一个人都没有时**保留标题 + 缺省态插画**，且该模块仍在锚点导航里（消失 → 整块 `return null` 了）
*   长文本**点击**弹出完整内容（点了没反应 → `title` 挂到父节点上了，或没用外壳）
*   时间轴圆点与每条首行文字对齐、虚线连续（断断续续 → 用了 `column-gap`，见 1.5）
*   区块标题进了锚点导航（没进 → 元数据没配中文名称）

有问题继续让 AI 调。**报现象时说清是哪个动作触发的**——增删移走标品广播、  
新增/刷新走二开插件自推、翻页走 window 通道，是三条完全不同的路。

* * *

## 排错对照表

| 现象 | 可能原因 |
| --- | --- |
| 设计器控件方案下拉里没有你的控件 | step2 预置 SQL 没执行，或 `FSCHEMAID` 与 `APP_NAME` 不一致 |
| 区块出来了但永远没数据 | 没挂插件（4.5）；或自定义控件字段标识与插件 `CONTROL_KEY` 不一致（4.2）；或 `CONTROL_KEY` 误填了控件方案标识 |
| **删人、排序没反应**，新增和刷新正常 | 前端缺 `handleAction`，或 `update` 没做 action 分派（1.4） |
| **新增、刷新没反应**，删人排序正常 | 二开插件没补 `insert` / 全量推送（标品清单不含二开控件） |
| F7 选完人不刷新，且无任何报错 | `registerListener` 里没注册 F7 监听；或照抄 PC 注册了 `multiemployee`，移动端应是 `multiassignment`（1.7-2） |
| 区块显示正常、广播也收到，但不在锚点导航里 | 控件挂在 `contentflex` 直接子级，要进导航须放 `scrollflex` 下（4.1）；或没配中文名称 |
| 新增对比人后区块少一列或顺序不对 | 插件顺序排在标品之前，读到旧的 `currentEmployeeIds` |
| 新增对比人后多出一列重复数据 | 按事件参数推了 `insert` 增量；应按缓存顺序推全量 |
| **翻页时本区块不动 / 显示的不是同一对人** | 没接 `usePairSync`（没用外壳）；或 `handleAction` 里没同步调整 `pairIndex` |
| **新增对比人后本区块列数没变、看不到新人**（其它区块正常） | `setAjaxData` 里没同步 `employeeOrder` 快照，新人被 `slicePair` 跳过（1.4-2.1） |
| **删掉最后一个对比人后仍显示残留卡片** | 空态判定读了 `empty` 字段；移除走 `remove` 增量、payload 里没有 `empty`，应按合并后列表长度判空 |
| **增删移后播了滑动动效** | 用 `publish` 广播了集合变更；应只 `writeSnapshot` 不广播 |
| **退出再进，起手就停在最后一对**，且翻页全无反应 | `destoryed` 里没调 `getPairSyncBus()?.dispose?.()` |
| 点锚点导航跳不到本区块 | 前端 `moduleKey` 与元数据控件标识不一致（4.2） |
| 一个人都没有时该模块从导航里消失 | 整块 `return null` 了；移动端要保留标题与锚点（用外壳即自动满足） |
| 某人没数据时整块消失 | 后端把没数据的人跳过了，应下发 `entries: []` |
| 长文本点击没反应 | `title` 挂到了不截断的父节点上；或没用外壳、没调 `attachTextTooltip` |
| 长文本不出省略号 | 文本节点缺 `min-width: 0`（1.4-4） |
| 引了 antd Tooltip，手机上点不出来 | 移动端要用原生 `title` + 点击委托，不用 Tooltip |
| 列宽不随屏幅变化 | 照抄了设计稿固定宽（343 / 138 / 120px），应交给外壳等分 |
| 装饰元素（时间轴圆点 / 序号）逐条偏离它对应的记录 | 把装饰抽成独立一列按份数等分了，应让装饰跟着每条记录走（1.5-2） |
| 连接线 / 竖分隔线断成一截截 | 记录间距用了 `gap`，应改为非末行 `padding-bottom`（1.5-3） |
| 两列的同序号记录横向对不齐 | 默认两列各自独立、不保证对齐；需要逐条对齐要用外壳的 `rowMode`（1.5-1） |
| 圆形元素显示成方块 | figma CSS 没导出 `border-radius`，要自己补 `50%`（1.1） |
| 控件白屏 | 静态资源没部署，或前端运行时报错（看浏览器控制台） |
| 界面显示词条 key 名 | 词条未注册且没做兜底 |
| 隐藏的区块收不到任何推送 | 预期行为，插件推送前判可见性，不可见不查数不推数 |

## 收尾自查

IDE 侧：

*   [ ]  `app.config.js` 的 `APP_NAME` / `ISV` / `MODULE_ID` 已改，`variable.less` 变量前缀已同步
*   [ ]  `src/shared/` 已整个拷入，且**没有改动**
*   [ ]  业务组件用 `PairCompareSection`，`moduleKey` 与元数据控件标识一致
*   [ ]  `index.tsx` **与** `indexDev.tsx` 的 `update` 都按 action 分派
*   [ ]  store 已补 `handleAction`（三种都实现），且同步调整 `pairIndex`（insert→0；remove/move→clamp）
*   [ ]  **`setAjaxData`（全量分支）里也同步了 `employeeOrder` 快照**（漏了新增的人不显示，1.4-2.1）
*   [ ]  `pairIndex` 调整后只 `writeSnapshot`、**没有** `publish` 广播
*   [ ]  空态按合并后列表长度判定，**没有**依赖 `empty` 字段（移除最后一人时不会下发 `empty`）
*   [ ]  `index.tsx` 与 `indexDev.tsx` 的 `destoryed` 都调了 `getPairSyncBus()?.dispose?.()`
*   [ ]  超长文本用原生 `title`，**没有引 antd Tooltip**；`title` 挂在真正带省略号的元素本身
*   [ ]  截断文本有 `min-width: 0`
*   [ ]  列宽交给外壳等分，**没照抄设计稿固定宽**；高度没写死
*   [ ]  `renderColumn` 只渲染单个员工，没有跨列对齐的逻辑；需逐条横向对齐的用 `rowMode`
*   [ ]  装饰性元素（时间轴 / 序号 / 连接线）跟着每条记录走，没抽成独立一列
*   [ ]  有连续装饰时，记录间距用非末行 `padding-bottom` 而非 `gap`（否则线断成一截截）
*   [ ]  figma 未导出的属性已补（如圆形元素的 `border-radius: 50%`）；  
    竖线用 `border-left` 画，没照抄 `rotate(90deg)`
*   [ ]  单人无数据渲染成空内容区（无文案、无写死高度）
*   [ ]  `employeeId` 存在且为字符串，每次推送都带 `times`
*   [ ]  后端对查不到数据的员工仍下发 `entries: []`，没跳过这个人
*   [ ]  插件父类是 `AbstractMobFormPlugin`（不是 PC 的 `AbstractFormPlugin`）
*   [ ]  插件在 `registerListener` 里注册了 **`multiassignment`** 的 F7 监听（不是 `multiemployee`）
*   [ ]  插件读标品 pageCache 的 `currentEmployeeIds`，没自己另存
*   [ ]  插件不继承标品对比插件，也没为 `pairIndex` 加逻辑
*   [ ]  做了多个二开控件的话，**只有一个扩展插件**，控件标识清单 + 按 key 分派
*   [ ]  中文串走 `ResManager.loadKDString`，词条已写进 properties
*   [ ]  控件方案预置 SQL 已生成，ID 已查库校验无冲突，**没有复用 PC 端那份**
*   [ ]  若控件目录有硬编码清单的批量打包脚本，控件名已加入（移动端清单）
*   [ ]  `npm run build` 与 lint 通过

平台侧（扩展 `hrti_talentcomparison_m`）：

*   [ ]  控件放在 `scrollflex` 下（放 `contentflex` 直接子级收得到广播但不进锚点导航）
*   [ ]  **自定义控件字段标识**唯一，且与插件 `CONTROL_KEY`、前端 `moduleKey` **三处一致**
*   [ ]  没把控件方案标识（`FSCHEMAID`）和自定义控件字段标识搞混（4.2）
*   [ ]  控件方案已绑定
*   [ ]  控件配了中文名称（否则不进锚点导航）
*   [ ]  **没有**加 PC 端那段 `$ { overflow: visible; }`（移动端不需要）
*   [ ]  插件已注册，且顺序在标品插件**之后**；**不管几个控件都只挂一个插件**
*   [ ]  加载、添加、移除、拖拽排序、滑动翻页、退出再进 六个动作都验过

[自定义控件部署](https://developer.kingdee.com/knowledge/768125700280322048?specialId=194046086670543360&productLineId=29&isKnowledge=2&lang=zh-CN)
