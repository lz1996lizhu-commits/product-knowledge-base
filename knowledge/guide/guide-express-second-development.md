---
title: 招聘服务直通车二开指引
category: guide
tags: [招聘服务直通车, 二开, 实体扩展, 数据同步, Moka, OpenAPI, 第三方招聘系统, 集成]
author: HR产品部
created: 2026-08-14
updated: 2026-08-25
cloud: 人才供应云
aliases: [招聘直通车二开, 招聘服务直通车扩展开发]
---

# 招聘服务直通车二开指引

## 版本修订记录

| 版本 | 修订内容 | 日期 | 备注 |
|------|----------|------|------|
| V1.0 | 初始拟定 | 2026-05-21 | |
| V1.1 | 补充集成背调场景 | 2026-05-22 | |
| V1.2 | 增加核心实体 | 2026-05-29 | |
| V1.3 | 增加数据流图 | 2026-05-29 | |
| V1.4 | 增加第三方招聘系统集成 | 2026-06-23 | |
| V1.5 | 增加第三方招聘系统集成-图片和附件集成 | 2026-06-23 | |
| V1.6 | 知识库同步更新（内容验证无变更） | 2026-08-17 | 来源: vip.kingdee.com |
| V1.7 | 补充Moka同步候选人二开必填字段风险提示 | 2026-08-25 | |

## 1 实体扩展

### 1.1 核心实体

招聘服务直通车涉及的核心实体清单如下：

| 序号 | 模块 | 编码 | 名称 | 说明 |
|------|------|------|------|------|
| 1 | 招聘职位 | tsrsc_jobposition | 招聘职位 | |
| 2 | 候选人 | tsrsc_appfile | 候选人 | |
| 3 | 简历 | tsrsc_candidate | 标准简历 | 简历主表，会自动同步到入职单 |
| 4 | 简历附表 | tsrsc_cancontactinfo | 联系方式 | 会自动同步到入职单 |
| 5 | 简历附表 | tsrsc_cancre | 证件信息 | 会自动同步到入职单 |
| 6 | 简历附表 | tsrsc_caneduexp | 教育经历 | 会自动同步到入职单 |
| 7 | 简历附表 | tsrsc_canlgability | 语言能力 | 会自动同步到入职单 |
| 8 | 简历附表 | tsrsc_canocpqual | 职业资格 | 会自动同步到入职单 |
| 9 | 简历附表 | tsrsc_canprework | 工作经历 | 会自动同步到入职单 |
| 10 | 简历附表 | tsrsc_canprojectexp | 项目经历 | 会自动同步到入职单 |
| 11 | 简历附表 | tsrsc_intention | 求职意向 | 不会自动同步到入职单，招聘特有字段 |
| 12 | 简历附表 | tsrsc_rsmprosk | 专业技能 | 会自动同步到入职单 |
| 13 | 录用申请 | tsrsc_offer | 录用申请单 | |
| 14 | 录用通知 | tsrsc_offernotice | 录用通知单 | |
| 15 | 入职协同 | tsrsc_induction | 入职协同申请单 | 会自动同步到入职单 |

### 1.2 扩展元数据

扩展元数据支持以下两种方式：

1. **在实体上扩展元数据增加字段**：直接在现有实体上扩展元数据并新增字段。
2. **在简历模版上增加字段**：简历及其附表需要在简历模版上增加字段。例如扩展工作经历元数据，"工作经历(tsrsc_canprework)"的元数据模版为"tsrsc_canpreworktpl"，即扩展`tsrsc_canpreworktpl`元数据。

### 1.3 新增元数据

1. 新增二开的元数据。
2. 新增简历附表元数据时，**必须继承**"标准简历附表对话框模板`tsrsc_candidatereltpl`"，否则附表的数据无法同步到后续环节。

## 2 数据同步

### 2.1 数据流图

招聘服务直通车数据从候选人/录用到入职协同再到入职单的核心流转如下：

![招聘服务直通车数据流图](../images/guide/express-second-development-v1.5-img1.png)

上图展示了四个泳道（候选人、录用、入职协同、入职单）之间的数据映射关系：

- **路线1**：录用申请单 → 入职协同单
- **路线2**：入职协同单 → 入职单
- **路线3**：标准简历及附表 → 拟入职人员及附表

### 2.2 录用同步入职协同单配置

录用申请单二开字段写入入职协同单的配置步骤：

