---
title: URP自定义后处理特效第零篇：前期准备
date: 2022-02-22 21:50:00
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

## 1.2 我为什么用URP

主要原因是因为URP提供了2D光照系统，可以很方便快速地布置光照

## 1.3 怎么切换成URP

### 1.3.1 安装URP

本文使用的Unity版本为在2020.3.27f1c1，我们打开Unity的Window/Package Manager，Packages选择Unity Registry，可以找到Universal RP的包，点击Install进行安装

### 1.3.2 新建URPAsset

在Project窗口中，选一个位置右键，点Create/Rendering/Universal Render Pipeline/Pipeline Asset(Forward Renderer)新建一个URPAsset，默认名称是UniversalRenderPipelineAsset，我们可以改名为URPAsset。新建之后还会跟着多出一个叫UniversalRenderPipelineAsset_Renderder的文件，点击URPAsset，可以看到Inspector窗口中的General/RendererList已经自动放好了这个默认的文件。它是一个称之为Forward Renderer的文件，可以在Create/Rendering/中找到。由于我们的后处理特效是通过这个文件来设置的，因此我们可以将这个文件改名为CustomForwardRendererData。

### 1.3.3 切换为默认是支持2D光照的2DRenderer，且保留CustomForwardRendererData

在Project窗口中，选一个位置右键，点Create/Rendering/Universal Render Pipeline/2D Renderer(Experimental)新建一个2D Renderer，默认名称是New 2D Renderer，我们可以改名为2DRenderer。将该2DRenderer放入上面第2步创建的URPAsset文件的Inspector窗口中的General/RendererList的0号位置中，替换掉原来默认的CustomForwardRendererData。这个CustomForwardRendererData还应该加回去来做2DRenderer无法实现的功能，因此我们点击Inspector窗口中的General/Renderer List中的+，新增一行然后将CustomForwardRendererData拖进去

### 1.3.4 切换为URP渲染管线

打开Unity的Edit/Project Settings/Graphics，将URPAsset拖入Scriptable Render Pipeline Settings一栏中，到此即可切换为URP渲染管线了

# 二、兼容2D光照的URP自定义后处理特效

此部分内容主要参考自B站一位大佬的架构设计，非常感谢分享了这样一个方便的架构！https://www.bilibili.com/video/av588100663/

## 2.1 架构设计

![](/img/CustomPostProcessing.png)

架构图如上，其具体代码为：

```csharp
using System.Collections;
using System.Collections.Generic;
using System.Linq;
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class CustomPostProcessRendererFeature : ScriptableRendererFeature {
    // 不同插入点的render pass
    CustomPostProcessRenderPass afterOpaqueAndSky;
    CustomPostProcessRenderPass beforePostProcess;
    CustomPostProcessRenderPass afterPostProcess;

    // 所有自定义的VolumeComponent
    List<CustomVolumeComponent> components;

    // 用于after PostProcess的render target
    RenderTargetHandle afterPostProcessTexture;

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData) {
        // CustomPostProcessRenderPass中定义了一个变量activeComponents来存储当前可用的的后处理组件，
        // 在Render Feature的AddRenderPasses中，需要先判断Render Pass中是否有组件处于激活状态，
        // 如果没有一个组件激活，那么就没必要添加这个Render Pass，
        // 这里调用先前在组件中定义好的Setup方法初始化，随后调用IsActive判断其是否处于激活状态
        if (renderingData.cameraData.postProcessEnabled) {
            // 为每个render pass设置render target
            var source = new RenderTargetHandle(renderer.cameraColorTarget);
            if (afterOpaqueAndSky.SetupComponents()) {
                afterOpaqueAndSky.Setup(source, source);
                renderer.EnqueuePass(afterOpaqueAndSky);
            }
            if (beforePostProcess.SetupComponents()) {
                beforePostProcess.Setup(source, source);
                renderer.EnqueuePass(beforePostProcess);
            }
            if (afterPostProcess.SetupComponents()) {
                // 如果下一个Pass是FinalBlit，则输入与输出均为_AfterPostProcessTexture
                source = renderingData.cameraData.resolveFinalTarget ? afterPostProcessTexture : source;
                afterPostProcess.Setup(source, source);
                renderer.EnqueuePass(afterPostProcess);
            }
        }
    }

    // 初始化Feature资源，每当序列化发生时都会调用
    public override void Create() {
        // 从VolumeManager获取所有自定义的VolumeComponent
        var stack = VolumeManager.instance.stack;
        components = VolumeManager.instance.baseComponentTypeArray
            .Where(t => t.IsSubclassOf(typeof(CustomVolumeComponent)) && stack.GetComponent(t) != null)
            .Select(t => stack.GetComponent(t) as CustomVolumeComponent)
            .ToList();

        // 初始化不同插入点的render pass
        var afterOpaqueAndSkyComponents = components
            .Where(c => c.InjectionPoint == CustomPostProcessInjectionPoint.AfterOpaqueAndSky)
            .OrderBy(c => c.OrderInPass)
            .ToList();
        afterOpaqueAndSky = new CustomPostProcessRenderPass("Custom PostProcess after Opaque and Sky", afterOpaqueAndSkyComponents);
        afterOpaqueAndSky.renderPassEvent = RenderPassEvent.AfterRenderingOpaques;

        var beforePostProcessComponents = components
            .Where(c => c.InjectionPoint == CustomPostProcessInjectionPoint.BeforePostProcess)
            .OrderBy(c => c.OrderInPass)
            .ToList();
        beforePostProcess = new CustomPostProcessRenderPass("Custom PostProcess before PostProcess", beforePostProcessComponents);
        beforePostProcess.renderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing;

        var afterPostProcessComponents = components
            .Where(c => c.InjectionPoint == CustomPostProcessInjectionPoint.AfterPostProcess)
            .OrderBy(c => c.OrderInPass)
            .ToList();
        afterPostProcess = new CustomPostProcessRenderPass("Custom PostProcess after PostProcess", afterPostProcessComponents);
        // 为了确保输入为_AfterPostProcessTexture，这里插入到AfterRendering而不是AfterRenderingPostProcessing
        afterPostProcess.renderPassEvent = RenderPassEvent.AfterRendering;

        // 初始化用于after PostProcess的render target
        afterPostProcessTexture.Init("_AfterPostProcessTexture");
    }

    // 资源释放
    protected override void Dispose(bool disposing) {
        base.Dispose(disposing);
        if (disposing && components != null) {
            foreach (var item in components) {
                item.Dispose();
            }
        }
    }
}
```

