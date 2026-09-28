---
title: 15分钟内完成一个人才画像PC端卡片开发
category: guide
cloud: 人才发展云
tags: [人才画像, 画像卡片, 二开, PC端, 自定义控件, 苍穹表单, 控件方案, 前端控件, 后端插件, 开发指南]
aliases: [人才画像卡片二开, 画像卡片自定义开发, PC端人才画像卡片]
author: 金蝶云社区
created: 2026-08-20
updated: 2026-09-28
source: https://vip.kingdee.com/knowledge/878311629627593472
---

# 人才画像卡片二开实操步骤（PC 端）

按实际操作顺序记录，以「教育背景」卡片为例。

**本文只讲 PC 端。** 移动端卡片模板、视图配置入口、前端布局都不一样，  
另有一份移动端交付包，不要拿本文的截图和标识去配移动端。

配套示例代码在本包 `sample/` 下：`sample/frontend/`（前端控件工程，已剔除  
`node_modules` 与构建产物）、`sample/backend/`（后端插件模板 + 控件方案预置 SQL）。  
设计稿产物示例在 `figma/`。

前后端对接机制看 `doc/数据契约机制.md`（两端共用：卡片级字段、空态与日期约定、事件规范）。  
具体业务字段以前端 `sample/frontend/src/types/edu.ts` 的类型定义为准。

阅读约定：`<二开工程>`、`<你的包>`、`<你的控件目录>` 等占位符替换为实际值。

全流程六步，前三步在 IDE 里让 AI 做，后三步在苍穹平台手工配：

| 步骤 | 做什么 | 在哪做 |
| --- | --- | --- |
| step1 | 触发技能，按 Figma 截图 + CSS 生成前端控件与后端插件 | IDE |
| step2 | 放入图标让 AI 替换，生成控件方案预置脚本 | IDE |
| step3 | 调整后端取数逻辑（step1 已给全可省） | IDE |
| step4 | 挑一张预置卡片，按模式配控件或配字段 | 苍穹设计器 |
| step5 | 卡片编码加入白名单参数，再在画像视图配置里加入 | 平台配置页 |
| step6 | 从画像列表进员工画像验证 | 平台页面 |

* * *

## 卡片承载方式

**不要一上来就新建卡片表单。** 标品预留了 20 张空白卡片  
`hrti_basecard01` ~ `hrti_basecard20`，全部留给二开使用，直接挑一张来配即可，  
省掉新建表单、配继承、调外框样式这一串事。

优先级：

1.  **预置卡片 `hrti_basecard01` ~ `hrti_basecard20`**（首选）
2.  20 张全部用完，再继承 `hrti_profilecardtpl` 自建（见附录 C）

预置卡片都继承自 `hrti_profilecardtpl`，结构如下：

```
hrti_basecard01（继承 hrti_profilecardtpl）
├─ talentprofcardmark        卡片标记面板 ← 自定义控件模式隐藏这一层
│  ├─ flexpanelap            标题区（title 标题 + vectorap 图标）
│  └─ content                内容区 ← 苍穹表单模式在这里加字段
└─ customcontrolap           自定义控件 ← 苍穹表单模式隐藏它
```

**注意 `customcontrolap` 与 `talentprofcardmark` 是兄弟节点，不在 `content` 里面。**  
两种模式就是二选一地隐藏其中一支：

| 模式 | 隐藏 | 使用 |
| --- | --- | --- |
| 自定义控件 | `talentprofcardmark`（标题区一并隐藏） | `customcontrolap` 绑控件方案 |
| 苍穹表单 | `customcontrolap` | 保留标题区，在 `content` 里加字段 |

`talentprofcardmark` 的显隐同时是**承载页识别卡片模式的依据**，  
外壳阴影与圆角由 `hrti_workbench` 的 `gridcontainerap` 自定义样式自动切换，  
二开不需要写任何样式。这也是为什么控件模式必须隐藏整个 `talentprofcardmark`，  
只隐藏 `content` 会被判成表单模式、套上两层阴影。

预置卡片是**共享池，一张只能被一个二开卡片占用**。动手前先确认哪些还空着，  
建议团队内维护一份占用登记，避免两个人挑到同一张。

