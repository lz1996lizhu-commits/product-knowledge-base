---
title: 15分钟内完成一个人才画像卡片开发
category: guide
cloud: 人才发展云
tags: [人才画像, 画像卡片, 二开, 自定义控件, 苍穹表单, 控件方案, 前端控件, 后端插件]
aliases: [人才画像卡片二开, 画像卡片自定义开发, 教育背景卡片]
author: 金蝶云社区
created: 2026-08-20
updated: 2026-08-20
source: https://vip.kingdee.com/knowledge/878311629627593472
---

# 人才画像卡片二开实操步骤

按实际操作顺序记录，以「教育背景」卡片为例。

配套示例代码在本包 `sample/` 下：`sample/frontend/`（前端控件工程，已剔除 `node_modules` 与构建产物）、`sample/backend/`（后端插件模板 + 控件方案预置 SQL）。设计稿产物示例在 `figma/`。

阅读约定：`<二开工程>`、`<你的包>`、`<你的控件目录>` 等占位符替换为实际值。

全流程六步，前三步在 IDE 里让 AI 做，后三步在苍穹平台手工配：

| 步骤 | 做什么 | 在哪做 |
| --- | --- | --- |
| step1 | 触发技能，按 Figma 截图 + CSS 生成前端控件与后端插件 | IDE |
| step2 | 放入图标让 AI 替换，生成控件方案预置脚本 | IDE |
| step3 | 调整后端取数逻辑（step1 已给全可省） | IDE |
| step4 | 新增卡片表单，挂控件与插件 | 苍穹设计器 |
| step5 | 画像视图配置里加入这张卡片 | 平台配置页 |
| step6 | 从画像列表进员工画像验证 | 平台页面 |

---

## step1 触发技能生成前端控件与后端插件

### 1.1 先准备好设计稿产物

Figma 里选中卡片组件，导出两样东西放到一个目录（本例 `figma/`）：

- **截图**：2x PNG，有数据态即可
- **CSS**：右侧面板 `Inspect` → 复制 CSS → 存成 txt

空数据态不用导。技能生成的控件工程自带 `Empty` 组件（插画 + 「暂无数据」），直接引用即可，不需要按设计稿另画一套。

**CSS 必须导。** 有它才能把颜色、字号、字重、行高、字距、圆角、渐变、间距按原值落地，不用靠肉眼估。哪些照搬、哪些不照搬：

| 照搬 | 不照搬 |
| --- | --- |
| 色值、字号、字重、行高、letter-spacing | `position: absolute` 及 `left/top` |
| 圆角、阴影、边框、渐变 | 固定 `width` / `height` |
| padding、gap | Figma 自动生成的 Frame 层级 |

Figma 稿是定宽绝对定位（本例 521px），要改成流式布局：宽度由平台容器决定，内边距按设计稿数值换算。设计稿里 `display: none` 的元素不用实现。

### 1.2 触发技能

技能在本包 `skill/custom-control-full-stack-generic/`，先按 `skill/README.md` 装到 `~/.kiro/skills/` 下，否则下面的命令用不了。

调用 `custom-control-full-stack-generic`，把这些信息一次给全，AI 就不用反复问：

```
/custom-control-full-stack-generic
根据 Figma 导出的图片和 CSS 为我生成自定义控件，严格还原颜色、间距、字体、字重等样式。
控件标识 secDevTalentProfileEdu，使用 React + antd。控件生成在 <你的控件目录> 下。
```

技能会先确认这几项（已给的不重复问）：

| 项 | 本例取值 | 说明 |
| --- | --- | --- |
| 控件名 | `secDevTalentProfileEdu` | 后面预置 SQL 的 `FSCHEMAID` 必须与它完全一致 |
| 前缀约定 | `secDev`（demo） | 团队有云产品前缀规范则按规范 |
| 框架 | React + Antd | |
| ISV | `kingdee` | |
| MODULE_ID | `hr` | |
| 目标目录 | 你指定的目录 | 技能不会猜，必须给 |

**取数逻辑在这一步就一起说清，能省掉 step3。** 例如实体标识、过滤字段、需要哪些字段、排序分组规则、派生数据怎么算。不说的话技能会先用静态数据打通链路。

### 1.3 技能产出

前端工程（自动完成）：

- 从离线模板复制，不是拷现有项目改
- 改 `package.json` 的 `name`
- 改 `app.config.js` 的 `APP_NAME` / `ISV` / `MODULE_ID`
- 把 `src/styles/variable.less` 里 `react_demo` 变量前缀换成 APP_NAME
- `npm install`、拷 `CLAUDE.md`

