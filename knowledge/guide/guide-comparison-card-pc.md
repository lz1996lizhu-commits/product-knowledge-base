---
title: 人才对比卡片二开实操步骤（PC 端）
category: guide
cloud: 人才发展云
tags: [人才对比, 二开, PC端, 人才星图, 卡片开发, 苍穹表单, 开发指南]
aliases: [人才对比卡片PC端, PC端对比卡片二开]
author: 金蝶云社区
created: 2026-09-28
updated: 2026-09-28
source: https://vip.kingdee.com/knowledge/892423019137219584
---

# 人才对比卡片二开实操步骤（PC 端）
以「教育经历」区块为例，按实际操作顺序记录。**本文只讲 PC 端**，移动端另见移动端交付包。

配套代码：`sample/frontend/`（前端控件工程）、`sample/backend/`（后端插件 + 控件方案预置 SQL）、`figma/`（设计稿产物）。  
对接机制看 `doc/人才对比数据展示控件事件规范.md`，**必读**，  
四种 setData 指令、列宽、空数据、超长截断规范都在里面。

占位符 `<二开工程>`、`<你的包>`、`<你的控件目录>` 替换为实际值。

整体做法：**不新建表单、没有视图配置页**，直接扩展标品表单 `hrti_talentcomparison`，  
在内容区 `contentflex` 下加自定义控件，再挂一个扩展插件推数据。  
控件标识自己起名（一张表单挂十几个控件，不能重名），元数据里加了就在页面上。

数据形态是**多员工并排、一列一人**，还要处理对比人的增加、移除、拖拽排序。

五步，前三步在 IDE 让 AI 做，后两步在平台配：

| 步骤 | 做什么 | 在哪 |
| --- | --- | --- |
| step1 | 触发技能生成前端控件与后端插件 | IDE |
| step2 | 生成控件方案预置脚本 | IDE |
| step3 | 换真实取数（step1 说清了可省） | IDE |
| step4 | 扩展 `hrti_talentcomparison`，加控件、配样式、挂插件 | 苍穹设计器 |
| step5 | 从对比入口验证增删移 | 平台页面 |

* * *

## step1 触发技能生成前端控件与后端插件

### 1.1 准备设计稿产物

Figma 选中区块，导出 2x PNG 截图 + `Inspect` 面板的 CSS（存 txt），放一个目录（本例 `figma/`）。  
CSS 必须导，颜色字号字重行高圆角间距才能按原值落地。

**唯一要特别小心的是卡片宽度。** 设计稿标 `width: 280.67px`，那是 Figma 在特定画板宽度下量的，  
照抄会写死列宽、不随容器伸缩，**你的区块就和上下标品区块对不齐**。  
标品统一用弹性宽度，见 1.3。

其余不照搬的：`position: absolute` 及 `left/top`、固定 `height`、Figma 自动生成的 Frame 层级。

### 1.2 触发技能

技能在 `skill/custom-control-full-stack-generic/`，**先按 `skill/README.md` 装到 AI 工具能加载的位置**。

```
/custom-control-full-stack-generic
根据 Figma 导出的图片和 CSS 为我生成自定义控件，严格还原颜色、间距、字体、字重等样式。
控件标识 secDevTalentComparisonEdu，使用 React + antd。
自定义控件生成在 <你的控件目录> 下，后端插件写在 <你的插件目录> 下。

取数逻辑：
查询实体：<实体标识>
过滤条件：按 employeeIds 批量过滤（一次最多 20 人）
需要字段：<字段清单，基础资料字段说明取 name 还是 id>
排序分组：<排序规则，以及按 employeeId 分组>
派生数据：<需要计算/拼接的字段及规则，如起止年份拼成 2016-2019>
多语言模块标识：<你工程的常量或字符串>

请同时阅读 doc/人才对比数据展示控件事件规范.md，
按第二节补 update 的 action 分派与 store 的 handleAction，
按第三节落地列宽、空数据、超长截断规范。
```

不支持斜杠命令的工具，改成「请阅读 skill/custom-control-full-stack-generic/SKILL.md 并严格按其流程执行」+ 同样的需求。