**预置卡片默认在画像视图配置里选不到。** 这 20 张是留给二开的空位，标品用户配置视图时  
不该看到它们，所以标品把它们全部屏蔽了。你配完卡片后，需要把编码加入白名单参数才能选到，  
见 step5.1。这一步漏掉的表现就是「卡片明明配好了，画像视图配置里却找不到」。

* * *

## step1 触发技能生成前端控件与后端插件

### 1.1 先准备好设计稿产物

Figma 里选中卡片组件，导出两样东西放到一个目录（本例 `figma/`）：

*   **截图**：2x PNG，有数据态即可
*   **CSS**：右侧面板 `Inspect` → 复制 CSS → 存成 txt

空数据态不用导。技能生成的控件工程自带 `Empty` 组件（插画 + 「暂无数据」），  
直接引用即可，不需要按设计稿另画一套。

**CSS 必须导。** 有它才能把颜色、字号、字重、行高、字距、圆角、渐变、间距按原值落地，  
不用靠肉眼估。哪些照搬、哪些不照搬：

| 照搬 | 不照搬 |
| --- | --- |
| 色值、字号、字重、行高、letter-spacing | `position: absolute` 及 `left/top` |
| 圆角、阴影、边框、渐变 | 固定 `width` / `height` |
| padding、gap | Figma 自动生成的 Frame 层级 |

Figma 稿是定宽绝对定位（本例 521px），要改成流式布局：宽度由平台容器决定，  
内边距按设计稿数值换算。设计稿里 `display: none` 的元素不用实现。

### 1.2 触发技能

技能在本包 `skill/custom-control-full-stack-generic/`，**先按 `skill/README.md`  
装到你的 AI 编码工具能加载到的位置**，否则下面的命令用不了。  
不同工具的加载机制不一样，`skill/README.md` 里给了几种常见工具的做法。

把这些信息一次给全，AI 就不用反复问。支持斜杠命令的工具：

```
/custom-control-full-stack-generic
根据 Figma 导出的图片和 CSS 为我生成自定义控件，严格还原颜色、间距、字体、字重等样式。
控件标识 secDevTalentProfileEdu，使用 React + antd。控件生成在 <你的控件目录> 下。
```

不支持斜杠命令的工具，把技能文件当上下文引用，效果一样：

```
请阅读 skill/custom-control-full-stack-generic/SKILL.md 并严格按其中的流程执行。
根据 Figma 导出的图片和 CSS 为我生成自定义控件，严格还原颜色、间距、字体、字重等样式。
控件标识 secDevTalentProfileEdu，使用 React + antd。控件生成在 <你的控件目录> 下。
```

技能会先确认这几项（已给的不重复问）：

| 项 | 本例取值 | 说明 |
| --- | --- | --- |
| 控件名 | `secDevTalentProfileEdu` | 后面预置 SQL 的 `FSCHEMAID` 必须与它完全一致 |
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
*   把 `src/styles/variable.less` 里 `react_demo` 变量前缀换成 APP\_NAME
*   `npm install`、拷 `CLAUDE.md`

后端插件（本包 `sample/backend/`，两个文件）：

*   `AbstractExtPortraitCardPlugin`：二开自己的基类
*   `EduBackgroundCardPlugin`：卡片插件

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

基类**不做 pageCache 缓存**——完全自建的卡片数据由自己插件独占推送，缓存没意义。  
只有「扩展标品卡片、在标品已推数据上追加字段」才需要缓存，见附录 A。

### 1.4 如果只生成了前端、没生成后端插件

技能理论上会主动问「是否需要生成对应的后端 Java 插件」，但实际不一定问到，  
或者你当时答了「先不用」。这时候补一次即可，不用重跑整个技能。

**关键是让 AI 继承二开模板基类，而不是继承标品的 `AbstractPortraitCardPlugin`。**  
不明确说的话，AI 看到工作区里有标品基类，很可能直接继承过去——那样二开就被绑死在  
标品版本上了。

先把 `sample/backend/AbstractExtPortraitCardPlugin.java` 拷进你的二开工程、  
改好包名，然后让 AI 照它写卡片插件。提问时把下面几项一次给全：