```csharp
using System.Collections;
using System.Collections.Generic;
using System.Linq;
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

// 官方示例中，一个Renderer Feature对应一个自定义后处理效果，各个后处理相互独立，好处是灵活自由易调整；
// 坏处也在此，相互独立意味着每个效果都可能要开临时RT，耗费资源比双缓冲互换要多，
// 并且Renderer Feature在Renderer Data下，相对于场景中的Volume来说在代码中调用起来反而没那么方便。
// 那么这里的思路便是将所有相同插入点的后处理组件放到同一个Render Pass下渲染，这样就可以做到双缓冲交换，又保持了Volume的优势。

public class CustomPostProcessRenderPass : ScriptableRenderPass {
    List<CustomVolumeComponent> volumeComponents; // 所有自定义后处理组件
    List<int> activeComponents; // 当前可用的组件下标

    string profilerTag;
    List<ProfilingSampler> profilingSamplers; // 每个组件对应的ProfilingSampler

    RenderTargetHandle source; // 当前源与目标
    RenderTargetHandle destination;
    RenderTargetHandle tempRT0; // 临时RT
    RenderTargetHandle tempRT1;

    /// <param name="profilerTag">Profiler标识</param>
    /// <param name="volumeComponents">属于该RendererPass的后处理组件</param>
    public CustomPostProcessRenderPass(string profilerTag, List<CustomVolumeComponent> volumeComponents) {
        this.profilerTag = profilerTag;
        this.volumeComponents = volumeComponents;
        activeComponents = new List<int>(volumeComponents.Count);
        profilingSamplers = volumeComponents.Select(c => new ProfilingSampler(c.ToString())).ToList();

        tempRT0.Init("_TemporaryRenderTexture0");
        tempRT1.Init("_TemporaryRenderTexture1");
    }

    // 你可以在这里实现渲染逻辑。
    // 使用<c>ScriptableRenderContext</c>来执行绘图命令或Command Buffer
    // https://docs.unity3d.com/ScriptReference/Rendering.ScriptableRenderContext.html
    // 你不需要手动调用ScriptableRenderContext.submit，渲染管线会在特定位置调用它。
    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData) {
        var cmd = CommandBufferPool.Get(profilerTag);
        context.ExecuteCommandBuffer(cmd);
        cmd.Clear();

        // 获取Descriptor
        var descriptor = renderingData.cameraData.cameraTargetDescriptor;
        descriptor.msaaSamples = 1;
        descriptor.depthBufferBits = 0;

        // 初始化临时RT
        RenderTargetIdentifier buff0, buff1;
        bool rt1Used = false;
        cmd.GetTemporaryRT(tempRT0.id, descriptor);
        buff0 = tempRT0.id;
        // 如果destination没有初始化，则需要获取RT，主要是destinaton为_AfterPostProcessTexture的情况
        if (destination != RenderTargetHandle.CameraTarget && !destination.HasInternalRenderTargetId()) {
            cmd.GetTemporaryRT(destination.id, descriptor);
        }

        // 执行每个组件的Render方法
        // 如果只有一个组件，则直接source -> buff0
        if (activeComponents.Count == 1) {
            int index = activeComponents[0];
            using (new ProfilingScope(cmd, profilingSamplers[index])) {
                volumeComponents[index].Render(cmd, ref renderingData, source.Identifier(), buff0);
            }
        } else {
            // 如果有多个组件，则在两个RT上左右横跳
            cmd.GetTemporaryRT(tempRT1.id, descriptor);
            buff1 = tempRT1.id;
            rt1Used = true;
            Blit(cmd, source.Identifier(), buff0);
            for (int i = 0; i < activeComponents.Count; i++) {
                int index = activeComponents[i];
                var component = volumeComponents[index];
                using (new ProfilingScope(cmd, profilingSamplers[index])) {
                    component.Render(cmd, ref renderingData, buff0, buff1);
                }
                CoreUtils.Swap(ref buff0, ref buff1);
            }
        }

        // 最后blit到destination
        Blit(cmd, buff0, destination.Identifier());

        // 释放
        cmd.ReleaseTemporaryRT(tempRT0.id);
        if (rt1Used)
            cmd.ReleaseTemporaryRT(tempRT1.id);

        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }

    /// <summary>
    /// 设置后处理组件
    /// </summary>
    /// <returns>是否存在有效组件</returns>
    public bool SetupComponents() {
        activeComponents.Clear();
        for (int i = 0; i < volumeComponents.Count; i++) {
            volumeComponents[i].Setup();
            if (volumeComponents[i].IsActive()) {
                activeComponents.Add(i);
            }
        }
        return activeComponents.Count != 0;
    }

    /// <summary>
    /// 设置渲染源和渲染目标
    /// </summary>
    public void Setup(RenderTargetHandle source, RenderTargetHandle destination) {
        this.source = source;
        this.destination = destination;
    }
}
```

