---
title: URP自定义后处理特效第一篇：次元斩！
date: 2022-02-22 21:50:00
toc: true
tags:
- Unity
- URP
- PostEffect
categories:
- Unity
banner_img: /img/DimensionalCut/DimensionalCut.png
banner_img_set: /img/DimensionalCut/DimensionalCut.png
---

# 一、效果展示

![](/img/DimensionalCut/DimensionalCut.gif)

# 二、起因

前段时间刷B站看到的一个DNF刃影的补丁，空间斩，链接：https://www.bilibili.com/video/BV1io4y1D7R6

虽然简单但很有视觉冲击力，因此打算自己也实现一个看看，我的这个就叫次元斩好了！

# 三、思路

## 3.1 确定参数

次元斩切开屏幕是一条直线，显然我们可以用点斜式来描述。于是首先有点的坐标(x_p, y_p)，线的斜率k，为了方便，这里k改用角度制的θ描述，并且限制θ的取值范围为0°到360°，于是我们可以得到直线公式为：
$$
f(x)=kx+b\\k=tan(θ*\pi/180)\\b=y_p-kx_p
$$
我们再引入一个变量o，来表示次元斩斩开空间的程度，用偏移量offset表示，数值越大，则斩开的两侧贴图越往中间靠拢，因此单位以uv的x轴为参考，只影响x方向而不影响y方向，为0时表示无斩开效果。斩开的过程，o将从一个正数值线性地减少到0为止，即逐渐恢复为未斩开的状态

在使用次元斩后的某一时刻，shader的片元着色器函数frag中，对于每个位于(x, y)的像素点都有一个进行偏移后的取值点(x_o, y)，其自变量应该为o，下面分情况讨论：

如果θ在1，3象限，且当前像素点坐标位于次元斩切割直线的上方，我们让上方图片切割后偏移到右侧，即：
$$
x_o(o)=x-o,0°<θ<90°|180°<θ<270°,f(x)>f(x_p)
$$
如果θ在1，3象限，且当前像素点坐标位于次元斩切割直线的下方，我们让下方图片切割后偏移到左侧，即：
$$
x_o(o)=x+o,0°<θ<90°|180°<θ<270°,f(x)\leq f(x_p)
$$

如果θ在2，4象限，且当前像素点坐标位于次元斩切割直线的上方，我们让上方图片切割后偏移到左侧，即：

$$
x_o(o)=x+o,90°<θ<180°|270°<θ<360°,f(x)>f(x_p)
$$
如果θ在2，4象限，且当前像素点坐标位于次元斩切割直线的下方，我们让下方图片切割后偏移到右侧，即：
$$
x_0(o)=x-o,90°<θ<180°|270°<θ<360°,f(x)\leq f(x_p)
$$

如果θ为0°或180°，且当前像素点坐标位于次元斩切割直线的左侧，我们让左侧图片切割后偏移到右侧，即：

$$
x_o(o)=x-o,θ=0°|θ=180°,x<x_p
$$
如果θ为0°或180°，且当前像素点坐标位于次元斩切割直线的右侧，我们让右侧图片切割后偏移到左侧，即：
$$
x_o(o)=x+o,θ=0°|θ=180°,x>x_p
$$
如果θ为90°或270° ，且当前像素点坐标位于次元斩切割直线的上方，我们让上方图片切割后偏移到右侧，即：
$$
x_o(o)=x-o,θ=90°|θ=270°,y>y_p
$$
如果θ为90°或270° ，且当前像素点坐标位于次元斩切割直线的下方，我们让下方图片切割后偏移到左侧，即：
$$
x_o(o)=x+o,θ=90°|θ=270°,x>x_p
$$
综上，我们可以由此写出shader程序：

```hlsl
Shader "Custom/DimensionalCut" {
    Properties {
        _MainTex("Main Texture", 2D) = "" {}
        _X("X", float) = 0 // 原点坐标x
        _Y("Y", float) = 0 // 原点坐标y
        _Angle("Angle", float) = 0 // 角度，由此推算斜率
        _Offset("Offset", float) = 0
    }

    SubShader {
        Tags {"Queue" = "Geometry" "RenderType" = "Opaque"}

        Pass {
            Cull Off
            ZTest LEqual
            ZWrite On
            AlphaTest Off
            Lighting Off
            ColorMask RGBA
            Blend Off

            CGPROGRAM
            #pragma target 2.0
            #pragma fragment frag
            #pragma vertex vert
            #include "UnityCG.cginc"

            sampler2D _MainTex;
            float _X;
            float _Y;
            float _Angle;
            float _Offset;

            struct AppData {
                float4 vertex : POSITION;
                half2 texcoord : TEXCOORD0;
            };

            struct VertexToFragment {
                float4 pos : POSITION;
                half2 uv : TEXCOORD0;
            };

            VertexToFragment vert(AppData v) {
                VertexToFragment o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.uv = v.texcoord.xy;
                return o;
            }
            
            float F(float x) {
                float k = tan(_Angle * 0.01745); // 1度 = π / 180 ≈ 0.01745弧度
                float b = _Y - k * _X;
                return k * x + b;
            }

            float WhenNeq(float x, float y) {
                return abs(sign(x - y));
            }

            fixed4 frag(VertexToFragment i) : COLOR {
                /*
                    x = c > d ? a : b;
                    =>
                    x = lerp(a, b, step(c, d));
                *//*
                    y = g == h ? e : f;
                    =>
                    y = lerp(e, f, WhenNeq(g, h));
                */
                float2 offset = i.uv;
                float a = offset.x + _Offset;
                float b = offset.x - _Offset;
                float c = i.uv.x;
                float d1 = _X;
                float d2 = F(i.uv.y);
                float e = lerp(a, b, step(c, d1));
                float f = lerp(a, b, step(c, d2));
                float g = _Angle % 180;
                float h = 90;
                offset.x = lerp(e, f, WhenNeq(g, h));
                return tex2D(_MainTex, frac(offset));
            }
            ENDCG
        }
    }
}
```

其中由于优化性能的原因，去掉了if判断，改为了性能更优的lerp，step，abs，sign的写法

# 四、控制与视觉反馈优化

以上完成了调整offset参数来完成不同程度屏幕斩开的效果，我们需要再写一个自动调整offset变化的脚本。除此之外，为了更好的视觉反馈，我还添加了切开时有Bloom的效果。

关键代码如下：

```csharp
private void Update() {
    timer -= Time.deltaTime;
    if (timer < 0) {
        timer = 0;
    }
    float percent = (timer / duration);
    offset = maxOffset * percent;
    bloom.threshold.SetValue(new MinFloatParameter(minBloomThrehold.value + (maxBloomThrehold.value - minBloomThrehold.value) * (1 - percent), 0));
}
```

其中的timer变量随时间减小到0，如果使用了次元斩，则timer会被设置为一个定值，表示从斩开到完全恢复所需要的时长。maxOffset表示最大能斩开的程度，实际偏移量offset由timer控制，计算完毕后直接传参数给shader的_Offset属性：`material.SetFloat("_Offset", offset.value)`。bloom由指定最小值（最亮）逐渐线性地变到指定最大值（最暗）。综合起来便得到了最终效果