| 要告诉 AI 的 | 说明 | 本例 |
| --- | --- | --- |
| 继承哪个基类 | **必须点明是二开模板基类**，给出它在你工程里的完整类名 | `<你的包>.AbstractExtPortraitCardPlugin` |
| 二开插件目录 | 新插件放哪个模块、哪个包下 | `<二开工程>/code/<xxx>-formplugin/src/main/java/<你的包>/portrait/` |
| 查询什么实体 | 实体标识 | 教育经历实体 |
| 过滤字段 | 按什么条件筛，值从哪来 | 按 `employee` 等于当前员工 ID |
| 需要哪些字段 | 逐个列出，含要不要点路径带出的关联字段 | 学校、学历、专业、是否最高学历、起止日期 |
| 排序分组规则 | 按什么排序、要不要分组、要不要去重 | 按结束日期倒序，近的在前 |
| 派生数据 | 需要计算或拼接的字段 | 起止时间拼成 `2010 - 2013`，结束为空显示「至今」 |
| 多语言模块标识 | `ResManager.loadKDString` 第 3 个参数用什么 | 你工程的常量或字符串 |

一句话模板：

```
参照 sample/backend/AbstractExtPortraitCardPlugin.java（已拷到 <你的包> 下），
写一个人才画像「<卡片名>」卡片的后端插件，继承这个二开基类，
不要继承标品的 AbstractPortraitCardPlugin。

插件放在：<二开工程>/code/<xxx>-formplugin/src/main/java/<你的包>/portrait/
查询实体：<实体标识>
过滤条件：<按什么字段过滤>
需要字段：<字段清单，基础资料字段说明要取 name 还是 id>
排序分组：<排序规则、是否分组去重>
派生数据：<需要计算/拼接的字段及规则>
多语言模块标识：<你工程的常量或字符串>

推送的字段名要和前端 src/types/<xxx>.ts 里的 interface 完全一致。
```

最后一句很重要：**让 AI 先读前端的类型定义再写推送逻辑**。字段名对不上是这里最容易  
出的错，而且前端不报错、只是显示空白，排查起来费时间。

补完后自查三点：继承的是二开基类；推送字段名与前端 interface 逐一对应；  
查数遵守 step3 列的那几条硬性规范。

### 1.5 技能的两个提示

技能跑完会给出：

1.  **提示要换 icon**：识别到设计稿里有图标资源，让你把源文件放进 `src/assets/icons/`。  
    在你确认资源到位前，AI 用占位实现顶着，不会硬编码不存在的路径。
2.  **问询是否生成控件方案预置脚本**：这步不做，平台里选不到控件。答「是」进 step2。

### 1.6 顺手确认数据契约

契约是前后端唯一的对接口径，`src/types/edu.ts`：

`export interface EduItem {   id: string             school: string         highest?: boolean      degree?: string        major?: string         desc?: string          period?: string      }  export interface EduCardData {   empty?: boolean        cardTitle?: string     list?: EduItem[] }`

三个约定值得固化：

*   `empty` 让后端判定，前端不猜「列表为 0 是没数据还是没查」
*   `period` 由后端格式化好，日期格式化涉及多语言和平台日期格式，放后端更稳
*   `cardTitle` 可缺省，用户改过标题才下发

### 1.7 本地验一遍

`npm run mock       npm run dev:ram`   

mock 业务数据单独放 `mock/data/init.js`，由 `mock/data/index.js` 组装进 `initMock`，  
不要塞公共文件。至少过这几种：有数据、空数据（`empty: true`）、条目溢出（看列表是否内部滚动、  
卡片是否没被撑破）、超长文本、多语言切换、主题色切换。

需要连真实环境时用 `npm run dev`，在测试环境页面 URL 后拼  
`&kdcus_cdn=http://localhost:<DEV_RAM_PORT>`，平台会来本地拉控件资源。  
端口冲突改 `app.config.js` 的 `DEV_RAM_PORT` / `DEV_CACHE_PORT` / `MOCK_PORT`。

* * *

## step2 替换图标 + 生成控件方案预置脚本

### 2.1 放图标并让 AI 替换

把设计稿导出的 svg 放到 `src/assets/icons/`，然后告诉 AI 替换。

替换前（占位实现）：

![1.1 替换图标前.png](../images/01001290f1e25bf1423ca6abc6b6ee659c87.png)  
替换后：

![1.2 替换图标后.png](../images/01005d9de81151a24acfb5a572a32ee422ca.png)

