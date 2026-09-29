---
title: 15分钟内完成一个人才画像移动端卡片开发
category: guide
cloud: 人才发展云
tags: [人才画像, 移动端, 快速开发, 15分钟, 卡片开发, 人才星图, 开发指南, HR自助工作台]
aliases: [移动端人才画像卡片, 画像卡片移动端开发]
author: 金蝶云社区
created: 2026-09-28
updated: 2026-09-28
source: https://vip.kingdee.com/knowledge/892419210474329344
---

# 人才画像卡片二开实操步骤（移动端）

按实际操作顺序记录，以「教育背景」卡片为例。

**本文只讲移动端。** 另有一份 PC 端交付包，两边的卡片、标识、截图不要混用。

配套示例代码在本包 `sample/` 下：`sample/frontend/`（移动端前端控件工程，已剔除  
`node_modules` 与构建产物）、`sample/backend/`（后端插件模板 + 控件方案预置 SQL）。  
设计稿产物示例在 `figma/`。

前后端对接机制看 `doc/数据契约机制.md`（卡片级字段、空态与日期约定、事件规范）。  
具体业务字段以前端 `sample/frontend/src/types/edu.ts` 的类型定义为准，  
**展示什么由二开按业务定**。

阅读约定：`<二开工程>`、`<你的包>`、`<你的控件目录>` 等占位符替换为实际值。

全流程六步，前三步在 IDE 里让 AI 做，后三步在苍穹平台手工配：

| 步骤 | 做什么 | 在哪做 |
| --- | --- | --- |
| step1 | 触发技能，按 Figma 截图 + CSS 生成前端控件与后端插件 | IDE |
| step2 | 生成控件方案预置脚本（移动端独立一组 ID） | IDE |
| step3 | 调整后端取数逻辑（step1 已给全可省） | IDE |
| step4 | 挑一张预置卡片，按模式配控件或配字段 | 苍穹设计器 |
| step5 | 卡片编码加入白名单参数，再在移动端视图配置里加入 | 平台配置页 |
| step6 | 从 HR 自助工作台人才搜索进画像移动端验证 | 平台页面 |

* * *

## 卡片承载方式

**不要一上来就新建卡片表单。** 标品预留了 20 张空白移动端卡片  
`hrti_basecard01_m` ~ `hrti_basecard20_m`，全部留给二开使用，直接挑一张来配即可，  
省掉新建表单、配继承这一串事。

优先级：

1.  **预置卡片 `hrti_basecard01_m` ~ `hrti_basecard20_m`**（首选）
2.  20 张全部用完，再继承 `hrti_profilecardtpl_m` 自建（见附录 B）

预置卡片都继承自 `hrti_profilecardtpl_m`，结构如下：

```
hrti_basecard01_m（继承 hrti_profilecardtpl_m）
└─ flexpanelap
   ├─ title
   │  └─ labelap            卡片标题
   └─ content
      └─ customcontrolap    自定义控件
```

**注意 `customcontrolap` 在 `content` 里面。** 两种模式的配置方式：

| 模式 | 隐藏 | 使用 |
| --- | --- | --- |
| 自定义控件 | `title` | `customcontrolap` 绑控件方案，标题由控件自绘 |
| 苍穹表单 | `customcontrolap` | `labelap` 设标题文案，在 `content` 里加字段 |

预置卡片目前是纯空白（没有预置业务字段），表单模式要自己往 `content` 里加。

预置卡片是**共享池，一张只能被一个二开卡片占用**。动手前先确认哪些还空着，  
建议团队内维护一份占用登记，避免两个人挑到同一张。

**预置卡片默认在移动端视图配置里选不到。** 这 20 张是留给二开的空位，标品用户配置视图时  
不该看到它们，所以标品把它们全部屏蔽了。你配完卡片后，需要把编码加入白名单参数才能选到，  
见 step5.1。这一步漏掉的表现就是「卡片明明配好了，移动端视图配置里却找不到」。

* * *

## step1 触发技能生成前端控件与后端插件

### 1.1 先准备好设计稿产物

Figma 里选中卡片组件，导出两样东西放到一个目录（本例 `figma/`）：