后端插件（本包 `sample/backend/`，两个文件）：

- `AbstractExtPortraitCardPlugin`：二开自己的基类
- `EduBackgroundCardPlugin`：卡片插件

**注意后端插件不继承标品的 `AbstractPortraitCardPlugin`。** 原因：标品类随版本演进，签名或行为变化会直接打断二开；跨工程继承会把二开绑死在标品版本上。基类只依赖平台 `kd.bos.*` 和标品公开的约定（控件标识、页面参数 key），标品升级不影响二开编译。

基类提供的能力：

| 能力 | 说明 |
| --- | --- |
| `preOpenForm` | 设 VIEW 状态，卡片是只读展示 |
| `getEmployeeId()` | 先取自身页面参数，再取父视图参数，参数 key 是 `employee` |
| `pushData(Map)` | 推数据，自动补 `times` 时间戳与 `cardTitle` |
| `pushEmpty()` | 推 `{ empty: true }` |
| `getCardTitle()` | 解析用户在画像视图配置里改过的标题 |

`times` 是必要的：内容相同的两次 `setData` 前端可能识别不到变化，补时间戳保证每次触发 update。

基类**不做 pageCache 缓存**——完全自建的卡片数据由自己插件独占推送，缓存没意义。只有「扩展标品卡片、在标品已推数据上追加字段」才需要缓存，见附录 A。

### 1.4 技能的两个提示

技能跑完会给出：

1. **提示要换 icon**：识别到设计稿里有图标资源，让你把源文件放进 `src/assets/icons/`。在你确认资源到位前，AI 用占位实现顶着，不会硬编码不存在的路径。
2. **问询是否生成控件方案预置脚本**：这步不做，平台里选不到控件。答「是」进 step2。

### 1.5 顺手确认数据契约

契约是前后端唯一的对接口径，`src/types/edu.ts`：

```ts
export interface EduItem {
  id: string          // 列表 key
  school: string      // 学校名称
  highest?: boolean   // 是否最高学历，true 时展示「最高学历」标签
  degree?: string     // 学历
  major?: string      // 专业
  desc?: string       // 描述行，不传则用 degree · major 拼接
  period?: string     // 起止时间，后端格式化后下发，如 2010 - 2013
}

export interface EduCardData {
  empty?: boolean     // 后端判定无数据时 true，前端渲染空态
  cardTitle?: string  // 卡片标题，平台配置优先，缺省用前端词条
  list?: EduItem[]
}
```

三个约定值得固化：

- `empty` 让后端判定，前端不猜「列表为 0 是没数据还是没查」
- `period` 由后端格式化好，日期格式化涉及多语言和平台日期格式，放后端更稳
- `cardTitle` 可缺省，用户改过标题才下发

### 1.6 本地验一遍

```bash
npm run mock      # 一个终端
npm run dev:ram   # 另一个终端
```

mock 业务数据单独放 `mock/data/init.js`，由 `mock/data/index.js` 组装进 `initMock`，不要塞公共文件。至少过这几种：有数据、空数据（`empty: true`）、条目溢出（看列表是否内部滚动、卡片是否没被撑破）、超长文本、多语言切换、主题色切换。

需要连真实环境时用 `npm run dev`，在测试环境页面 URL 后拼 `&kdcus_cdn=http://localhost:<DEV_RAM_PORT>`，平台会来本地拉控件资源。端口冲突改 `app.config.js` 的 `DEV_RAM_PORT` / `DEV_CACHE_PORT` / `MOCK_PORT`。

---

## step2 替换图标 + 生成控件方案预置脚本

### 2.1 放图标并让 AI 替换

把设计稿导出的 svg 放到 `src/assets/icons/`，然后告诉 AI 替换。

替换前（占位实现）：

![替换图标前](../images/guide/portrait-card-icon-before.png)

替换后：

![替换图标后](../images/guide/portrait-card-icon-after.png)

**一个必踩的坑**：设计稿导出的 svg 常常自带背景底色和圆角。本例这个是 48x48、带 `linear-gradient(134.2deg, #A9DCF8, #B3A0F6)` 底和 8px 圆角。这时要把容器上的 `background` 撤掉，只留尺寸约束，否则渐变叠两层、颜色偏。