**取数逻辑在这一步就说清，能省掉 step3。** 不说的话技能会先用静态数据打通链路，  
后面还得再返工一次。人才对比的取数有两点和普通卡片不同，写需求时要交代：  
**按 employeeIds 批量查再在内存分组**（不是单员工查询），  
**查不到数据的员工也要占一列**（下发 `entries: []`，详见 1.5）。

**最后那段也必须说。** 技能模板本身不含增量处理和那几条规范，不说明生成出来的控件  
**首次加载正常、删人和拖拽排序没反应**。

技能会确认：控件名（`secDevTalentComparisonEdu`，后面预置 SQL 的 `FSCHEMAID` 必须一致）、  
框架（React + Antd）、ISV（`kingdee`）、MODULE\_ID（`hr`）、目标目录（必须给，技能不猜）。

> `sample/` 里的示例用的是静态模拟数据，只为让 demo 不依赖环境就能跑起来，  
> **不是推荐做法**，你的二开要按上面的模板把真实取数说清。

### 1.3 前端产出必须自查的三处

技能从离线模板复制，自动改好 `package.json` 的 `name`、`app.config.js` 的三项、  
`variable.less` 变量前缀，并跑 `npm install`。以下三处模板不带，**逐一确认 AI 补上了**：

**（1）`update` 按 action 分派**（`src/index.tsx`）。模板原样是无脑 `setAjaxData(props)`，  
会把增量指令当全量数据覆盖掉：

```
update(props: ReturnDataType) {
  saveConfig(props, this.zustandStore)
  const store = this.zustandStore.useGlobalStore.getState()
  const action = props?.data?.action
  if (action === 'insert' || action === 'remove' || action === 'move') {
    store.handleAction(props.data)   // 增量：合并进已有数据
  } else {
    store.setAjaxData(props)         // 全量：整体覆盖
  }
}
```

**（2）store 里补 `handleAction`**（`src/store/useGlobalStore.ts`），实现 insert（插到最前）/  
remove / move 三种合并。模板没这个方法，且 store 要从 `create(set => ...)` 改成  
`create((set, get) => ...)` 才读得到当前状态。完整代码在事件规范 2.2 节。

漏了（1）（2）的表现：**首次加载正常，删人、拖拽排序后卡片不动**。

**（3）列宽用弹性值，不是设计稿的固定像素**。标品六个控件逐字一致：

`.cardRow {                       display: flex;   flex-wrap: nowrap;             align-items: stretch;          gap: 16px;   width: 100%; } .card {   flex: 1 1 280px;               min-width: 280px;   max-width: 410px;   padding: 16px;   border: 1px solid #eaeef2;   border-radius: 16px;   overflow: hidden; }`

**长文本截断要配套三件事**，少一件提示就不弹或弹了看不全：

*   文本节点要 `flex: 1`（或 `flex: 0 1 auto`）+ `min-width: 0`，否则文本把行撑开、  
    `text-overflow` 根本不触发，连带提示也永远不弹。同一行里次要字段（如学历）  
    和分隔线、标签用 `flex-shrink: 0` 保持完整
*   hover 提示用 antd Tooltip 且只在真截断时弹（标品统一白底深字），  
    **PC 端不要用原生 `title` 属性**——原生是系统灰底、延迟约 1 秒、没截断也弹，  
    和标品观感不一致
*   元数据上要加 `$ { overflow: visible; }`，否则浮层被宿主容器裁掉，见 step4.4

```
const EllipsisTooltip = ({ text, className }) => {
  const ref = useRef<HTMLSpanElement>(null)
  const [isOverflow, setIsOverflow] = useState(false)
  useEffect(() => {
    const el = ref.current
    if (el) setIsOverflow(el.scrollWidth > el.clientWidth)
  }, [text])
  return (
    // 没截断时传 undefined，Tooltip 就不弹
    <Tooltip title={isOverflow ? text : undefined} color="#fff" overlayInnerStyle={{ color: '#333' }}>
      <span>{text}</span>
    </Tooltip>
  )
}
```

