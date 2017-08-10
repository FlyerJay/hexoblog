---
title: 再读js高程
date: 2017-07-06 10:44:04
tags: 'javascript'
---
### 前言
&#8195;&#8195;《javascript高级程序设计》一书被称为javascript入门圣经、红宝书。第一次通读全书，其实有很多地方没看明白，只是掌握了一些js的基本知识，对于一些稍微高级的用法不明所以，但学到的东西已足够日常所有。现在我也差不多从事前端工作一年有余，回过头再来读一遍这本书，发现又有了一些新收获。
### 数据类型
&#8195;&#8195;js中有6种数据类型，分别是：undefined、boolean、string、number、object、function。可以通过操作符typeof查看变量的数据类型。这里需要强调的是null是object类型，表示一个空对象，虽然 null == undefined 返回true，但是null === undefined 返回false，因为它们是两个不同类型的数据。
&#8195;&#8195;typeof在检测数据类型时确实很有用，但还有些场景是typeof无能为力的。有时我们想要知道一个引用对象的具体类型如：Array，RegExp等，typeof会始终返回object，这并不是我们想要的结果。instanceof适用于检测引用对象的构造类型，[] instanceof Array,/\*/ instanceof RegExp等，需要注意的是所有引用类型instanceof Object都会返回true，基本类型不能使用instanceof来检测，都会返回false。
### 基本类型和引用类型
&#8195;&#8195;基本类型有：string、number、undefined、null、boolean，引用类型有：object、array、function。基本类型在保存数据时是按照实际值保存的，而引用类型是保存的内存中一个对象的地址，通过保存的地址来访问内存中的对象。在赋值