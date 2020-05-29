---
author: jieteki
categories:
- 技术文章
date: 2016-12-17T13:18:53+08:00
description: ""
keywords:
- android, recyclerview, swipe_refresh_layout
title: Load more and pull refresh
url: ""
---


列表的下拉刷新是一个常见的场景， Android 提供了 SwipeRefreshLayout 这个布局来实现。列表到顶了再下拉，就会有个刷新的小圆圈表示在加载，这个圆圈是可以定制的。与此同时触发一个回调 OnRefresh 方法。当加载完成后，你需要手动将这个圆圈消除。
把 ListView 套在这个 Layout 里面，这个 ListView 就具备下拉刷新的功能了。在更复杂的列表结构中，ListView 不能优雅胜任（或是性能不好？），此时 Android 提供了 RecyclerView 这个“item可回收”的列表容器来支持。RecyclerView 与对应的 RecyclerView.Adapter 组合可以做到 ListView 或者 GridView 的效果。

<!--more-->

## 1. 我们要做什么

RT, 我们要做列表的下拉刷新和上拉加载更多。下拉刷新 SwipeRefreshLayout 妥妥的支持了，接下来我们看如何做上拉的操作。从 layout 开始吧。

## 2. 创建 layout

一个简单的 layout，SwipeRefreshLayout 套在 RecyclerView 的外面。

## 2. Activity

把 SwipeRefreshLayout 和 RecyclerView 拿住。写一些一会儿要实现的桩 stub.

## 3. 定一个上拉加载更多的接口

当上拉列表的时候，我们会调用这个接口方法

## 4. 关键的 Adapter

继承自 RecyclerView.Adapter

## 5. 效果

## 6. 多itemType

现在开始有点意思了，当你在屏幕滑动的时候，如何重复利用之前显示过，现在没显示了的 “同类型的 view ” 来填充即将显示的 “同类型的 view ”

### 6.1 根据 position 得到不同的 viewType

onCreateViewHolder

### 6.2 根据 viewType 设置填充 view 的数据

onBindViewHolder

### 6.3 用枚举来表示吧