*   **截图**：2x PNG，有数据态即可（本例 `教育背景移动端.png`）
*   **CSS**：右侧面板 `Inspect` → 复制 CSS → 存成 txt（本例 `教育背景移动端CSS.txt`）

空数据态不用导。技能生成的控件工程自带 `Empty` 组件（插画 + 「暂无数据」），  
直接引用即可，不需要按设计稿另画一套。

**CSS 必须导。** 有它才能把颜色、字号、字重、行高、字距、圆角、渐变、间距按原值落地，  
不用靠肉眼估。哪些照搬、哪些不照搬：

| 照搬 | 不照搬 |
| --- | --- |
| 色值、字号、字重、行高、letter-spacing | `position: absolute` 及 `left/top` |
| 圆角、阴影、边框、渐变 | 固定 `width` / `height` |
| padding、gap | Figma 自动生成的 Frame 层级 |

Figma 稿是定宽绝对定位（本例卡片 343px），要改成流式布局：宽度由平台容器决定，  
内边距按设计稿数值换算。设计稿里 `display: none` 的元素（本例「历史记录」按钮、  
「考察组意见」描述行）不用实现，但可以保留样式类供二开按业务打开。

### 1.2 触发技能

技能在本包 `skill/custom-control-full-stack-generic/`，**先按 `skill/README.md`  
装到你的 AI 编码工具能加载到的位置**，否则下面的命令用不了。

把这些信息一次给全，AI 就不用反复问。支持斜杠命令的工具：

```
/custom-control-full-stack-generic
根据 Figma 导出的图片和 CSS 为我生成自定义控件，严格还原颜色、间距、字体、字重等样式。
控件标识 secDevTalentProfileEduMob，使用 React + antd。
前端控件生成在 <你的控件目录> 下，后端插件写在 <你的后端目录> 下。
```

不支持斜杠命令的工具，把技能文件当上下文引用，效果一样：

```
请阅读 skill/custom-control-full-stack-generic/SKILL.md 并严格按其中的流程执行。
根据 Figma 导出的图片和 CSS 为我生成自定义控件，严格还原颜色、间距、字体、字重等样式。
控件标识 secDevTalentProfileEduMob，使用 React + antd。
```

技能会先确认这几项（已给的不重复问）：

| 项 | 本例取值 | 说明 |
| --- | --- | --- |
| 控件名 | `secDevTalentProfileEduMob` | 后面预置 SQL 的 `FSCHEMAID` 必须与它完全一致；带 `Mob` 后缀标明是移动端控件 |
| 前缀约定 | `secDev`（demo） | 团队有云产品前缀规范则按规范 |
| 框架 | React + Antd |  |
| ISV | `kingdee` |  |
| MODULE\_ID | `hr` |  |
| 目标目录 | 你指定的目录 | 技能不会猜，必须给 |

**取数逻辑在这一步就一起说清，能省掉 step3。** 例如实体标识、过滤字段、需要哪些字段、  
排序分组规则、派生数据怎么算。不说的话技能会先用静态数据打通链路。

### 1.3 技能产出

前端工程（自动完成）：

*   从离线模板复制，不是拷现有项目改
*   改 `package.json` 的 `name`
*   改 `app.config.js` 的 `APP_NAME` / `ISV` / `MODULE_ID`
*   把 `src/styles/variable.less` 里 `react_demo` 变量前缀换成 APP\_NAME（本例 `secDevTalentProfileEduMob`）
*   `npm install`、拷 `CLAUDE.md`

后端插件：

*   `AbstractExtPortraitCardPlugin`：二开自己的基类（公共基类，在 `common/sample/backend/` 下）
*   `EduBackgroundCardPluginMob`：移动端卡片插件（本包 `sample/backend/`）

**注意后端插件不继承标品的 `AbstractPortraitCardPlugin`。** 原因：标品类随版本演进，  
签名或行为变化会直接打断二开；跨工程继承会把二开绑死在标品版本上。  
基类只依赖平台 `kd.bos.*` 和标品公开的约定（控件标识、页面参数 key），标品升级不影响二开编译。

基类提供的能力：