**一个必踩的坑**：设计稿导出的 svg 常常自带背景底色和圆角。本例这个是 48×48、  
带 `linear-gradient(134.2deg, #A9DCF8, #B3A0F6)` 底和 8px 圆角。  
这时要把容器上的 `background` 撤掉，只留尺寸约束，否则渐变叠两层、颜色偏。

`.icon {   flex: none;   width: 48px;   height: 48px; }`

svgr 模板里已配好，直接当组件用：

```
import GraduationIcon from '@/assets/icons/GraduationIcon.svg'

<GraduationIcon className={Style.icon} />
```

### 2.2 生成控件方案预置脚本

指定输出目录让 AI 生成。三件事按序做：生成 ID → 查库校验 → 写 SQL。

**生成 ID**：`FID` 是 19 位正 long，`FPKID` 是 12 位平台风格字符串，  
**必须用雪花算法工具生成，不能手写或复制改写**。雪花算法高位是当前毫秒时间戳，  
"现在"生成的 ID 必然大于历史上任何已生成的 ID，天然不冲突；编造的数字保证不了唯一，  
会导致预置数据覆盖或主键冲突。技能内置了零依赖工具，有 JDK 即可跑：

`javac -encoding UTF-8 SnowflakeIdGen.java java SnowflakeIdGen 3`

源文件含 UTF-8 中文注释，编译必须带 `-encoding UTF-8`，否则默认 GBK 的 Windows 环境会失败。

一组 ID 只用于一条记录。多语言表每种语言一行、各自一个 `FPKID`，  
所以本例需要 1 个 FID + 2 个 FPKID。

**查库校验**：概率上不冲突，仍要写入前查一次，防他人已提交的预置数据。  
元数据类预置数据在 meta 库：

`SELECT FID FROM T_META_CTLSCHEMA  WHERE FID = <生成的FID> OR FSCHEMAID = '<你的控件标识>'; SELECT FPKID FROM T_META_CTLSCHEMA_L  WHERE FPKID IN (<生成的FPKID列表>);`

除主键外还要校验业务唯一键 `FSCHEMAID` 不与已有控件重复。两条都返回空才可落库。

**写 SQL**：DELETE + INSERT 幂等，可重复执行。

`DELETE FROM T_META_CTLSCHEMA_L WHERE FID = 1539508029412610048; DELETE FROM T_META_CTLSCHEMA WHERE FID = 1539508029412610048;  DELETE FROM T_META_CTLSCHEMA_L WHERE FID IN (   SELECT FID FROM T_META_CTLSCHEMA WHERE FSCHEMAID = 'secDevTalentProfileEdu'); DELETE FROM T_META_CTLSCHEMA WHERE FSCHEMAID = 'secDevTalentProfileEdu';  INSERT INTO T_META_CTLSCHEMA     (FID, FSCHEMAID, FPREVIEWIMG, FVERSION, FATTACHMENTCOUNT, FISVID, FMODULEID) VALUES     (1539508029412610048, 'secDevTalentProfileEdu', ' ', '1', 0, 'kingdee', 'hr');  INSERT INTO T_META_CTLSCHEMA_L (FPKID, FID, FLOCALEID, FSCHEMANAME) VALUES ('4X4PKG68I6O/', 1539508029412610048, 'zh_CN', '人才画像教育背景二开demo');  INSERT INTO T_META_CTLSCHEMA_L (FPKID, FID, FLOCALEID, FSCHEMANAME) VALUES ('4X4PKG6C3MWI', 1539508029412610048, 'zh_TW', '人才畫像教育背景二開demo');`

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

插件主流程：

`public class EduBackgroundCardPlugin extends AbstractExtPortraitCardPlugin {      @Override     public void afterBindData(EventObject e) {         super.afterBindData(e);          Long employeeId = getEmployeeId();         if (employeeId == null) {             logger.info("EduBackgroundCardPlugin: employeeId not found, push empty.");             pushEmpty();             return;         }          List<Map<String, Object>> list = queryEduList(employeeId);         if (list.isEmpty()) {             pushEmpty();             return;         }          Map<String, Object> data = new HashMap<>(4);         data.put("list", list);                  pushData(data);     } }`

**推送的 Map 字段名必须与前端 interface 逐一对应**，写之前先把前端类型定义读一遍。