```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

/// 后处理插入位置
public enum CustomPostProcessInjectionPoint {
    AfterOpaqueAndSky, // 天空渲染之后
    BeforePostProcess, // 内置后处理之前
    AfterPostProcess // 内置后处理之后
}

public abstract class CustomVolumeComponent : VolumeComponent, IPostProcessComponent, IDisposable {
    // 插入位置
    public virtual CustomPostProcessInjectionPoint InjectionPoint => CustomPostProcessInjectionPoint.AfterPostProcess;

    // 在同一个插入点可能会存在多个后处理组件，所以还需要一个排序编号来确定谁先谁后：
    // 在InjectionPoint中的渲染顺序
    public virtual int OrderInPass => 0;

    // 初始化，将在RenderPass加入队列时调用
    public abstract void Setup();

    // 执行渲染
    public abstract void Render(CommandBuffer cmd, ref RenderingData renderingData, RenderTargetIdentifier source, RenderTargetIdentifier destination);

    #region IPostProcessComponent
    public abstract bool IsActive(); // 返回当前组件是否处于激活状态

    public virtual bool IsTileCompatible() => false;
    #endregion

    #region IDisposable 由于渲染可能需要临时生成材质，在这里将它们释放
    public void Dispose() {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    public virtual void Dispose(bool disposing) { }
    #endregion
}
```

代码注释较为全面了，看注释即可

## 2.2 使用方式

我们点击CustomForwardRendererData，在Inspector界面点击Add Renderer Feature，然后新建一个Custom Post Process Renderer Feature，名称随意不影响使用，这里取名为CustomPostProcessRendererFeature

有了这个RendererFeature之后，我们就能随心所欲地添加我们的自定义后处理特效了！只要我们继承实现自己的CustomVolumeComponent类就行。由于篇幅原因，这部分内容放下一章讲。在这之前，这里先说明一下URP中默认提供的一些后处理特效，也是Volume的基本使用吧

在Hierarchy右键，新建一个Volume/Global Volume，我们可以在Inspector界面中看到它需要传入一个Profile，这里我们在Project中右键Create/Volume Profile，名称就叫GlobalVolumeProfile好了。创建结束之后将其拖到Profile中。完成这一步之后，我们就可以给这个Profile添加各种后处理特效了，我们点击Global Volume的Inspector界面的Add Override，新增一个Bloom特效，现在我们试一下勾上这个特效的Threshold盒Intensity，数值分别填入0.5和1，我们就可以直接在Scene界面和Game界面观察到图片变亮了（请自己在场景放一张图片），有了Bloom的特效

好了，我们下一章见