| 能力 | 说明 |
| --- | --- |
| `preOpenForm` | 设 VIEW 状态，卡片是只读展示 |
| `getEmployeeId()` | 先取自身页面参数，再取父视图参数，参数 key 是 `employee` |
| `pushData(Map)` | 推数据，自动补 `times` 时间戳与 `cardTitle` |
| `pushEmpty()` | 推 `{ empty: true }` |
| `getCardTitle()` | 解析用户在画像视图配置里改过的标题 |

`times` 是必要的：内容相同的两次 `setData` 前端可能识别不到变化，补时间戳保证每次触发 update。

**基类在二开工程里拷一份就够，各卡片插件都继承它，不要每张卡片另拷一份改名。**

### 1.4 如果只生成了前端、没生成后端插件

技能理论上会主动问「是否需要生成对应的后端 Java 插件」，实际不一定问到。  
这时候补一次即可，不用重跑整个技能。

**关键是让 AI 继承二开模板基类，而不是继承标品的 `AbstractPortraitCardPlugin`。**  
不明确说的话，AI 看到工作区里有标品基类，很可能直接继承过去——那样二开就被绑死在  
标品版本上了。

先把 `common/sample/backend/AbstractExtPortraitCardPlugin.java` 拷进你的二开工程、  
改好包名，然后让 AI 照它写卡片插件。提问时把继承的基类、插件目录、查询实体、  
过滤字段、需要字段、排序分组、派生数据、多语言模块标识一次给全，并强调  
**先读前端 `src/types/edu.ts` 再写推送逻辑**。字段名对不上是这里最容易出的错，  
前端不报错、只是显示空白，排查费时间。

### 1.5 技能的两个提示

技能跑完会给出：

1.  **提示要放资源文件**：识别到设计稿里有图标 / 背景图等资源，让你放进对应目录。  
    本例移动端卡片用纯 CSS 画时间轴圆点和虚线，无外部图标依赖，可跳过。
2.  **问询是否生成控件方案预置脚本**：这步不做，平台里选不到控件。答「是」进 step2。

### 1.6 顺手确认数据契约

契约是前后端唯一的对接口径，移动端 `src/types/edu.ts`：

`export interface EduItem {   id: string             period?: string        school: string         degree?: string        major?: string       }  export interface EduCardData {   empty?: boolean        cardTitle?: string     list?: EduItem[] }`

字段按移动端业务定。三个约定值得固化：

*   `empty` 让后端判定，前端不猜「列表为 0 是没数据还是没查」
*   `period` 由后端格式化好，日期格式化涉及多语言和平台日期格式，放后端更稳
*   `cardTitle` 可缺省，用户改过标题才下发

### 1.7 本地验一遍

`npm run mock       npm run dev:ram`   

mock 业务数据单独放 `mock/data/init.js`，由 `mock/data/index.js` 组装进 `initMock`，  
不要塞公共文件。至少过这几种：有数据、空数据（`empty: true`）、条目溢出（看列表是否内部滚动、  
卡片是否没被撑破）、超长文本、多语言切换、主题色切换。

移动端还要重点看窄屏表现：时间轴圆点与虚线对齐、胶囊标签换行、学校名超长省略号。

需要连真实环境时用 `npm run dev`，在测试环境页面 URL 后拼  
`&kdcus_cdn=http://localhost:<DEV_RAM_PORT>`，平台会来本地拉控件资源。  
端口冲突改 `app.config.js` 的 `DEV_RAM_PORT` / `DEV_CACHE_PORT` / `MOCK_PORT`。

* * *

## step2 生成控件方案预置脚本

指定输出目录让 AI 生成。三件事按序做：生成 ID → 查库校验 → 写 SQL。

**每个控件都要重新生成一组 ID，不要拷用其他控件的 SQL 改名。**  
一个 `FSCHEMAID` 对应一个控件方案，复用会互相顶掉。

**生成 ID**：`FID` 是 19 位正 long，`FPKID` 是 12 位平台风格字符串，  
**必须用雪花算法工具生成，不能手写或复制改写**。技能内置了零依赖工具，有 JDK 即可跑：

`javac -encoding UTF-8 SnowflakeIdGen.java java SnowflakeIdGen 3`