多语言：所有面向用户的中文串走 `ResManager.loadKDString("教育背景", "EduBackgroundCardPlugin_0", "<你工程的模块标识>")`，第 2 个参数是 `类名_序号`，  
序号从 0 递增。每个词条都要写进工程的 `resources/<工程名>_zh_CN.properties`。  
英文标识符、日志、纯数字编码不需要处理。

* * *

## step4 在预置卡片上配置

进苍穹设计器，打开一张预置卡片改它，**不新建表单**。

### 4.1 挑一张未被占用的预置卡片

从 `hrti_basecard01` ~ `hrti_basecard20` 里挑一张还没人用的。挑好后按 4.2 或 4.3  
二选一配置，两种模式不能混用。

预置卡片已经继承好 `hrti_profilecardtpl`，继承路径满足画像视图配置的筛选条件，  
不需要另配继承关系。

### 4.2 自定义控件模式

适合设计稿复杂、要 100% 还原 UI 的卡片。三个动作：

**① 隐藏 `talentprofcardmark`。** 整个标记面板设为不可见，标题区随之隐藏。  
标题由控件内部自绘（含设计稿的字号字重字距），不会和模板标题重复。

标题文案的来源：用户在画像视图配置里改过标题就用配置值  
（基类 `getCardTitle()` 从 pageCache 的 `cardTitleMapEntry` 解析后下发），  
没改就用前端词条兜底。

**必须隐藏整个 `talentprofcardmark`，不是只隐藏 `content`。**  
承载页靠它的显隐判断卡片模式，只隐藏 `content` 会被判成表单模式，  
卡片外壳套一层阴影、控件自己又一层，变成双层阴影。

**② 给 `customcontrolap` 绑控件方案。** 控件方案选 step2 预置的那个，  
下拉里能看到 `人才画像教育背景二开demo` 说明预置 SQL 生效了。  
选不到就回查 `FSCHEMAID` 与 `APP_NAME` 是否一致、SQL 是否执行成功。

`customcontrolap` 是预置卡片自带的，标识保持默认不要改，  
改了插件里要覆盖 `getControlKey()`。

**③ 控件高度设成 100%。** 这一步直接决定卡片高度表现，让控件宿主容器撑满卡片。

配套的是前端侧也要撑满，两边必须一致：

`.card {   box-sizing: border-box;   display: flex;   flex-direction: column;   height: 100%;         overflow: hidden;     padding: 24px 0 0; }  .list {   flex: 1 1 auto;   min-height: 0;        overflow-y: auto;   padding: 0 32px 24px; }  .emptyWrap {   display: flex;   flex: 1 1 auto;   align-items: center;   justify-content: center;   min-height: 0; }`

**为什么前端是 `height: 100%` 而不是 `min-height: 100%`：** 平台宿主容器的高度不是  
CSS 显式声明的（靠 flex / 定位撑开），`min-height` 的百分比基准取不到值会退化成内容高度，  
`height` 才会被撑开。本例最初用 `min-height`，表现是「本地看着正常、平台上卡片比邻居矮一截」。

另外 `overflow: hidden` 之后内容超高会被裁，所以列表必须自己是滚动区；  
`min-height: 0` 不写，flex 子项不会收缩、滚动条出不来。

### 4.3 苍穹表单模式

适合常规字段罗列、表格、分录的卡片，不写前端控件。两个动作：

**① 隐藏 `customcontrolap`。** 这条路不用自定义控件，把它设为不可见。

**② 在 `content` 里加字段。** `talentprofcardmark` 与标题区保持可见，标题就用模板的。  
往 `content` 里拖标准控件，和开发普通动态表单完全一样：

*   文本/数字/日期字段：展示单值
*   分录（EntryGrid）：展示列表数据，例如多条教育经历
*   Flex 容器：分组、控制排列
*   标签（Label）：静态文案

要点：

*   字段标识自己定，后端插件里靠标识 `setValue`，两边保持一致
*   卡片是只读展示，字段设成不可编辑（插件里 `preOpenForm` 设 VIEW 状态也会统一压成只读）
*   卡片高度跟着模板走，不需要像控件模式那样单独设 100%
*   标题不用隐藏，这条路没有「控件内部又画一个标题」的重复问题

插件写法与表单模式特有的注意点见附录 B。

### 4.4 注册后端插件（两种模式都要）

在卡片表单的插件列表里挂上你的插件：

