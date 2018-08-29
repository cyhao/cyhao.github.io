---
layout:     post
title:      MySQL sql优化之munis实现
subtitle:   查询a表存在而b表不存在的记录
date:       2018-08-29
author:     cyhao
catalog: true
tags:
    - munis
    - 派生表
---

>Oracle中求差集可以使用munis，但是MySQL中没有这个功能，如何实现是下面查询需求需要用到的，这里我们使用相应的变换实现同等功能。