源文件含 UTF-8 中文注释，编译必须带 `-encoding UTF-8`，否则默认 GBK 的 Windows 环境会失败。  
一组 ID 只用于一条记录，多语言表每种语言一行、各自一个 `FPKID`，  
本例需要 1 个 FID + 2 个 FPKID。

**查库校验**：概率上不冲突，仍要写入前查一次。元数据类预置数据在 meta 库：

`SELECT FID FROM T_META_CTLSCHEMA  WHERE FID = <生成的FID> OR FSCHEMAID = 'secDevTalentProfileEduMob'; SELECT FPKID FROM T_META_CTLSCHEMA_L  WHERE FPKID IN (<生成的FPKID列表>);`

除主键外还要校验业务唯一键 `FSCHEMAID` 不与已有控件重复。两条都返回空才可落库。

**写 SQL**：DELETE + INSERT 幂等，可重复执行。本例产物见  
`sample/backend/kd_secdev_meta_ctlschema_talentprofileedumob_preset.sql`：

`DELETE FROM T_META_CTLSCHEMA_L WHERE FID = 1543891644498055168; DELETE FROM T_META_CTLSCHEMA WHERE FID = 1543891644498055168;  DELETE FROM T_META_CTLSCHEMA_L WHERE FID IN (   SELECT FID FROM T_META_CTLSCHEMA WHERE FSCHEMAID = 'secDevTalentProfileEduMob'); DELETE FROM T_META_CTLSCHEMA WHERE FSCHEMAID = 'secDevTalentProfileEduMob';  INSERT INTO T_META_CTLSCHEMA     (FID, FSCHEMAID, FPREVIEWIMG, FVERSION, FATTACHMENTCOUNT, FISVID, FMODULEID) VALUES     (1543891644498055168, 'secDevTalentProfileEduMob', ' ', '1', 0, 'kingdee', 'hr');  INSERT INTO T_META_CTLSCHEMA_L (FPKID, FID, FLOCALEID, FSCHEMANAME) VALUES ('4XPPL0+WEP7H', 1543891644498055168, 'zh_CN', '人才画像教育背景二开demo（移动端）');  INSERT INTO T_META_CTLSCHEMA_L (FPKID, FID, FLOCALEID, FSCHEMANAME) VALUES ('4XPPL0+WEP7J', 1543891644498055168, 'zh_TW', '人才畫像教育背景二開demo（移動端）');`

字段对应：

| 字段 | 来源 |
| --- | --- |
| `FSCHEMAID` | 前端 `app.config.js` 的 `APP_NAME`，**必须一致** |
| `FMODULEID` | `MODULE_ID` |
| `FISVID` | `ISV` |
| `FSCHEMANAME` | 控件方案名称，即设计器里选控件时显示的名字 |

`FPKID` 里出现 `+` `/` `=` 是正常的，它是 long 主键的 base64 风格编码，  
字符集 `0-9A-Z+/=`，单引号包裹能正常入库，不用因此重新生成。

文件最终放到 `<二开工程>/datamodel/.../preinsdata/`，按工程版本号命名规范命名。

* * *

## step3 调整后端取数逻辑

**step1 已经把取数逻辑说清、AI 生成了正确实现的话，这步可省。**

技能默认给的是静态数据（打通链路用），替换为真实查询时这几条是硬性的：

*   `QueryServiceHelper.query` 返回 `DynamicObjectCollection`，不是 `DynamicObject[]`
*   基础资料字段不会自动带出动态对象，只能用点路径取值，如 `education.name`
*   多选基础资料按分录路径过滤（如 `xxx.fbasedataid.id`），结果会展开成多行，  
    分组时直接取扁平化路径值
*   需要按多条记录分别查关联数据时，先收集 ID 用 `QCP.in` 一次性批量查，  
    再在内存里分组，**不要循环查库**
*   日期格式化用 `HRDateTimeUtils`，禁用 `SimpleDateFormat`（非线程安全）

移动端插件主流程（`EduBackgroundCardPluginMob`）：