**缺省展示**：一个对比人都没有时整块 `return null`（标品 17 个控件一致），  
**不需要缺省态插画**——标品 PC 端全目录没有插画，别去画一个。单人无数据是空卡片，见 1.5。

### 1.4 后端插件要解决什么

标品对二开控件的覆盖范围**不全**，这是二开必须写插件的唯一原因：

| 指令 | 覆盖二开控件 | 二开要做什么 |
| --- | --- | --- |
| `remove` / `move` | 【emoji】 | 不用管，标品遍历 `contentflex` 广播 |
| `insert`（新增对比人） | 【emoji】 | **自己补推** |
| 全量（刷新 / F7 选完人） | 【emoji】 | **自己补推** |

`remove` / `move` 是遍历 `contentflex` 子项广播的，能覆盖二开；`insert` 和全量走的是  
**标品控件常量清单**，二开控件不在清单里，拿不到数据就不推。  
漏了插件的表现：**删人、排序正常，新增对比人和刷新时不刷新**。

**不要继承或覆写标品 `TalentComparisonPlugin`**：二开不能改标品代码，且会绑死标品版本。  
只依赖平台 `kd.bos.*` 和标品公开的约定（控件标识、pageCache key、事件名）。

插件里有三处**错了不会报错、很难查**，必须照做：

**（1）必须自己注册 F7 监听**，否则 `afterF7Select` 根本不会被调用。监听注册在控件实例上，  
标品注册的是它自己，不会连带把二开插件注册进去。漏了**没有任何报错**，只是选完人不刷新。

`@Override public void registerListener(EventObject e) {     super.registerListener(e);     MulEmployeeEdit multiEmployee = (MulEmployeeEdit) this.getView().getControl("multiemployee");     if (multiEmployee != null) {         multiEmployee.addAfterF7SelectListener(this);     } }`

**（2）员工顺序读标品的 pageCache，别自己另存**。key 是 `currentEmployeeIds`，  
标品增删移时都会更新它，二开与标品同页共享 pageCache，直接读即可。自己存必然和标品不一致。

**（3）新增对比人时推全量，不推 `insert` 增量**。事件参数里的 ID 是前端原样传来的，  
标品处理时还会做去重和 20 人上限截断，二开插件跑在标品之后，**拿不到「本次真正新增了谁」的差集**——  
对已在列表里的人或被截断掉的人推 `insert`，前端会多出一列重复数据。按缓存顺序推全量既正确又幂等。

参考实现：`sample/backend/TalentComparisonSecDevExtPlugin.java`。

#### 做多个二开控件时：只写一个插件，不要一个控件一个插件

**插件是挂在表单上的，不是挂在控件上的。** 所以做 N 个二开控件也只需要一个扩展插件，  
N 个控件的取数与推送都写在里面。标品自己就是这么做的：一个  
`TalentComparisonPlugin` 驱动 19 个控件，元数据上业务插件只注册了一个。

一个控件挂一个插件会出问题，**而且不是「多余但无害」**：

*   `afterBindData` / `customEvent` / `afterF7Select` 都是**表单级事件**，  
    挂 N 个插件就触发 N 次。每个插件各自读缓存、各自查库，**查库次数翻 N 倍**
*   每个插件都要重复写 F7 监听注册、读 `currentEmployeeIds`、判可见性这套样板代码
*   插件顺序要逐个保证排在标品之后，多一个就多一处能配错的地方

写法上照标品的模式：**控件标识清单 + 按 key 分派**，把「什么时候推」和  
「推什么数据」拆开，新增控件只需加一个 case：

`private static final String[] EXT_CONTROLS = { "edusecdevdemo", "othersecdevdemo" };  private void pushFullData(List<Long> employeeIds) {     for (String controlKey : EXT_CONTROLS) {         CustomControl control = this.getControl(controlKey);                  if (control == null || control.isInvisible()) {             continue;         }         Map<String, Object> data = buildDataForControl(controlKey, employeeIds);         if (data == null) {             continue;         }         data.put("times", System.currentTimeMillis());         control.setData(data);     } }  private Map<String, Object> buildDataForControl(String controlKey, List<Long> employeeIds) {     switch (controlKey) {         case "edusecdevdemo":   return buildEduData(employeeIds);         case "othersecdevdemo": return buildOtherData(employeeIds);         default:                return null;     } }`

