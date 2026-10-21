---
title: AI HR通用定制化开发规范
category: guide
tags: [AI HR, 定制化开发, 开发规范, 二开, SDK接口, 历史模型, 时间轴, HR模板, 导入导出, 自定义控件, HR报表]
author: 金蝶云社区
created: 2026-10-21
updated: 2026-10-21
source: https://vip.kingdee.com/knowledge/900705517435893248
---

# AI HR通用定制化开发规范

## 一、编写目的

为保证系统稳定、高效运行，加强规范AIHR定制化开发、伙伴开发，促进业务健康发展。根据公司相关规定，结合金蝶云·苍穹定制化开发规范和金蝶云·星瀚定制化开发规范，特制定金蝶AIHR定制化开发规范。

注意：此规范适用于AIHR，不适用于星瀚人力云。

## 二、适用人群

- AI HR产品 定制化开发人员
- AI HR产品 ISV伙伴开发人员

## 三、前置约定

读者已阅读完《金蝶云·苍穹定制化开发规范》，链接：[https://vip.kingdee.com/knowledge/specialDetail/218022218066869248?category=218063485690097920&id=498888207505798912&type=Knowledge&productLineId=29&lang=zh-CN](https://vip.kingdee.com/knowledge/specialDetail/218022218066869248?category=218063485690097920&id=498888207505798912&type=Knowledge&productLineId=29&lang=zh-CN)

读者已阅读完《金蝶云·星瀚定制化开发规范》，链接：[https://vip.kingdee.com/knowledge/specialDetail/218022218066869248?category=218063485690097920&id=560864077808026624&type=Knowledge&productLineId=29&lang=zh-CN](https://vip.kingdee.com/knowledge/specialDetail/218022218066869248?category=218063485690097920&id=560864077808026624&type=Knowledge&productLineId=29&lang=zh-CN)

## 四、说明

依据约束力强弱及故障敏感性，规约依次分为【强制】、【推荐】两大类。在延伸信息中，"说明"对规约做了适当扩展和解释；"正例"提倡什么样的编码和实现方式；"反例"说明需要提防的雷区，以及真实的错误案例。

## 五、内容

### 1.【强制】针对历史模型、时间轴实体的增删改必须通过SDK接口进行

SDK接口为：

- `kd.sdk.hr.hbp.business.helper.history.HisModelServiceHelper`
- `kd.sdk.hr.hbp.business.helper.timeline.TimelineServiceHelper`

**说明**：历史模型、时间轴实体的多行数据间有约束（版本约束、时间段约束），如果不通过SDK接口对数据进行增删改，会绕过约束校验，破坏数据约束，产生脏数据。

**正例**：对时间轴实体的数据进行修改时，调用 `TimelineServiceHelper` 提供的SDK接口完成，由SDK内部保证多行数据间的版本约束、时间段约束不被破坏。

```java
DynamicObject dynamicObject = BusinessDataServiceHelper.loadSingle(123L, "test_timeline1");
dynamicObject.set("name","test1");
dynamicObject.set("enddate",new Date());
TimelineServiceHelper.saveTimespans("test_timeline1",new DynamicObject[]{dynamicObject});
```

**反例**：绕开SDK接口，通过通用持久化方式直接对历史模型、时间轴实体执行新增、修改、删除操作，导致产生脏数据。

```java
DynamicObject dynamicObject = BusinessDataServiceHelper.loadSingle(123L, "test_timeline1");
dynamicObject.set("name","test1");
dynamicObject.set("enddate",new Date());
SaveServiceHelper.save(new DynamicObject[]{dynamicObject});
```

**影响领域**：HR全领域

### 2.【强制】二开在HR云下创建基础资料、单据，必须继承HR的模板，不要直接继承平台的模板

**说明**：HR的基础资料、单据模板统一增加了HR领域相关的功能和限制，所有HR的基础资料、单据都要有这些通用功能和限制。直接继承平台的模板，会缺失HR领域的通用能力和限制，导致二开单据与HR标准行为不一致。典型的HR模板有：HR基础资料全页面模板(hbp_bd_tpl_all)，HR带组织基础资料全页面模板（hbp_bd_orgtpl_all），HR带主组织单据基模板（hbp_hrbillorgtpl），HR无组织单据基模板（hbp_hrbilltpl）。

**正例**：在HR云下新建基础资料、单据时，选择继承HR对应的基础资料、单据模板。

**反例**：在HR云下新建基础资料、单据时，直接继承平台的基础资料、单据模板。

**影响领域**：HR全领域

### 3.【强制】如果有导入导出功能，必须使用HR导入导出，不要使用平台的导入导出

**说明**：HR导入导出支持了HR的权限和历史模型时间轴，平台的导入导出对HR的特性支持不够。使用平台的导入导出，会出现权限控制失效、历史模型时间轴数据不完整等问题。

配置路径：HR基础服务云 >> HR通用服务 >> HR导入导出 >> 导入导出模板配置

开发者社区链接：[https://vip.kingdee.com/knowledge/specialDetail/799733045246316032?productLineId=42&lang=zh-CN](https://vip.kingdee.com/knowledge/specialDetail/799733045246316032?productLineId=42&lang=zh-CN)

**正例**：二开单据需要导入导出能力时，使用HR导入导出进行配置和实现。

**反例**：在HR的单据上直接使用平台的导入导出实现数据导入导出功能。

**影响领域**：HR全领域

### 4.【强制】HR行政组织、HR岗位、HR员工必须使用HR的自定义控件，不要使用平台的基础资料控件

**说明**：HR行政组织、HR岗位、HR员工对应的HR自定义控件，内置了HR领域相关的控制逻辑（如数据权限、数据过滤等HR特性）；平台的基础资料控件不具备这些控制能力，会导致权限、数据范围控制失效。

完整控件列表：HR行政组织、多选HR行政组织、HR岗位、多选HR岗位、HR员工、HR多选员工

**正例**：单据中引用HR行政组织、HR岗位、HR员工数据时，使用对应的HR自定义控件。

**反例**：单据中引用HR行政组织、HR岗位、HR员工数据时，直接使用平台的基础资料控件。

**影响领域**：HR全领域

### 5.【强制】使用HR报表来配置报表功能，不要使用平台报表去开发

**说明**：

1. 平台报表是占用许可的，商务上没有预留这部分许可，使用平台报表会带来许可合规风险；
2. 平台的报表对HR的特性支持不够。

配置路径：HR基础服务云 >> HR报表管理 >> 报表管理

开发者社区链接：[https://vip.kingdee.com/knowledge/specialDetail/799733045447642624?productLineId=42&lang=zh-CN](https://vip.kingdee.com/knowledge/specialDetail/799733045447642624?productLineId=42&lang=zh-CN)

**正例**：基于HR报表能力配置实现HR相关的报表功能。

**反例**：使用平台报表开发HR相关报表功能，导致占用许可且无法支持HR特性。

**影响领域**：HR全领域

## 相关条目

- [金蝶云·苍穹定制化开发规范](https://vip.kingdee.com/knowledge/specialDetail/218022218066869248?category=218063485690097920&id=498888207505798912&type=Knowledge&productLineId=29&lang=zh-CN)
- [金蝶云·星瀚定制化开发规范](https://vip.kingdee.com/knowledge/specialDetail/218022218066869248?category=218063485690097920&id=560864077808026624&type=Knowledge&productLineId=29&lang=zh-CN)