`public class EduBackgroundCardPluginMob extends AbstractExtPortraitCardPlugin {      @Override     public void afterBindData(EventObject e) {         super.afterBindData(e);          Long employeeId = getEmployeeId();         if (employeeId == null) {             logger.info("EduBackgroundCardPluginMob: employeeId not found, push empty.");             pushEmpty();             return;         }          List<Map<String, Object>> list = queryEduList(employeeId);         if (list.isEmpty()) {             pushEmpty();             return;         }          Map<String, Object> data = new HashMap<>(4);         data.put("list", list);                                    data.put("cardTitle", ResManager.loadKDString("教育背景二开demo",                 "EduBackgroundCardPluginMob_0", "<你工程的多语言模块标识>"));         pushData(data);     } }`

**推送的 Map 字段名必须与前端 interface 逐一对应**，写之前先把前端类型定义读一遍。  
移动端字段是 `id / period / school / degree / major`。

多语言：所有面向用户的中文串走 `ResManager.loadKDString("教育背景二开demo", "EduBackgroundCardPluginMob_0", "<你工程的模块标识>")`，第 2 个参数是 `类名_序号`，  
序号从 0 递增。每个词条都要写进工程的 `resources/<工程名>_zh_CN.properties`：

```
EduBackgroundCardPluginMob_0=教育背景二开demo
```

英文标识符、日志、纯数字编码不需要处理。业务模拟数据（学校 / 专业）也不需要。

* * *

## step4 在预置卡片上配置

进苍穹设计器，打开一张预置卡片改它，**不新建表单**。

### 4.1 挑一张未被占用的预置卡片

从 `hrti_basecard01_m` ~ `hrti_basecard20_m` 里挑一张还没人用的。挑好后按 4.2 或 4.3  
二选一配置，两种模式不能混用。

预置卡片已经继承好 `hrti_profilecardtpl_m`，继承路径满足移动端视图配置的筛选条件，  
不需要另配继承关系。

### 4.2 自定义控件模式

适合设计稿复杂、要 100% 还原 UI 的卡片。三个动作：

**① 隐藏 `title`。** 模板自带标题区，控件内部已经画了标题，两个会重复，把 `title` 隐藏掉。

标题文案统一由控件内部渲染：用户在移动端画像视图配置里改过标题就用配置值  
（基类 `getCardTitle()` 解析后下发），没改就用前端词条兜底。

**② 给 `customcontrolap` 绑控件方案。** 控件方案选 step2 预置的 `secDevTalentProfileEduMob`，  
下拉里能看到「人才画像教育背景二开demo（移动端）」说明预置 SQL 生效了。  
选不到就回查 `FSCHEMAID` 与 `APP_NAME` 是否一致、SQL 是否执行成功。

`customcontrolap` 是预置卡片自带的（在 `content` 里），标识保持默认不要改，  
改了插件里要覆盖 `getControlKey()`。

![1.2 隐藏标题，给自定义控件绑定控件方案，注册表单插件.png](../images/0100cfb16ac6db5e4afc96c7fe451f477b67.png)

**③ 控件高度撑满。** 让控件宿主容器撑满卡片，前端侧 `.card` 用 `height: 100%`  
（不是 `min-height: 100%`，百分比基准取不到值会退化成内容高度），  
`overflow: hidden` 保证圆角不被内层滚动条切直角，列表区  
`flex: 1 1 auto` + `min-height: 0` + `overflow-y: auto` 自己滚动。本例前端工程已写好。

### 4.3 苍穹表单模式

适合常规字段罗列、表格、分录的卡片，不写前端控件，step1 / step2 整个跳过。两个动作：

**① 隐藏 `customcontrolap`。** 这条路不用自定义控件，把它设为不可见。

**② `labelap` 设标题、在 `content` 里加字段。** `title` 与 `labelap` 保持可见，  
标题文案写在 `labelap` 上。往 `content` 里拖标准控件，和开发普通动态表单完全一样：

*   文本/数字/日期字段：展示单值
*   分录（EntryGrid）：展示列表数据，例如多条教育经历
*   Flex 容器：分组、控制排列
*   标签（Label）：静态文案

要点：

*   字段标识自己定，后端插件里靠标识 `setValue`，两边保持一致
*   卡片是只读展示，字段设成不可编辑（插件里 `preOpenForm` 设 VIEW 状态也会统一压成只读）
*   移动端是窄屏，字段不要横向铺太多，优先纵向排列