```less
// 图标 svg 自带渐变底色与 8px 圆角，此处只固定尺寸
.icon {
  flex: none;
  width: 48px;
  height: 48px;
}
```

svgr 模板里已配好，直接当组件用：

```tsx
import GraduationIcon from '@/assets/icons/GraduationIcon.svg'

<GraduationIcon className={Style.icon} />
```

### 2.2 生成控件方案预置脚本

指定输出目录让 AI 生成。三件事按序做：生成 ID → 查库校验 → 写 SQL。

**生成 ID**：`FID` 是 19 位正 long，`FPKID` 是 12 位平台风格字符串，**必须用雪花算法工具生成，不能手写或复制改写**。雪花算法高位是当前毫秒时间戳，"现在"生成的 ID 必然大于历史上任何已生成的 ID，天然不冲突；编造的数字保证不了唯一，会导致预置数据覆盖或主键冲突。技能内置了零依赖工具，有 JDK 即可跑：

```bash
javac -encoding UTF-8 SnowflakeIdGen.java
java SnowflakeIdGen 3
```

源文件含 UTF-8 中文注释，编译必须带 `-encoding UTF-8`，否则默认 GBK 的 Windows 环境会失败。

一组 ID 只用于一条记录。多语言表每种语言一行、各自一个 `FPKID`，所以本例需要 1 个 FID + 2 个 FPKID。

**查库校验**：概率上不冲突，仍要写入前查一次，防他人已提交的预置数据。元数据类预置数据在 meta 库：

```sql
SELECT FID FROM T_META_CTLSCHEMA
 WHERE FID = <生成的FID> OR FSCHEMAID = '<你的控件标识>';
SELECT FPKID FROM T_META_CTLSCHEMA_L
 WHERE FPKID IN (<生成的FPKID列表>);
```

除主键外还要校验业务唯一键 `FSCHEMAID` 不与已有控件重复。两条都返回空才可落库。

**写 SQL**：DELETE + INSERT 幂等，可重复执行。

```sql
DELETE FROM T_META_CTLSCHEMA_L WHERE FID = 1539508029412610048;
DELETE FROM T_META_CTLSCHEMA WHERE FID = 1539508029412610048;
DELETE FROM T_META_CTLSCHEMA_L WHERE FID IN (
  SELECT FID FROM T_META_CTLSCHEMA WHERE FSCHEMAID = 'secDevTalentProfileEdu');
DELETE FROM T_META_CTLSCHEMA WHERE FSCHEMAID = 'secDevTalentProfileEdu';

INSERT INTO T_META_CTLSCHEMA
    (FID, FSCHEMAID, FPREVIEWIMG, FVERSION, FATTACHMENTCOUNT, FISVID, FMODULEID)
VALUES
    (1539508029412610048, 'secDevTalentProfileEdu', ' ', '1', 0, 'kingdee', 'hr');

INSERT INTO T_META_CTLSCHEMA_L (FPKID, FID, FLOCALEID, FSCHEMANAME)
VALUES ('4X4PKG68I6O/', 1539508029412610048, 'zh_CN', '人才画像教育背景二开demo');

INSERT INTO T_META_CTLSCHEMA_L (FPKID, FID, FLOCALEID, FSCHEMANAME)
VALUES ('4X4PKG6C3MWI', 1539508029412610048, 'zh_TW', '人才畫像教育背景二開demo');
```

字段对应：

| 字段 | 来源 |
| --- | --- |
| `FSCHEMAID` | 前端 `app.config.js` 的 `APP_NAME`，**必须一致** |
| `FMODULEID` | `MODULE_ID` |
| `FISVID` | `ISV` |
| `FSCHEMANAME` | 控件方案名称，即设计器里选控件时显示的名字 |

`FPKID` 里出现 `+` `/` `=` 是正常的，它是 long 主键的 base64 风格编码，字符集 `0-9A-Z+/=`，单引号包裹能正常入库，不用因此重新生成。

文件最终放到 `<二开工程>/datamodel/.../preinsdata/`，按工程版本号命名规范命名。

---

## step3 调整后端取数逻辑

**step1 已经把取数逻辑说清、AI 生成了正确实现的话，这步可省。**

技能默认给的是静态数据（打通链路用），替换为真实查询时这几条是硬性的：