1. **源实体**：在录用申请单元数据上添加控件并保存元数据。
2. **目标实体**：在入职协同单元数据上添加控件并保存元数据。
3. 打开`tsrbd_fieldmap`元数据列表，找到"所属场景"字段值为"候选人及其他到入职协同单"的记录，打开详情页。
4. 在字段映射分录中，将步骤1新增的控件标识填入"源字段"列，将步骤2新增的控件标识填入"目标字段"列，并将"是否启用"开关打开，点击保存。

### 2.3 入职协同单同步入职单配置

#### 2.3.1 入职协同单二开字段写入入职单

源实体配置：

- 在标准简历（含附表）实体上增加字段。
- 增加标准简历附表实体，需继承`tsrsc_candidatereltpl`。
- 在入职协同单上增加字段。

目标实体配置：

- 在拟入职人员（含附表）实体上增加字段（hcf应用下的元数据）。
- 增加拟入职人员实体。
- 在入职单上增加字段。

配置映射：

- 打开`tsrbd_fieldmap`元数据列表，找到"所属场景"为"入职协同到拟入职人员"的记录，将对应的实体字段映射配置上去。

字段映射场景清单：

| 编码 | 名称 | 源实体.名称 | 目标实体.名称 | 是否主实体 | 所属场景 |
|------|------|-------------|---------------|------------|----------|
| 4011_S | 标准简历TO拟入职人员 | 标准简历 | 拟入职人员 | √ | 入职协同到拟入职人员 |
| 4010_S | 社招候选人TO拟入职人员 | 候选人 | 拟入职人员 | √ | 入职协同到拟入职人员 |
| 4012_S | 标准简历联系方式TO拟入职人员联系方式 | 标准简历联系方式 | 拟入职人员联系方式 | | 入职协同到拟入职人员 |
| 4013_S | 证件信息TO拟入职人员证件信息 | 证件信息 | 拟入职人员证件信息 | | 入职协同到拟入职人员 |
| 4014_S | 职业资格TO拟入职人员职业资格 | 职业资格 | 拟入职人员职业资格 | | 入职协同到拟入职人员 |
| 4015_S | 专业技能TO拟入职人员专业技能 | 专业技能 | 拟入职人员专业技能 | | 入职协同到拟入职人员 |
| 4016_S | 语言能力TO拟入职人员语言能力 | 语言能力 | 拟入职人员语言技能 | | 入职协同到拟入职人员 |
| 4017_S | 项目经历TO拟入职人员项目经历 | 项目经历 | 拟入职人员项目经历 | | 入职协同到拟入职人员 |
| 4018_S | 工作经历TO拟入职人员工作经历 | 工作经历 | 拟入职人员工作经历 | | 入职协同到拟入职人员 |
| 4019_S | 教育经历TO拟入职人员教育经历 | 教育经历 | 拟入职人员教育经历 | | 入职协同到拟入职人员 |
| 5010_S | 录用申请单TO入职申请单 | 录用申请单 | 入职申请 | √ | 入职协同到入职申请单 |
| 5011_S | 入职协同单TO入职申请单 | 入职协同申请单 | 入职申请 | √ | 入职协同到入职申请单 |
| 5012_S | 社招候选人TO入职申请单 | 候选人 | 入职申请 | √ | 入职协同到入职申请单 |
| 5013_S | 标准简历联系方式TO入职申请单 | 标准简历联系方式 | 入职申请 | √ | 入职协同到入职申请单 |
| 5014_S | 证件信息TO入职申请 | 证件信息 | 入职申请 | √ | 入职协同到入职申请单 |
| 5015_S | 标准简历TO入职申请单 | 标准简历 | 入职申请 | √ | 入职协同到入职申请单 |
| 6010_S | 社招候选人TO入职协同单 | 候选人 | 入职协同申请单 | √ | 候选人及其他到入职协同单 |
| 6011_S | 录用申请单TO入职协同单 | 录用申请单 | 入职协同申请单 | √ | 候选人及其他到入职协同单 |

#### 2.3.2 直通车二开附件同步到入职单

可以在`tsrsc_induction`的"发起入职"操作上增加操作插件，在操作插件的`afterExecuteOperationTransaction`中写入到入职单相关的附件。

关键逻辑：

- `tsrsc_induction`的`appfile`字段 → 根据`appfile`再找`tsrsc_appfile`的主键。
- 入职协同单元数据`tsrsc_induction`上有入职单ID字段`onbrdinfo`，可以通过该字段获取或更新入职单的相关附表或附件。

## 3 第三方系统集成

### 3.1 Moka标准集成方案

**开闭原则**：所有二开扩展需要在"前置"或"后置"节点处理，禁止修改非前置或后置的服务流程，避免标品升级覆盖二开的内容。

![Moka同步候选人前置处理服务流程变量](../images/guide/express-second-development-v1.5-img2.png)