**多个控件共用一批员工数据时，先查一次再分给各控件**，别每个控件各查一遍同样的表  
（标品用 `prepareEmployeeContext` / `clearEmployeeContext` 做这件事）。

`sample/backend/` 里的示例只有一个控件，所以直接用了单个 `CONTROL_KEY` 常量；  
你要做第二个控件时，按上面的清单 + 分派改造，不要复制一份插件出来改。

### 1.5 数据契约

`src/types/edu.ts` 是前后端唯一口径：

`export interface EduPerson {   employeeId: string          entries?: EduEntry[]      } export interface EduCompareData {   empty?: boolean             cardTitle?: string   list?: EduPerson[]          times?: number            }`

四条硬约定：

*   **`employeeId` 必须是字符串**，类型不一致会静默匹配不上
*   **`entries: []` ≠ 不下发这个人**。查不到数据的人也要占一列（渲染成空卡片），  
    跳过会导致各区块列数不一致、横向错位
*   **`times` 每条都带**（含全量）。内容相同的两次 `setData` 前端识别不到变化，靠它触发 update
*   日期由后端格式化好下发，前端不做日期处理

### 1.6 本地验

`npm run mock       npm run dev:ram`   

mock 业务数据放 `mock/data/init.js`。**每个人给不同的 `employeeId`**，  
后面在平台调增删移时才看得出是哪列在动。至少过：多人并排（列宽是否等分）、  
某人 `entries: []`（是否渲染成空卡片而非整块消失）、条目数不同的人并排（各列是否等高）、  
超长学校名/专业名（省略号 + Tooltip）、多语言、主题色。

连真实环境用 `npm run dev`，测试环境 URL 后拼 `&kdcus_cdn=http://localhost:<DEV_RAM_PORT>`。

* * *

## step2 生成控件方案预置脚本

不做这步，平台里选不到控件。三件事按序：生成 ID → 查库校验 → 写 SQL。

**生成 ID**：`FID` 是 19 位正 long，`FPKID` 是 12 位平台风格字符串，  
**必须用雪花算法工具生成，不能手写或复制改写**（编造的数字保证不了唯一，会覆盖数据或主键冲突）。  
技能内置零依赖工具，有 JDK 即可：

`javac -encoding UTF-8 SnowflakeIdGen.java java SnowflakeIdGen 3`

源文件含 UTF-8 中文注释，**编译必须带 `-encoding UTF-8`**，否则默认 GBK 的 Windows 环境会失败。  
一组 ID 只用于一条记录；多语言表每种语言一行、各自一个 `FPKID`，本例需要 1 个 FID + 2 个 FPKID。

**查库校验**（元数据在 meta 库），两条都返回空才可落库：

`SELECT FID FROM T_META_CTLSCHEMA WHERE FID = <生成的FID> OR FSCHEMAID = '<你的控件标识>'; SELECT FPKID FROM T_META_CTLSCHEMA_L WHERE FPKID IN (<生成的FPKID列表>);`

除主键外还要校验业务唯一键 `FSCHEMAID` 不与已有控件重复。

**写 SQL**：DELETE + INSERT 幂等，完整脚本见  
`sample/backend/kd_secdev_meta_ctlschema_talentcomparisonedu_preset.sql`。字段对应：

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
*   多选基础资料按分录路径过滤（如 `xxx.fbasedataid.id`），结果展开成多行，分组时取扁平化路径值
*   **务必用 `QCP.in` 按 employeeIds 一次性批量查，再在内存里按员工分组**。  
    一次最多 20 人，循环查库会放大 20 倍
*   日期格式化用 `HRDateTimeUtils`，禁用 `SimpleDateFormat`（非线程安全）

**按员工组装时，查不到数据的人也要放进 list**，这是空框规范的后端一半（标品  
`EducationAssembler` 就是这么写的）：

