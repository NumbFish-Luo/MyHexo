---
title: URP自定义后处理特效第零篇：前期准备
date: 2022-02-22 19:50:00
toc: true
tags:
- Unity
- URP
- PostEffect
categories:
- Unity
banner_img: /img/PinkCloud.jpg
banner_img_set: /img/PinkCloud.jpg
---

# 一、关于URP（Universal Render Pipeline）

## 1.1 什么是URP

关于URP，在Package Mananger中，官方是这样描述的：

> **The Universal Render Pipeline (URP)** is a prebuilt **Scriptable Render Pipeline**, made by Unity. URP provides artist-friendly workflows that let you quickly and easily create optimized graphics across a range of platforms, from mobile to high-end consoles and PCs.
>
> **通用渲染管线（Universal Render Pipeline，URP）**是一个由Unity官方开发的预构建的**可编程渲染管线（Scriptable Render Pipeline，SRP）**。URP提供了能让你快速轻松地在一系列平台，从手机到高端游戏机和电脑上创建优化图像的艺术家友好型的工作流。

说到URP，就得说SRP是什么，说到SRP就得说RP是什么...这里暂时不表，后续再补充说明，详见https://zhuanlan.zhihu.com/p/103457229

## 1.2 为什么用URP

主要原因是因为URP提供了2D光照系统，可以很方便快速地布置光照

## 1.3 怎么切换成URP

### 1.3.1 安装URP

本文使用的Unity版本为在2020.3.27f1c1，我们打开Unity的Window/Package Manager，Packages选择Unity Registry，可以找到Universal RP的包，点击Install进行安装

### 1.3.2 新建URPAsset

在Project窗口中，选一个位置右键，点Create/Rendering/Universal Render Pipeline/Pipeline Asset(Forward Renderer)新建一个URPAsset，默认名称是UniversalRenderPipelineAsset，我们可以改名为URPAsset。新建之后还会跟着多出一个叫UniversalRenderPipelineAsset_Renderder的文件，我们可以改名为3DRenderer。点击URPAsset，可以看到Inspector窗口中的General/RendererList已经自动放好了默认的3DRenderer文件

### 1.3.3 切换为默认是2DRenderer，且保留3DRenderer

在Project窗口中，选一个位置右键，点Create/Rendering/Universal Render Pipeline/2D Renderer(Experimental)新建一个2D Renderer，默认名称是New 2D Renderer，我们可以改名为2DRenderer。将该2DRenderer放入上面第2步创建的URPAsset文件的Inspector窗口中的General/RendererList的0号位置中，替换掉原来默认的3DRenerer。这个3DRenderer还应该加回去来做2DRenderer无法实现的功能，因此我们点击Inspector窗口中的General/Renderer List中的+，新增一行然后将3DRenderer拖进去

### 1.3.4 切换为URP渲染管线

打开Unity的Edit/Project Settings/Graphics，将URPAsset拖入Scriptable Render Pipeline Settings一栏中，即可切换为URP渲染管线了

# 二、URP自定义后处理特效

## 2.1 架构设计

todo...

## 2.2 使用方式

todo...