#### 3.1.1 招聘职位

##### 3.1.1.1 增加字段

**业务场景**：在招聘职位或候选人（含简历附表）增加字段。

**扩展方案**：通常都在"预置_Moka_同步候选人前置处理 `tsrsc_moka_application_before_pre`"服务流程处理二开的内容。

关键流程变量：

- `v_tsrsc_moka_getjobs_r`：Moka获取到的招聘职位。
- `v_tsrsc_moka_getapplications_r`：Moka获取到的候选人及其附表集合的数据。
- `v_allinfomap`：直通车的候选人及其附表对象。
- `v_tsrsc_jobposition`：直通车的招聘职位。

在`tsrsc_moka_application_before_pre`节点中，给`v_allinfomap`、`v_tsrsc_jobposition`对象赋值，即可将二开的字段传到直通车的对象上。

##### 3.1.2 候选人

参照"招聘职位"章节的扩展方式处理。

> **风险提示**：在候选人主实体`tsrsc_appfile`/`tsrsc_appfilesocial`或简历附表上新增**必填字段**时，需确保Moka同步前置处理服务流程`tsrsc_moka_application_before_pre`能够为该字段赋值。若Moka返回的字段映射未覆盖该必填字段，保存候选人时会因数据校验不通过导致整单回滚，表现为Moka端显示导入成功、金蝶端无候选人数据。排查日志可见`EntityOperateService.execute(..., save)`返回`successIds = null`及`Msg数据校验发现错误`。建议将二开字段设为非必填，或在服务流程中补充默认值/字段映射。

#### 3.1.3 背调信息

**业务需求**：将背调信息的附件集成到招聘服务直通车，以下以背调所有的附件都集成到招聘服务直通车的候选人详情为例。

实施步骤：

1. 增加"背调元数据"。
2. 在元数据上增加基础资料"候选人(`tsrsc_appfilesocial`)"。
3. 在元数据上增加附件面板。
4. 在候选人元数据上增加Tab页签，并采用"页面嵌入控件"将背调元数据嵌入进来。

![页面嵌入控件](../images/guide/express-second-development-v1.5-img3.png)

5. **数据集成**：在服务流程`tsrsc_moka_application_after_pre`（预置_Moka_同步候选人后置处理）完成数据集成。

脚本说明：`sya5_bdfj`为二开的元数据，`id`、`sya5_appfile`、`sya5_url`为`sya5_bdfj`的字段。

示例脚本：

```javascript
var v_sya5_exam = v_v2applicationJson[0].examInfo;
if(v_sya5_exam is Not Null){
  var v_surveys = v_sya5_exam.surveys;
  if(v_surveys is Not Null){
    var v_sya5_url = String.FormatJson(v_surveys);
    var entity = "sya5_bdfj";
    var actions = ['save'];
    var data = {"id":new_int_id(),"sya5_appfile":v_appfileid,"sya5_url":v_sya5_url}; 
    var judgeFields = {'$':['id']};
    var result = $action(r_KDIERP, entity,actions,data,judgeFields);
    return result;
	}
}
```

6. 增加背调的Save操作插件，在操作的`afterExcuteOperationTransaction`中执行附件从Moka下载并上传到苍穹平台的附件面板。同时将之前的记录`sya5_bdfj`标记为已删除/失效。

#### 3.1.4 集成脚本帮助文档

![集成平台脚本帮助手册](../images/guide/express-second-development-v1.5-img4.png)

集成平台提供脚本函数、运算符、过程控制等帮助文档，可在"集成管理 > 帮助手册"中查阅。

#### 3.1.5 Moka接口文档