![2.3 给二开卡片绑定插件.png](../images/010024582114c10349b8b1ac1aab30e864a5.png)

**这步漏掉的表现是：卡片框架能出来，但控件模式永远空态、表单模式字段全空**，  
因为没人给它推数据。

### 4.5 外壳阴影与圆角不用管

卡片外框的阴影、圆角由承载页 `hrti_workbench` 的 `gridcontainerap` 自定义样式  
按 `talentprofcardmark` 的显隐自动切换：mark 可见走表单模式外壳，  
mark 隐藏则由 `customcontrolap` 自己出阴影。**二开不需要写任何样式。**

前提是两种模式的显隐配对，别配串了。

* * *

## step5 放开白名单并加入画像视图

### 5.1 把卡片编码加入白名单参数

预置卡片默认全部被屏蔽，先放开你用的这张，否则 5.2 里选不到。

改人才发展开发参数配置（元数据标识 `tdcs_cfgparam`）里的这条参数：

| 项 | 值 |
| --- | --- |
| 参数编码 | `hrti_portraitcard_whitelist` |
| 参数名称 | 人才画像PC端卡片白名单 |
| 取值 | 你用的预置卡片编码，多个用英文逗号分隔 |

例如你用了 `hrti_basecard01` 和 `hrti_basecard03`，就填：

```
hrti_basecard01,hrti_basecard03
```

标品预置为空串，即所有预置卡片都不可选。**这个参数没有菜单入口**，  
直接改数据库或用查询分析器更新，注意只追加你自己的编码，不要覆盖别人已加的。

白名单只控制配置页的可选范围。已经配进视图的卡片，即使后来被移出白名单，  
画像页仍会照常渲染，不会突然消失。

自建卡片（附录 C）不受白名单限制，不用加。

### 5.2 画像视图配置增加二开卡片

进画像视图配置（PC 端），把卡片加进视图：

![3\. 画像视图配置-PC端 增加二开卡片.png](../images/0100790641c3cb8d4eaa9bd82579e1fe2b85.png)

**这个页面里所有卡片都显示「暂无数据」是正常的**，包括标品卡片。  
配置页只负责编排卡片位置和标题，不会传 `employee` 参数给卡片，  
插件取不到 employeeId 就走 `pushEmpty()`。

判断方法：看旁边的标品卡片。它们也空 → 正常配置态；只有你的卡片空 → 才是你的问题。

配置页里能确认的正常信号：

*   卡片标题显示的是你配置的名字 → 控件方案生效、控件加载成功、`cardTitle` 解析并下发成功
*   空态样式撑满容器、和标品卡片一样高 → 控件高度 100% 生效（见 step4.2）
*   卡片外壳阴影与相邻标品卡片一致 → 两种模式的显隐配对正确

* * *

## step6 从画像列表验证实际效果

全景人才画像列表点员工超链进入画像页：

![4\. 全景人才画像列表点击超链查看画像.png](../images/0100cc29a8725d7e4004a3709949ee4ab693.png)

这时页面带 `employee` 参数，插件能取到 employeeId，卡片渲染真实数据。  
到这里整条链路走完。

* * *

## 排错对照表

| 现象 | 可能原因 |
| --- | --- |
| 设计器控件方案下拉里没有你的控件 | step2 预置 SQL 没执行，或 `FSCHEMAID` 与 `APP_NAME` 不一致 |
| 预置卡片在画像视图配置里选不到 | 编码没加入白名单参数 `hrti_portraitcard_whitelist`（见 step5.1）；或那张已被其他二开卡片占用，换一张 |
| 画像视图配置里选不到这张卡片 | 自建卡片没继承 `hrti_profilecardtpl`（预置卡片不会有这问题） |
| 卡片框架出来了但永远空态 | step4.4 没挂插件；或控件标识不是 `customcontrolap` 且插件没覆盖 `getControlKey()`；或后端插件根本没生成（见 step1.4） |
| 二开插件继承了标品基类 | 生成插件时没点明继承二开模板基类，见 step1.4 |
| 配置页里所有卡片都空态 | 正常，配置页不传 `employee`（对照标品卡片是否也空） |
| 卡片里出现两个标题 | 控件模式没隐藏 `talentprofcardmark`，标题区还露着 |
| 卡片出现双层阴影 | 控件模式只隐藏了 `content`，没隐藏整个 `talentprofcardmark`，被判成表单模式 |
| 卡片外壳没有阴影 | 表单模式误把 `talentprofcardmark` 隐藏了，被判成控件模式 |
| 卡片高度比邻居矮 | 控件高度没设 100%，或前端用了 `min-height: 100%` |
| 卡片白屏 | 控件静态资源没部署，或前端运行时报错（看浏览器控制台） |
| 前端收不到 `setData` | 推送字段名与前端 interface 不匹配；或 `getEmployeeId()` 返回 null 走了空态 |
| 界面显示词条 key 名 | 词条未注册且没做 FALLBACK 兜底 |
| 图标颜色偏 | svg 自带渐变底，容器又叠了一层 `background` |