`for (Long empId : employeeIds) {     Map<String, Object> personData = new HashMap<>(4);     personData.put("employeeId", String.valueOf(empId));     List<Map<String, Object>> entries = new ArrayList<>();     if (!CollectionUtils.isEmpty(eduList)) {              }     personData.put("entries", entries);        list.add(personData); }`

字段级空值用 `"-"` 占位（标品常量 `CommConstants.SHOW_EMPTY`），不要下发 `null` 或空串。

**推送的字段名必须与前端 interface 逐一对应**，写之前先读一遍前端类型定义。  
对不上前端不报错、只显示空白，排查费时间。

多语言：中文串走 `ResManager.loadKDString("教育经历", "<插件类名>_0", <模块标识>)`，  
第 2 个参数是 `类名_序号`（从 0 递增），词条写进工程的 `resources/<工程名>_zh_CN.properties`。  
插件落在 hrti 工程时，模块标识用 `CommConstants.FORM_PLUGIN`，词条文件是  
`code/hrmp-hrti-formplugin/src/main/java/resources/hrmp-hrti-formplugin_zh_CN.properties`。  
英文标识符、日志、纯数字编码（如 `211` / `985`）不用处理。

* * *

## step4 扩展 hrti\_talentcomparison 并加入控件

不新建表单，扩展标品表单 `hrti_talentcomparison`。

![1.0 扩展人才对比.png](../images/0100ba825e544f864b79b9cd6732f60c5645.png)

然后在扩展页面内容区加控件、配样式、挂插件：

![1.0 扩展人才对比.png](../images/01009c0156582ea24981a5a2ed03475c3fcf.png)

五个动作，缺一个就出不来、没数据或样式对不上。

### 4.1 控件必须直接挂在 `contentflex` 下

内容区标识是 `contentflex`，控件作为它的**直接子级**，放在希望的展示位置（区块按元数据顺序排列）。

**不能套进任何 Flex 子容器。** 标品广播 `remove` / `move` 时只遍历 `contentflex` 直接子项、不递归：

`for (Control child : contentFlex.getItems()) {     if (child instanceof CustomControl && !child.isInvisible()) {         ((CustomControl) child).setData(command);     } }`

套子容器的表现是**删人、排序对你的区块无反应**，而新增和刷新却正常（那两条是二开插件自己推的），  
很难往容器层级上想。

### 4.2 控件标识自己起名，并与插件一致

**一张表单挂十几个控件，标识必须唯一**，不能用设计器给的默认值。  
本例用 `edusecdevdemo`，标品那些是 `eduexperience` / `skill` / `rewardpunishment`。

起好后要和后端插件里登记的控件标识一致（单控件时是 `CONTROL_KEY` 常量，  
多控件时是 1.4 里那个 `EXT_CONTROLS` 清单）：

`private static final String CONTROL_KEY = "edusecdevdemo";`

不一致的表现是控件能加载但永远空白——插件 `getControl(...)` 拿到 null 直接 return 了。  
做了多个控件的话，**每个控件的标识都要登记进清单**，漏登记的那个就是这个表现。

### 4.3 绑定控件方案

选 step2 预置的那个，下拉里能看到 `人才对比教育经历二开demo` 说明预置 SQL 生效了。  
选不到就回查 `FSCHEMAID` 与 `APP_NAME` 是否一致、SQL 是否执行成功。

### 4.4 样式对齐标品：外边距 + 自定义样式

决定你的区块和标品看起来是不是一套，**两件都要做**。

**（1）外边距**：用格式刷从相邻标品控件刷一下，或手工设。标品 18 个数据展示控件完全一致：

| Top | Left | Bottom | Right |
| --- | --- | --- | --- |
| `0px` | `20px` | `20px` | `20px` |

漏了的表现是与上下标品区块左右不齐、上下间距忽宽忽窄。

**（2）自定义样式**：标品每个数据展示控件都挂了同一段，**新增控件默认没有，必须手工加**：

`$ {   overflow: visible; }`

`$` 是平台自定义样式的选择器占位符，指代控件自身宿主节点。

**为什么要它**：宿主容器默认裁掉溢出内容，而对比控件里有需要溢出显示的浮层（1.3 那个 Tooltip）。  
不加的表现是**控件本身渲染正常、只有 hover 的 Tooltip 被切掉一半或整个不可见**，  
很容易误判成 Tooltip 组件的 bug。

