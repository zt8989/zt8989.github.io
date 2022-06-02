---
title: '后端国际化(一): kiwi-java简介'
date: 2022-06-02 12:43:17
tags:
---

## 国际化的困难在哪里？

通常项目中的异常消息等一系列文案都是中文，开发人员在开发的时候并没有考虑到国际化的情况，我们需要判断出哪些中文是注释，哪些中文是文案，哪些文案参与了业务逻辑。

## kiwi-java

kiwi-java参照[kiwi](https://github.com/alibaba/kiwi)的国际化java代码的一个解决方案。

## 流程图

![](/images/A88572FA-F9B0-4688-AE7A-D35C8A3A5EC0.png)

## 架构设计 

> kiwi-java主要包含四个核心模块：提取、过滤器、转换、翻译。

![](/images/架构设计.png)