## 收尾自查

IDE 侧：

*   [ ]  `app.config.js` 的 `APP_NAME` / `ISV` / `MODULE_ID` 已改，`variable.less` 变量前缀已同步
*   [ ]  图标已替换，容器没有叠加多余的 `background`
*   [ ]  若控件目录存在硬编码清单的批量打包脚本，控件名已加入
*   [ ]  空态、溢出滚动、超长文本、多语言、主题色都验过
*   [ ]  `npm run build` 与 lint 通过
*   [ ]  后端插件不继承标品类，推送字段名与前端 interface 逐一对应
*   [ ]  中文串走 `ResManager.loadKDString`，词条已写进 properties
*   [ ]  控件方案预置 SQL 已生成，ID 已查库校验无冲突

平台侧（通用）：

*   [ ]  用的是预置卡片 `hrti_basecard01` ~ `hrti_basecard20`，且这张没被别人占用
*   [ ]  卡片编码已加入白名单参数 `hrti_portraitcard_whitelist`
*   [ ]  后端插件已注册
*   [ ]  卡片已加入画像视图，画像工作台里数据正常
*   [ ]  外壳阴影与相邻标品卡片一致（没有双层、也没有缺失）

自定义控件模式追加：

*   [ ]  `talentprofcardmark` 整个隐藏（不是只隐藏 `content`）
*   [ ]  `customcontrolap` 已绑控件方案，高度 100%

苍穹表单模式追加：

*   [ ]  `customcontrolap` 已隐藏
*   [ ]  `talentprofcardmark` 与标题区保持可见
*   [ ]  字段加在 `content` 里，标识与插件 `setValue` 一致

* * *

## 附录 A 扩展标品卡片（另一种场景）

如果目标不是新增卡片，而是给标品已有卡片补字段，做法完全不同：**不新建卡片表单**，  
而是把你的插件注册在标品插件**之后**（插件按元数据顺序串行执行），  
走「读标品缓存 → 追加字段 → 回写缓存 → setData」：

`String json = getPageCache().getBigObject("customcontrolap"); if (json == null || json.isEmpty()) {          return; } Map<String, Object> data = SerializationUtils.fromJsonString(json, Map.class); if (Boolean.TRUE.equals(data.get("empty"))) {     return; } data.put("yourField", value); getPageCache().putBigObject("customcontrolap", SerializationUtils.toJsonString(data)); getControl("customcontrolap").setData(data);`

三个坑：缓存没就绪时不要推（会把标品字段整份顶掉）；`empty=true` 直接返回；  
标品若有增量刷新（前端点刷新触发 `customEvent` 后重推），要在对应 `customEvent`  
里同样补一次追加，否则你的字段会被覆盖。

* * *

## 附录 B 苍穹表单模式补充说明

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

**平台侧配置见 step4.3**（隐藏 `customcontrolap`、在 `content` 里加字段），  
这里只讲插件写法与这条路特有的坑。

### B.1 开发插件（继承二开模板插件）

插件仍然**继承你自己工程的 `AbstractExtPortraitCardPlugin`**，不继承标品类，原因同正文。

基类里 `getEmployeeId()`、`preOpenForm` 这两个能力照用；  
`pushData()` / `pushEmpty()` / `getCardTitle()` 是给自定义控件推 JSON 用的，  
这条路不需要，改成往表单字段写值。