![1.3 苍穹表单插件模式，则继承hrti\_profilecardtpl\_m创建，隐藏自定义控件，设置卡片标题lavelap，在content下增加苍穹表单字段.png](../images/0100558207e390c84044b569e2a8fe75a0e6.png)

插件写法与表单模式特有的注意点见附录 A。

### 4.4 注册后端插件（两种模式都要）

在卡片表单的插件列表里挂上你的插件（本例 `EduBackgroundCardPluginMob`）。

**这步漏掉的表现是：卡片框架能出来，但控件模式永远空态、表单模式字段全空**，  
因为没人给它推数据。

* * *

## step5 放开白名单并加入移动端画像视图

### 5.1 把卡片编码加入白名单参数

预置卡片默认全部被屏蔽，先放开你用的这张，否则 5.2 里选不到。

改人才发展开发参数配置（元数据标识 `tdcs_cfgparam`）里的这条参数：

| 项 | 值 |
| --- | --- |
| 参数编码 | `hrti_portraitcard_whitelist_m` |
| 参数名称 | 人才画像移动端卡片白名单 |
| 取值 | 你用的预置卡片编码，多个用英文逗号分隔 |

例如你用了 `hrti_basecard01_m` 和 `hrti_basecard03_m`，就填：

```
hrti_basecard01_m,hrti_basecard03_m
```

标品预置为空串，即所有预置卡片都不可选。**这个参数没有菜单入口**，  
直接改数据库或用查询分析器更新，注意只追加你自己的编码，不要覆盖别人已加的。

移动端用带 `_m` 后缀的参数编码，填的卡片编码也要带 `_m`。

白名单只控制配置页的可选范围。已经配进视图的卡片，即使后来被移出白名单，  
画像移动端仍会照常渲染，不会突然消失。

自建卡片（附录 B）不受白名单限制，不用加。

### 5.2 移动端画像视图配置增加二开卡片

进画像视图配置，切到**移动端视图配置**，把卡片加进视图，上下移到合适位置，  
点确认返回画像视图配置页，保存。

![2.1 画像视图配置-移动端视图配置，添加卡片，上下移卡片到合适位置，点确认按钮返回画像视图配置页，保存画像视图配置.png](../images/0100aed43bb3f94c442e9f351b06dcb317bc.png)  
**这个页面里所有卡片都显示「暂无数据」是正常的**，包括标品卡片。  
配置页只负责编排卡片位置和标题，不会传 `employee` 参数给卡片，  
插件取不到 employeeId 就走 `pushEmpty()`。判断方法：看旁边的标品卡片，  
它们也空 → 正常配置态；只有你的卡片空 → 才是你的问题。

* * *

## step6 从 HR 自助工作台验证实际效果

打开 HR 自助工作台的**人才搜索**，搜索人才，点击人才卡片超链（姓名、工号）  
打开人才画像移动端，查看卡片实现效果。

![3.1 打开HR自助工作台-人才搜索，搜索人才，点击人才卡片超链（姓名、工号）打开人才画像移动端，查看实现的效果.png](../images/01005d929a1d4abe48b7b826769bd59aaaf9.png)

这时页面带 `employee` 参数，插件能取到 employeeId，卡片渲染真实数据。  
到这里整条链路走完。

效果不理想时，把画像移动端里卡片的截图交给 AI 调样式，AI 按截图批注调整前端代码：

![3.2 检查并让AI调整样式.png](../images/01001acea7ddf2254fbc9842ddd54497c9ad.png)

调整后：

![3.3 调整样式后.png](../images/010009d65ad798e245038d4259d0a3bfe187.png)

* * *

## 排错对照表