> 标品里三个控件的自定义样式不是这段：`headertitle`（多 `position: sticky` 吸顶）、  
> 两个收起态浮层（绝对定位 + 渐变背景）。**数据展示类照抄上面这段就对了。**

另外元数据上**要给控件配中文名称**，否则不进右侧「快速定位」导航（导航是运行时读  
`contentflex` 子控件生成的，无名称的被跳过）。

### 4.5 注册插件，且必须排在标品之后

扩展页面插件列表挂上你的插件，**顺序必须在  
`kd.hr.hrti.formplugin.web.comparison.TalentComparisonPlugin` 之后**。

插件按元数据顺序串行执行。二开插件靠读标品写入 pageCache 的 `currentEmployeeIds` 拿员工顺序，  
标品在增删移时更新它。**排在标品前面会读到上一次的旧顺序**——表现是新增对比人后，  
你的区块比别人少一列或顺序不对。

**不管做了几个二开控件，这里只挂一个插件**（原因见 1.4）。标品也只注册了一个业务插件、  
驱动全部 19 个控件。挂多个的表现不是报错，而是表单级事件被触发多次、查库次数成倍增加。

漏挂插件的表现：控件能加载、标题能出来，但永远没数据。

* * *

## step5 从对比入口验证

从全景人才画像点「人才对比」进页面，**逐个测添加、移除、移动**：

![2.1 从全景人才画像点击人才对比进入人才对比页面，点击添加、移除、移动测试，看页面效果以及数据是否正常，有问题则继续让AI调整.png](../images/01002d52cd9d95294a7087503b7d8b96af9c.png)

四个动作走的链路不同、失败原因也不同，**必须全测**：

| 动作 | 验证 | 挂了往哪查 |
| --- | --- | --- |
| 页面初次加载 | `afterBindData` 全量推送 | 插件是否注册、`CONTROL_KEY` 是否一致 |
| 添加对比人（F7 选人） | `afterF7Select` + 自己补的全量推送 | `registerListener` 里是否注册了 F7 监听（1.4） |
| 移除某人 | 标品 `remove` 广播 + 前端 `handleAction` | 控件是否直挂 `contentflex`；store 有没有 `handleAction` |
| 拖拽排序 | 标品 `move` 广播 + 前端 `handleAction` | 同上 |

同时看视觉是否与标品一致：

*   列宽与上下标品区块**逐列对齐**（不齐 → 宽度写死了固定值）
*   某人没数据时是**空框**（不是整块消失、也不是「暂无数据」文案）
*   各列**等高**（不等高 → 容器少了 `align-items: stretch`）
*   长文本 hover 的 Tooltip 完整显示（被切 → 少了 4.4 的 `overflow: visible`）
*   区块标题进了右侧「快速定位」导航（没进 → 元数据没配中文名称）

有问题继续让 AI 调。**报现象时说清是哪个动作触发的**——增删移走标品广播、  
新增/刷新走二开插件自推，是两条完全不同的路。

* * *

## 排错对照表