- `QueryServiceHelper.query` 返回 `DynamicObjectCollection`，不是 `DynamicObject[]`
- 基础资料字段不会自动带出动态对象，只能用点路径取值，如 `education.name`
- 多选基础资料按分录路径过滤（如 `xxx.fbasedataid.id`），结果会展开成多行，分组时直接取扁平化路径值
- 需要按多条记录分别查关联数据时，先收集 ID 用 `QCB.in` 一次性批量查，再在内存里分组，**不要循环查库**
- 日期格式化用 `HRDateTimeUtils`，禁用 `SimpleDateFormat`（非线程安全）

插件主流程：

```java
public class EduBackgroundCardPlugin extends AbstractExtPortraitCardPlugin {

    @Override
    public void afterBindData(EventObject e) {
        super.afterBindData(e);

        Long employeeId = getEmployeeId();
        if (employeeId == null) {
            logger.info("EduBackgroundCardPlugin: employeeId not found, push empty.");
            pushEmpty();
            return;
        }

        List<Map<String, Object>> list = queryEduList(employeeId);
        if (list.isEmpty()) {
            pushEmpty();
            return;
        }

        Map<String, Object> data = new HashMap<>(4);
        data.put("list", list);
        pushData(data);
    }
}
```

**推送的 Map 字段名必须与前端 interface 逐一对应**，写之前先把前端类型定义读一遍。

多语言：所有面向用户的中文串走 `ResManager.loadKDString("教育背景", "EduBackgroundCardPlugin_0", "<你工程的模块标识>")`，第 2 个参数是 `类名_序号`，序号从 0 递增。每个词条都要写进工程的 `resources/<工程名>_zh_CN.properties`。英文标识符、日志、纯数字编码不需要处理。

---

## step4 在 hrti 扩展应用下增加卡片

进苍穹设计器，在 hrti 的扩展应用下新增卡片表单。

![在hrti扩展应用下新增卡片-1](../images/guide/portrait-card-add-form-1.png)

![在hrti扩展应用下新增卡片-2](../images/guide/portrait-card-add-form-2.png)

五个动作，缺一个卡片就出不来或没数据：

### 4.1 继承 `bos_card_template_new`

新建的卡片表单要继承平台卡片模板 `bos_card_template_new`。**画像视图配置页是靠继承路径里是否包含小部件基类来筛选可选卡片的**，继承不对，step5 里选不到这张卡片。

### 4.2 隐藏卡片标题

平台卡片模板自带一个标题区。我们的自定义控件内部已经画了标题（含设计稿的字号字重字距），两个标题会重复，所以把模板自带的标题隐藏掉。

标题文案统一由控件内部渲染：用户在画像视图配置里改过标题就用配置值（基类 `getCardTitle()` 从 pageCache 的 `cardTitleMapEntry` 解析后下发），没改就用前端词条兜底。

### 4.3 增加自定义控件

往画布拖一个自定义控件。**控件标识保持默认的 `customcontrolap`**，改了的话插件里要覆盖 `getControlKey()`。

### 4.4 设置自定义控件高度 100%

这一步直接决定卡片高度表现。**控件高度设成 100%**，让控件宿主容器撑满卡片。

配套的是前端侧也要撑满，两边必须一致：

```less
.card {
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  height: 100%;      // 关键
  overflow: hidden;  // 保证圆角不被内层滚动条切出直角
  padding: 24px 0 0;
}

.list {
  flex: 1 1 auto;
  min-height: 0;
  overflow-y: auto;
  padding: 0 32px 24px;
}

.emptyWrap {
  display: flex;
  flex: 1 1 auto;
  align-items: center;
  justify-content: center;
  min-height: 0;
}
```

**为什么前端是 `height: 100%` 而不是 `min-height: 100%`：** 平台宿主容器的高度不是 CSS 显式声明的（靠 flex / 定位撑开），`min-height` 的百分比基准取不到值会退化成内容高度，`height` 才会被撑开。

另外 `overflow: hidden` 之后内容超高会被裁，所以列表必须自己是滚动区；`min-height: 0` 不写，flex 子项不会收缩、滚动条出不来。

### 4.5 绑定控件方案

自定义控件的控件方案选 step2 预置的那个。下拉里能看到 `人才画像教育背景二开demo`，说明预置 SQL 生效了。选不到就回查 `FSCHEMAID` 与 `APP_NAME` 是否一致、SQL 是否执行成功。

### 4.6 注册后端插件

在卡片表单的插件列表里挂上你的插件：

![给二开卡片绑定插件](../images/guide/portrait-card-bind-plugin.png)

**这步漏掉的表现是：控件能加载、卡片框架能出来，但永远空态**，因为没人给它推数据。

