---
title: 人才搜索的员工详情页无法查看人才画像
category: faq
cloud: 人才发展云
tags: [人才搜索, 人才画像, 常见问题, 故障排查, 参数配置, 人才星图, tdcs_cfgparam]
aliases: [人才画像看不到, 搜索详情无画像]
author: 金蝶云社区
created: 2026-09-28
updated: 2026-09-28
source: https://vip.kingdee.com/knowledge/892352426534648832
---

# 人才搜索的员工详情页无法查看人才画像

问题：人才搜索的员工，打开详情页，但是只能看到人才档案却无法查看人才画像

![上传图片](../images/0100abd5dd379e7b4714b1f25e1038a41ba6.png)

**解决方式：**

1、检查开发平台tdcs\_cfgparam公共参数配置-列表预览-人才详情视图-参数值是否=1,2

![上传图片](../images/0100ff14d4c840ab43898b7acee5ed0eeffa.png)

![上传图片](../images/01004010bdf47ebc4828ae7febb62a3999d9.png)

![上传图片](../images/010078c58350a69b4666aec977641bef4d2c.png)

![上传图片](../images/0100d939ced8ab264f9e9313841b1ba1f9e7.png)

2、查看是否配置了人才搜索的视图，如果标品没有预置，需要添加。

![上传图片](../images/0100b2a399cfd2c24badb7f7a2f1b9e56a78.png)