参考链接：[Moka API文档](https://www.mokahr.com/docs/api/?shell#v2)

### 3.2 第三方招聘系统集成方案

**适用场景**：在第三方异构系统里集成招聘服务直通车，集成逻辑做在第三方招聘系统侧。

#### 3.2.1 使用OpenAPI数据集成

##### 3.2.1.1 业务数据集成

通过招聘服务直通车OpenAPI实现业务数据集成。完整OpenAPI清单见本章末尾的"OpenAPI清单"。

##### 3.2.1.2 附件集成

使用平台通用接口进行附件集成，包含图片、附件：

| 序号 | 接口名称 | URL | 备注 |
|------|----------|-----|------|
| 1 | 上传图片并绑定到单据 | /v2/frame/image/uploadImage | |
| 2 | 下载图片字段的文件 | /v2/frame/image/downloadImage | |
| 3 | 上传附件 | /v2/frame/attachment/uploadFile | |
| 4 | 下载附件 | /v2/frame/attachment/download | |

#### 3.2.2 候选人状态回传

**方案1：订阅状态变更（实时）**

适用于对实时性要求高的场景。在"业务事件中心"订阅`tsrsc_induction.chg_event`，可以监听入职协同单的状态（`inductionstatus`）变化，包括"已取消入职、已入职、已转正、已离职"。`tsrsc_induction`有候选人ID和第三方候选人ID，可以进行关联。

**方案2：入职状态轮询查询**

使用OpenAPI查询，对应API为`/v2/tsrsc/tsrsc_induction/query`，参数为招聘服务直通车候选人ID。

## 4 OpenAPI清单

### 4.1 业务数据接口

| 序号 | 接口名称 | URL | 备注 |
|------|----------|-----|------|
| 1 | 查询社招候选人 | /v2/tsrsc/tsrsc_appfilesocial/query | |
| 2 | 保存社招候选人 | /v2/tsrsc/tsrsc_appfilesocial/save | |
| 3 | 查询简历联系方式 | /v2/tsrsc/tsrsc_cancontactinfo/query | |
| 4 | 保存简历联系方式 | /v2/tsrsc/tsrsc_cancontactinfo/save | |
| 5 | 查询简历证件信息 | /v2/tsrsc/tsrsc_cancre/query | |
| 6 | 保存简历证件信息 | /v2/tsrsc/tsrsc_cancre/save | |
| 7 | 查询简历主信息 | /v2/tsrsc/tsrsc_candidate/query | |
| 8 | 保存简历主信息 | /v2/tsrsc/tsrsc_candidate/save | |
| 9 | 查询简历教育经历 | /v2/tsrsc/tsrsc_caneduexp/query | |
| 10 | 保存简历教育经历 | /v2/tsrsc/tsrsc_caneduexp/save | |
| 11 | 查询简历语言能力 | /v2/tsrsc/tsrsc_canlgability/query | |
| 12 | 保存简历语言能力 | /v2/tsrsc/tsrsc_canlgability/save | |
| 13 | 查询简历职业资格 | /v2/tsrsc/tsrsc_canocpqual/query | |
| 14 | 保存简历职业资格 | /v2/tsrsc/tsrsc_canocpqual/save | |
| 15 | 查询简历工作经历 | /v2/tsrsc/tsrsc_canprework/query | |
| 16 | 保存简历工作经历 | /v2/tsrsc/tsrsc_canprework/save | |
| 17 | 查询简历项目经历 | /v2/tsrsc/tsrsc_canprojectexp/query | |
| 18 | 保存简历项目经历 | /v2/tsrsc/tsrsc_canprojectexp/save | |
| 19 | 查询简历求职意向 | /v2/tsrsc/tsrsc_intention/query | |
| 20 | 保存简历求职意向 | /v2/tsrsc/tsrsc_intention/save | |
| 21 | 查询面试评价 | /v2/tsrsc/tsrsc_intervieweval/query | |
| 22 | 保存面试评价 | /v2/tsrsc/tsrsc_intervieweval/save | |
| 23 | 查询招聘职位 | /v2/tsrsc/tsrsc_jobposition/query | |
| 24 | 保存招聘职位 | /v2/tsrsc/tsrsc_jobposition/save | |
| 25 | 查询简历专业技能 | /v2/tsrsc/tsrsc_rsmprosk/query | |
| 26 | 保存简历专业技能 | /v2/tsrsc/tsrsc_rsmprosk/save | |
| 27 | 根据行政组织查询招聘管理组织 | /v2/tsrbd/adminorgstragy/query | |
| 28 | 查询基础资料数据 | /v2/tsrbd/tsrbd_baseData/query | |
| 29 | 获取元数据实体字段属性 | /v2/tsrbd/tsrbd_entitymetadata/query | |
| 30 | 入职协同状态查询 | /v2/tsrsc/tsrsc_induction/query | |

### 4.2 通用附件接口

| 序号 | 接口名称 | URL | 备注 |
|------|----------|-----|------|
| 1 | 上传图片并绑定到单据 | /v2/frame/image/uploadImage | |
| 2 | 下载图片字段的文件 | /v2/frame/image/downloadImage | |
| 3 | 上传附件 | /v2/frame/attachment/uploadFile | |
| 4 | 下载附件 | /v2/frame/attachment/download | |

## 相关条目

- [招聘服务直通车外部系统集成方案](./guide-integration-external-recruitment.md)
- [Moka招聘系统连接配置](./guide-moka-integration.md)
- [招聘服务直通车产品概述](../product/feature-recruitment-express.md)
- [外部招聘录用流程](../business/process-external-recruitment.md)