---

## step5 画像视图配置增加二开卡片

进画像视图配置（PC 端），把卡片加进视图：

![画像视图配置-PC端 增加二开卡片](../images/guide/portrait-card-view-config.png)

**这个页面里所有卡片都显示「暂无数据」是正常的**，包括标品卡片。配置页只负责编排卡片位置和标题，不会传 `employee` 参数给卡片，插件取不到 employeeId 就走 `pushEmpty()`。

判断方法：看旁边的标品卡片。它们也空 → 正常配置态；只有你的卡片空 → 才是你的问题。

配置页里能确认的正常信号：

- 卡片标题显示的是你配置的名字 → 控件方案生效、控件加载成功、`cardTitle` 解析并下发成功
- 空态样式撑满容器、和标品卡片一样高 → step4.4 的高度设置生效

---

## step6 从画像列表验证实际效果

全景人才画像列表点员工超链进入画像页：

![全景人才画像列表点击超链查看画像](../images/guide/portrait-card-list-verify.png)

这时页面带 `employee` 参数，插件能取到 employeeId，卡片渲染真实数据。到这里整条链路走完。

---

## 排错对照表

| 现象 | 可能原因 |
| --- | --- |
| 设计器控件方案下拉里没有你的控件 | step2 预置 SQL 没执行，或 `FSCHEMAID` 与 `APP_NAME` 不一致 |
| 画像视图配置里选不到这张卡片 | step4.1 没继承 `bos_card_template_new` |
| 卡片框架出来了但永远空态 | step4.6 没挂插件；或控件标识不是 `customcontrolap` 且插件没覆盖 `getControlKey()` |
| 配置页里所有卡片都空态 | 正常，配置页不传 `employee`（对照标品卡片是否也空） |
| 卡片里出现两个标题 | step4.2 没隐藏模板自带标题 |
| 卡片高度比邻居矮 | step4.4 控件高度没设 100%，或前端用了 `min-height: 100%` |
| 卡片白屏 | 控件静态资源没部署，或前端运行时报错（看浏览器控制台） |
| 前端收不到 `setData` | 推送字段名与前端 interface 不匹配；或 `getEmployeeId()` 返回 null 走了空态 |
| 界面显示词条 key 名 | 词条未注册且没做 FALLBACK 兜底 |
| 图标颜色偏 | svg 自带渐变底，容器又叠了一层 `background` |

## 收尾自查

IDE 侧：

- [ ] `app.config.js` 的 `APP_NAME` / `ISV` / `MODULE_ID` 已改，`variable.less` 变量前缀已同步
- [ ] 图标已替换，容器没有叠加多余的 `background`
- [ ] 若控件目录存在硬编码清单的批量打包脚本，控件名已加入
- [ ] 空态、溢出滚动、超长文本、多语言、主题色都验过
- [ ] `npm run build` 与 lint 通过
- [ ] 后端插件不继承标品类，推送字段名与前端 interface 逐一对应
- [ ] 中文串走 `ResManager.loadKDString`，词条已写进 properties
- [ ] 控件方案预置 SQL 已生成，ID 已查库校验无冲突

平台侧：

- [ ] 卡片继承 `bos_card_template_new`
- [ ] 模板自带标题已隐藏
- [ ] 自定义控件高度 100%，控件方案已绑定
- [ ] 后端插件已注册
- [ ] 卡片已加入画像视图，画像工作台里数据正常

---

## 附录 A 扩展标品卡片（另一种场景）

如果目标不是新增卡片，而是给标品已有卡片补字段，做法完全不同：**不新建卡片表单**，而是把你的插件注册在标品插件**之后**（插件按元数据顺序串行执行），走「读标品缓存 → 追加字段 → 回写缓存 → setData」：

```java
String json = getPageCache().getBigObject("customcontrolap");
if (json == null || json.isEmpty()) {
    return;
}
Map<String, Object> data = SerializationUtils.fromJsonString(json, Map.class);
if (Boolean.TRUE.equals(data.get("empty"))) {
    return;
}
data.put("yourField", value);
getPageCache().putBigObject("customcontrolap", SerializationUtils.toJsonString(data));
getControl("customcontrolap").setData(data);
```

三个坑：缓存没就绪时不要推（会把标品字段整份顶掉）；`empty=true` 直接返回；标品若有增量刷新，要在对应 `customEvent` 里同样补一次追加，否则你的字段会被覆盖。

---

## 附录 B 用苍穹表单做卡片（不写前端控件）