`public class EduBackgroundFormCardPlugin extends AbstractExtPortraitCardPlugin {           private static final String ENTRY_EDU = "entryentity";      @Override     public void afterBindData(EventObject e) {         super.afterBindData(e);                   Long employeeId = getEmployeeId();         if (employeeId == null) {             logger.info("EduBackgroundFormCardPlugin: employeeId not found.");             return;         }          List<Map<String, Object>> list = queryEduList(employeeId);         if (list.isEmpty()) {                                       getView().setVisible(Boolean.FALSE, ENTRY_EDU);             return;         }          fillEntry(list);     }           private void fillEntry(List<Map<String, Object>> list) {         AbstractFormDataModel model = (AbstractFormDataModel) getModel();         model.beginInit();         model.deleteEntryData(ENTRY_EDU);         model.batchCreateNewEntryRow(ENTRY_EDU, list.size());         for (int i = 0; i < list.size(); i++) {             Map<String, Object> item = list.get(i);             model.setValue("school", item.get("school"), i);             model.setValue("degree", item.get("degree"), i);             model.setValue("major", item.get("major"), i);             model.setValue("period", item.get("period"), i);         }         model.endInit();         getView().updateView(ENTRY_EDU);     } }`

几个和自定义控件不同的注意点：

*   **批量写分录**要包在 `beginInit()` / `endInit()` 里，最后 `updateView` 一次，  
    逐行 `setValue` 触发的界面刷新会很慢
*   **空数据**没有前端 `empty` 约定，自己决定表现：隐藏内容区、显示一个「暂无数据」Label，  
    或者干脆让卡片不展示
*   **多语言**同样走 `ResManager.loadKDString`，词条写进工程的 properties
*   查数规范与正文 step3 完全一致（`DynamicObjectCollection`、点路径取基础资料、  
    `QCP.in` 批量查、`HRDateTimeUtils` 格式化日期）

### B.2 绑定插件

在卡片表单的插件列表里注册这个插件，做法与正文 step4.4 一样。  
漏了的表现是卡片能出来但字段全空。

### B.3 后续步骤同正文

剩下两步和自定义控件那条路完全一样：

*   **step5 放开白名单并加入画像视图**：先把卡片编码加入 `hrti_portraitcard_whitelist`  
    （表单模式同样受白名单限制），再进画像视图配置（PC 端）把卡片加进视图。  
    同样地，配置页不传 `employee`，所以配置页里卡片没数据是正常的，看标品卡片是否也空来判断。
*   **step6 从画像列表验证**：全景人才画像列表点员工超链进画像页，  
    这时带 `employee` 参数，插件取到 employeeId，卡片渲染真实数据。

### B.4 本附录的自查

*   [ ]  `customcontrolap` 已隐藏，`talentprofcardmark` 与标题区保持可见
*   [ ]  字段加在 `content` 里，标识与插件里 `setValue` 用的一致
*   [ ]  插件继承自己工程的 `AbstractExtPortraitCardPlugin`，不是标品类
*   [ ]  分录批量写包在 `beginInit` / `endInit` 里
*   [ ]  空数据有明确表现
*   [ ]  中文串走 `ResManager.loadKDString`，词条已写进 properties
*   [ ]  插件已在卡片表单上注册
*   [ ]  卡片已加入画像视图，画像工作台里数据正常

* * *

## 附录 C 预置卡片用完后自建

20 张 `hrti_basecard01` ~ `hrti_basecard20` 全部占满之后，才需要自己建卡片表单。

在 hrti 扩展应用下新建动态表单，**继承 `hrti_profilecardtpl`**（画像卡片公共父模板）。  
继承它就自动满足了画像卡片的两个前提：继承路径里含小部件基类（画像视图配置页才筛得到）、  
卡片外框结构与预置卡片一致。

`hrti_profilecardtpl` 本身是纯模板，只供继承，不是可独立选用的卡片，  
标品在画像视图配置的可选清单里把它屏蔽了。你继承出来的新卡片不受影响。

继承后的结构、两种模式的配置方式与预置卡片完全相同，按 step4.2 / step4.3 配即可。  
**自建卡片同样要遵守 `talentprofcardmark` 的显隐约定**：控件模式隐藏整个 mark，  
表单模式保留 mark 隐藏 `customcontrolap`。承载页的外壳样式靠这个判断模式，  
配串了就会出现双层阴影或阴影缺失。

[自定义控件部署](https://developer.kingdee.com/knowledge/768125700280322048?specialId=194046086670543360&productLineId=29&isKnowledge=2&lang=zh-CN)