| 现象 | 可能原因 |
| --- | --- |
| 设计器控件方案下拉里没有你的控件 | step2 预置 SQL 没执行，或 `FSCHEMAID` 与 `APP_NAME` 不一致，或拷了别的控件的 SQL 没换 ID |
| 预置卡片在移动端视图配置里选不到 | 编码没加入白名单参数 `hrti_portraitcard_whitelist_m`（见 step5.1）；或那张已被其他二开卡片占用，换一张 |
| 移动端视图配置里选不到这张卡片 | 自建卡片没继承 `hrti_profilecardtpl_m`（预置卡片不会有这问题） |
| 卡片框架出来了但永远空态 | step4.4 没挂插件；或控件标识不是 `customcontrolap` 且插件没覆盖 `getControlKey()`；或后端插件根本没生成（见 step1.4） |
| 二开插件继承了标品基类 | 生成插件时没点明继承二开模板基类，见 step1.4 |
| 配置页里所有卡片都空态 | 正常，配置页不传 `employee`（对照标品卡片是否也空） |
| 卡片里出现两个标题 | 控件模式没隐藏 `title`，模板标题还露着 |
| 表单模式卡片没有标题 | `labelap` 没设标题文案，或误把 `title` 隐藏了 |
| 时间轴虚线间距不对 / 圆点错位 | 前端时间轴样式，按截图批注让 AI 调 `.line` / `.axis` |
| 卡片白屏 | 控件静态资源没部署，或前端运行时报错（看浏览器控制台） |
| 前端收不到 `setData` | 推送字段名与前端 interface 不匹配；或 `getEmployeeId()` 返回 null 走了空态 |
| 界面显示词条 key 名 | 词条未注册且没做 FALLBACK 兜底 |

## 收尾自查

IDE 侧：

*   [ ]  `app.config.js` 的 `APP_NAME`(`secDevTalentProfileEduMob`) / `ISV` / `MODULE_ID` 已改，`variable.less` 变量前缀已同步
*   [ ]  空态、溢出滚动、超长文本、多语言、主题色、窄屏时间轴都验过
*   [ ]  `npm run build` 与 lint 通过
*   [ ]  后端插件不继承标品类，推送字段名与前端 interface 逐一对应
*   [ ]  中文串走 `ResManager.loadKDString`，词条已写进 properties
*   [ ]  控件方案预置 SQL 已生成，用的是独立的一组 ID，已查库校验无冲突

平台侧（通用）：

*   [ ]  用的是预置卡片 `hrti_basecard01_m` ~ `hrti_basecard20_m`，且这张没被别人占用
*   [ ]  卡片编码已加入白名单参数 `hrti_portraitcard_whitelist_m`
*   [ ]  后端插件 `EduBackgroundCardPluginMob` 已注册
*   [ ]  卡片已加入移动端画像视图，自助工作台进画像移动端数据正常

自定义控件模式追加：

*   [ ]  `title` 已隐藏
*   [ ]  `customcontrolap` 已绑定控件方案 `secDevTalentProfileEduMob`
*   [ ]  控件高度撑满，前端 `.card` 用的是 `height: 100%`

苍穹表单模式追加：

*   [ ]  `customcontrolap` 已隐藏
*   [ ]  `labelap` 已设标题文案
*   [ ]  字段加在 `content` 里，标识与插件 `setValue` 一致

* * *

## 附录 A 苍穹表单模式补充说明

正文以自定义控件模式为主线。如果卡片就是常规的字段罗列、表格、分录，  
**没必要写前端控件**，用苍穹表单模式更快：不用建前端工程、不用打包部署、  
不用预置控件方案，step1 / step2 整个跳过。

两种模式的取舍：

|  | 自定义控件 | 苍穹表单 |
| --- | --- | --- |
| UI 来源 | 前端代码，可 100% 还原设计稿 | 标准控件拖拉拽，受平台样式约束 |
| 前端工程 | 要建、要打包部署 | 不要 |
| 控件方案预置 SQL | 要 | 不要 |
| 数据下发方式 | `CustomControl.setData()` 推 JSON | `getModel().setValue()` 写字段值 |
| 适合场景 | 设计稿复杂、有自定义交互/图表 | 常规字段展示、列表、分录 |

**平台侧配置见 step4.3**（隐藏 `customcontrolap`、`labelap` 设标题、在 `content` 里加字段），  
这里只讲插件写法与这条路特有的坑。

### A.1 开发插件（继承二开模板插件）

插件仍然**继承你自己工程的 `AbstractExtPortraitCardPlugin`**，不继承标品类，原因同正文。  
基类 `getEmployeeId()`、`preOpenForm` 照用；`pushData()` / `pushEmpty()` 是给自定义控件  
推 JSON 用的，这条路不需要，改成往表单字段写值。