| 现象 | 可能原因 |
| --- | --- |
| 设计器控件方案下拉里没有你的控件 | step2 预置 SQL 没执行，或 `FSCHEMAID` 与 `APP_NAME` 不一致 |
| 区块出来了但永远没数据 | 没挂插件（4.5）；或控件标识与插件 `CONTROL_KEY` 不一致（4.2） |
| **删人、排序没反应**，新增和刷新正常 | 控件套了子容器、不是 `contentflex` 直接子级（标品广播不递归）；或前端缺 `handleAction`、`update` 没做 action 分派 |
| **新增、刷新没反应**，删人排序正常 | 二开插件没补 `insert` / 全量推送（标品清单不含二开控件） |
| F7 选完人不刷新，且无任何报错 | `registerListener` 里没注册 F7 监听 |
| 新增对比人后区块少一列或顺序不对 | 插件顺序排在标品之前，读到旧的 `currentEmployeeIds` |
| 多个二开控件时查库次数成倍增加、日志里同一段取数重复打印 | 一个控件挂了一个插件；应合成一个插件用控件清单 + 按 key 分派（1.4） |
| 做了多个控件，只有部分控件有数据 | 插件的控件标识清单漏登记了某个控件（4.2） |
| 新增对比人后多出一列重复数据 | 按事件参数推了 `insert` 增量；应按缓存顺序推全量 |
| 列宽和标品区块对不齐 | 照抄了设计稿固定宽，没用 `flex: 1 1 280px` |
| 各列高度不一致、内容错位 | 容器缺 `align-items: stretch`；或某人没数据时被后端跳过 |
| 某人没数据时整块消失 | 后端把没数据的人跳过了，应下发 `entries: []` |
| 长文本 hover 的 Tooltip 被切掉 | 4.4 的 `$ { overflow: visible; }` 没加 |
| 长文本不出省略号、Tooltip 不弹 | 文本节点缺 `flex: 1` + `min-width: 0`（1.3） |
| 区块左右/上下与标品不齐 | 外边距没设成 `0/20/20/20`（4.4） |
| 区块不在右侧快速定位导航里 | 元数据没给控件配中文名称 |
| 控件白屏 | 静态资源没部署，或前端运行时报错（看浏览器控制台） |
| 界面显示词条 key 名 | 词条未注册且没做 FALLBACK 兜底 |
| 隐藏的区块收不到任何推送 | 预期行为，插件推送前判可见性，不可见不查数不推数 |

## 收尾自查

IDE 侧：

*   [ ]  `app.config.js` 的 `APP_NAME` / `ISV` / `MODULE_ID` 已改，`variable.less` 变量前缀已同步
*   [ ]  `index.tsx` 的 `update` 已按 action 分派，store 已补 `handleAction`（三种都实现）
*   [ ]  列宽用 `flex: 1 1 280px` / `min 280` / `max 410`，**没照抄设计稿固定宽**
*   [ ]  容器是 `flex-wrap: nowrap` + `align-items: stretch`
*   [ ]  长文本节点有 `flex` + `min-width: 0`，同行次要字段 `flex-shrink: 0`
*   [ ]  hover 提示用 antd `Tooltip` 且仅真正溢出时弹，PC 端**没用原生 `title`**
*   [ ]  单人无数据渲染成空卡片（无文案、无写死 `min-height`），一个人都没有时整块 `return null`
*   [ ]  `employeeId` 存在且为字符串，每次推送都带 `times`
*   [ ]  后端对查不到数据的员工仍下发 `entries: []`，没跳过这个人
*   [ ]  插件在 `registerListener` 里注册了 F7 监听
*   [ ]  插件读标品 pageCache 的 `currentEmployeeIds`，没自己另存员工顺序
*   [ ]  插件不继承标品 `TalentComparisonPlugin`
*   [ ]  做了多个二开控件的话，**只有一个扩展插件**，控件标识清单 + 按 key 分派，  
    共用的员工数据只查一次
*   [ ]  中文串走 `ResManager.loadKDString`，词条已写进 properties
*   [ ]  控件方案预置 SQL 已生成，ID 已查库校验无冲突
*   [ ]  若控件目录有硬编码清单的批量打包脚本，控件名已加入
*   [ ]  `npm run build` 与 lint 通过

平台侧（扩展 `hrti_talentcomparison`）：

*   [ ]  控件是 `contentflex` 的**直接子级**，没套任何 Flex 子容器
*   [ ]  控件标识唯一，且与插件 `CONTROL_KEY` 一致
*   [ ]  控件方案已绑定
*   [ ]  外边距 `0px / 20px / 20px / 20px`
*   [ ]  自定义样式已加 `$ { overflow: visible; }`
*   [ ]  控件配了中文名称（否则不进快速定位导航）
*   [ ]  插件已注册，且顺序在标品插件**之后**；**不管几个控件都只挂一个插件**
*   [ ]  页面加载、添加、移除、拖拽排序四个动作都验过

[自定义控件部署](https://developer.kingdee.com/knowledge/768125700280322048?specialId=194046086670543360&productLineId=29&isKnowledge=2&lang=zh-CN)