正文那条路是「自定义控件」，UI 完全由前端代码画，适合设计稿复杂、标准控件拼不出来的卡片。如果卡片就是常规的字段罗列、表格、分录，**没必要写前端控件**，直接用苍穹表单做更快：不用建前端工程、不用打包部署、不用预置控件方案。

两条路的取舍：

| | 自定义控件（正文） | 苍穹表单（本附录） |
| --- | --- | --- |
| UI 来源 | 前端代码，可 100% 还原设计稿 | 标准控件拖拉拽，受平台样式约束 |
| 前端工程 | 要建、要打包部署 | 不要 |
| 控件方案预置 SQL | 要 | 不要 |
| 数据下发方式 | `CustomControl.setData()` 推 JSON | `getModel().setValue()` 写字段值 |
| 适合场景 | 设计稿复杂、有自定义交互/图表 | 常规字段展示、列表、分录 |

### B.1 新建卡片表单

在 hrti 扩展应用下新建动态表单，**直接继承 `hrti_profilecardtpl`**（画像卡片公共父模板）。

继承它就自动满足了画像卡片的两个前提：继承路径里含小部件基类（画像视图配置页才筛得到）、卡片外框样式与标品一致。

移动端卡片继承 `hrti_profilecardtpl_m`。

### B.2 加字段、画界面

**这一步和开发普通动态表单完全一样**：往画布拖标准控件、加字段、设布局。

要点：

- 字段标识自己定，后端插件里靠标识 `setValue`，两边保持一致
- 卡片是只读展示，字段设成不可编辑
- 卡片高度跟着模板走，不需要像自定义控件那样单独设 100%
- 标题就用模板的，不用隐藏

### B.3 开发插件（继承二开模板插件）

插件仍然**继承你自己工程的 `AbstractExtPortraitCardPlugin`**，不继承标品类，原因同正文。

```java
public class EduBackgroundFormCardPlugin extends AbstractExtPortraitCardPlugin {

    private static final String ENTRY_EDU = "entryentity";

    @Override
    public void afterBindData(EventObject e) {
        super.afterBindData(e);

        Long employeeId = getEmployeeId();
        if (employeeId == null) {
            logger.info("EduBackgroundFormCardPlugin: employeeId not found.");
            return;
        }

        List<Map<String, Object>> list = queryEduList(employeeId);
        if (list.isEmpty()) {
            getView().setVisible(Boolean.FALSE, ENTRY_EDU);
            return;
        }

        fillEntry(list);
    }

    private void fillEntry(List<Map<String, Object>> list) {
        AbstractFormDataModel model = (AbstractFormDataModel) getModel();
        model.beginInit();
        model.deleteEntryData(ENTRY_EDU);
        model.batchCreateNewEntryRow(ENTRY_EDU, list.size());
        for (int i = 0; i < list.size(); i++) {
            Map<String, Object> item = list.get(i);
            model.setValue("school", item.get("school"), i);
            model.setValue("degree", item.get("degree"), i);
            model.setValue("major", item.get("major"), i);
            model.setValue("period", item.get("period"), i);
        }
        model.endInit();
        getView().updateView(ENTRY_EDU);
    }
}
```

注意点：

- **批量写分录**要包在 `beginInit()` / `endInit()` 里，最后 `updateView` 一次
- **空数据**没有前端 `empty` 约定，自己决定表现
- **多语言**同样走 `ResManager.loadKDString`
- 查数规范与正文 step3 完全一致

### B.4 绑定插件

在卡片表单的插件列表里注册这个插件，做法与正文 step4.6 一样。

### B.5 后续步骤同正文

- **step5 画像视图配置增加二开卡片**
- **step6 从画像列表验证**

### B.6 本附录的自查

- [ ] 卡片表单继承 `hrti_profilecardtpl`（移动端 `hrti_profilecardtpl_m`）
- [ ] 字段标识与插件里 `setValue` 用的标识一致
- [ ] 插件继承自己工程的 `AbstractExtPortraitCardPlugin`，不是标品类
- [ ] 分录批量写包在 `beginInit` / `endInit` 里
- [ ] 空数据有明确表现
- [ ] 中文串走 `ResManager.loadKDString`，词条已写进 properties
- [ ] 插件已在卡片表单上注册
- [ ] 卡片已加入画像视图，画像工作台里数据正常

## 相关条目

- [人才储备池整体介绍](../product/feature-talent-reserve-pool.md)