`public class EduBackgroundFormCardPluginMob extends AbstractExtPortraitCardPlugin {           private static final String ENTRY_EDU = "entryentity";      @Override     public void afterBindData(EventObject e) {         super.afterBindData(e);          Long employeeId = getEmployeeId();         if (employeeId == null) {             logger.info("EduBackgroundFormCardPluginMob: employeeId not found.");             return;         }          List<Map<String, Object>> list = queryEduList(employeeId);         if (list.isEmpty()) {             getView().setVisible(Boolean.FALSE, ENTRY_EDU);             return;         }          fillEntry(list);     }           private void fillEntry(List<Map<String, Object>> list) {         AbstractFormDataModel model = (AbstractFormDataModel) getModel();         model.beginInit();         model.deleteEntryData(ENTRY_EDU);         model.batchCreateNewEntryRow(ENTRY_EDU, list.size());         for (int i = 0; i < list.size(); i++) {             Map<String, Object> item = list.get(i);             model.setValue("school", item.get("school"), i);             model.setValue("degree", item.get("degree"), i);             model.setValue("major", item.get("major"), i);             model.setValue("period", item.get("period"), i);         }         model.endInit();         getView().updateView(ENTRY_EDU);     } }`

几个和自定义控件不同的注意点：

*   **批量写分录**要包在 `beginInit()` / `endInit()` 里，最后 `updateView` 一次，  
    逐行 `setValue` 触发的界面刷新会很慢
*   **空数据**没有前端 `empty` 约定，自己决定表现：隐藏内容区、显示「暂无数据」Label，  
    或让卡片不展示
*   **多语言**同样走 `ResManager.loadKDString`，词条写进工程的 properties
*   查数规范与正文 step3 完全一致

### A.2 后续步骤同正文

*   **绑定插件**：在卡片表单的插件列表里注册这个插件（step4.4），  
    漏了的表现是卡片能出来但字段全空。
*   **step5 放开白名单并加入移动端画像视图**：先把卡片编码加入  
    `hrti_portraitcard_whitelist_m`（表单模式同样受白名单限制），  
    再切到移动端视图配置把卡片加进视图。配置页不传 `employee`，卡片没数据是正常的。
*   **step6 从自助工作台人才搜索验证**：进画像移动端，带 `employee` 参数，渲染真实数据。

### A.3 本附录的自查

*   [ ]  `customcontrolap` 已隐藏，`labelap` 已设标题文案
*   [ ]  字段加在 `content` 里，标识与插件里 `setValue` 用的一致
*   [ ]  插件继承自己工程的 `AbstractExtPortraitCardPlugin`，不是标品类
*   [ ]  分录批量写包在 `beginInit` / `endInit` 里
*   [ ]  空数据有明确表现
*   [ ]  中文串走 `ResManager.loadKDString`，词条已写进 properties
*   [ ]  插件已在卡片表单上注册
*   [ ]  卡片已加入移动端画像视图，画像移动端里数据正常

* * *

## 附录 B 预置卡片用完后自建

20 张 `hrti_basecard01_m` ~ `hrti_basecard20_m` 全部占满之后，才需要自己建卡片表单。

在 hrti 扩展应用下新建**移动端**动态表单（`MobileFormModel`），  
继承 `hrti_profilecardtpl_m`（带 `_m` 后缀的移动端模板）。  
继承它就自动满足了画像卡片的两个前提：继承路径里含小部件基类（移动端视图配置页才筛得到）、  
卡片外框样式与标品一致。

![继承 hrti\_profilecardtpl\_m 创建二开人才画像移动端卡片](../screenshot/1.1%20%E7%BB%A7%E6%89%BFhrti_profilecardtpl_m%E5%88%9B%E5%BB%BA%E4%BA%8C%E5%BC%80%E4%BA%BA%E6%89%8D%E7%94%BB%E5%83%8F%E7%A7%BB%E5%8A%A8%E7%AB%AF%E5%8D%A1%E7%89%87.png)

继承后的结构、两种模式的配置方式与预置卡片完全相同，按 step4.2 / step4.3 配即可。  
[自定义控件部署](https://developer.kingdee.com/knowledge/768125700280322048?specialId=194046086670543360&productLineId=29&isKnowledge=2&lang=zh-CN)
