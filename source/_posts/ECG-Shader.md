---
title: 心电图Shader
date: 2021-09-30 22:45:00
toc: true
tags:
- Shader
categories:
- Shader
banner_img: /img/ECG.jpg
banner_img_set: /img/ECG.jpg
---

# 效果

这个Shader是比赛的废案，至于为什么是废案？详见文章：[2021 益游未尽 作品——《气候护卫队》](https://numbfish-0g5ok9eq2a9e80fc-1306659420.ap-shanghai.app.tcloudbase.com/2021/09/30/2021YiYouWeiJin/)

效果如下，这里演示的是正常心跳的波形循环
![](/img/2021YiYouWeiJin/ECG.gif)

# Shader代码

frag中的主要工作是，把所有的效果叠加（波形线条、拖尾特效、闪烁特效、格子）

```GLSL
Shader "Custom/ECG" {
    Properties {
        [Header(Color Setting)]
        _backgroundColor("Background Color", Color) = (0, 0, 0, 1)
        _lineColor("Line Color", Color) = (0, 1, 0, 1)

        [Header(Line Setting)]
        [PowerSlider(3)]
        _lineWidth("Line Width", Range(0, 0.5)) = 0.02
        _lightTailLength("Light Tail Length", Range(0, 2)) = 1
        
        [Header(Grid Setting _ 1 small grid.xy equal 0.04s 0.1mV)]
        _bigGridCount("Big Grid Count", vector) = (5, 5, 0, 0)
        _bigGridWidth("Big Grid Width", Range(0, 1)) = 0.15
        _bigGridAlpha("Big Grid Alpha", Range(0, 1)) = 1
        _smallGridWidth("Small Grid Width", Range(0, 1)) = 0.2
        _smallGridAlpha("Small Grid Alpha", Range(0, 1)) = 0.75

        [Header(Other Setting)]
        _lightPoundAlpha("Light Pound Alpha", Range(0, 1)) = 0.125
    }

    SubShader{
        Pass {
            CGPROGRAM

            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            #define PI 3.14159

            // 一大格所代表的量, x为时间, y为电压
            #define GRID_X_SECOND (0.04 * 5)
            #define GRID_Y_MV (0.1 * 5)

            #define _ECG_Len 100
            fixed _ECG_Arr[_ECG_Len]; // 心电图波形数据，由C#代码传入，长度固定100

            struct appdata {
                float4 vertex : POSITION;
                fixed4 uv : TEXCOORD0;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                fixed4 uv : TEXCOORD0;
            };

            // Color Setting
            fixed4 _backgroundColor;
            fixed4 _lineColor;
            // Line Setting
            fixed _lineWidth;
            fixed _lightTailLength;
            // Grid Setting
            half4 _bigGridCount;
            fixed _bigGridWidth;
            fixed _bigGridAlpha;
            fixed _smallGridWidth;
            fixed _smallGridAlpha;
            // Other Setting
            fixed _lightPoundAlpha;

            // 心跳波形
            fixed HeartbeatFunc(fixed2 uv, fixed lightWidth) {
                half x = uv.x * (_ECG_Len - 1); // [0, 1] => [0, len - 1]
                int xl = floor(x);
                int xr = (xl + 1) % _ECG_Len;
                fixed y = lerp(_ECG_Arr[xl], _ECG_Arr[xr], frac(x)) / 2 + 0.5; // y值+0.5以将曲线移动到屏幕中间
                return 1.0 - smoothstep(0.0, lightWidth, abs(uv.y - y));
            }

            // 光拖尾(按时间透明化尾部)效果
            half LightFunc(fixed2 uv, half lightInterval, fixed lightTail) {
                half tt = _Time[1] % lightInterval;
                half lightSpeed = 1 / lightInterval;
                half highlightX = tt * lightSpeed; // the leftest highlightX
                half delta; // pixel distance from the first hightlightX on the right, delta~[0,1]
                if (uv.x <= highlightX) {
                    delta = highlightX - uv.x;
                } else {
                    delta = 1 - frac(uv.x - highlightX);
                }
                half light = lightTail > delta ? (lightTail - delta) : 0.0;
                return light;
            }

            // take the screen coordinates, return the 0~1 alpha according to the pound function
            // 屏幕闪烁效果
            fixed PoundFunc(fixed2 uv, half lightInterval) {
                half tt = (_Time[1] - uv.x * lightInterval / 2.0) % lightInterval;
                half percent = tt / 0.8f; // pound lasts 0.8 seconds
                return percent > 0.1 ? (-1.0 / 0.9 * (percent - 1.0)) : 0;
            }

            fixed TriangularWave(half x) { // => [-1, 1]
                return abs(frac(x) * 2 - 1) - 0.5;
            }

            // take the screen coordinates, return the 0~1 alpha according to the grid function
            fixed GridFunc(fixed2 uv, half2 gridXY) {
                fixed scanlineX = TriangularWave(uv.x * gridXY.x) - 0.5;
                fixed scanlineY = TriangularWave(uv.y * gridXY.y) - 0.5;
                return max(scanlineX, scanlineY);
            }

            v2f vert(appdata v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.uv = v.uv;
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed heartbeat = HeartbeatFunc(i.uv, _lineWidth);
                half totalInterval = GRID_X_SECOND * _bigGridCount.x;
                half light = LightFunc(i.uv, totalInterval, _lightTailLength);
                fixed pound = PoundFunc(i.uv, totalInterval) * _lightPoundAlpha;
                fixed bigGrid = (GridFunc(i.uv, _bigGridCount) + _bigGridWidth) * _bigGridAlpha;
                fixed smallGrid = (GridFunc(i.uv, _bigGridCount * 5) + _smallGridWidth) * _smallGridAlpha;
                fixed grid = max(bigGrid, smallGrid);

                fixed alpha = heartbeat * light + pound;
                alpha = max(alpha, grid);
                fixed4 fragColor = lerp(_backgroundColor, _lineColor, alpha);
                return fragColor;
            }
            ENDCG
        }
    }
}
```
# C#代码：传递波形数据给Shader

C#代码的主要工作是传入波形数据数组给Shader

```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class ECG : MonoBehaviour {
    public Material material;
    public Renderer renderer;

    private Vector2 bigGridCount;
    private float totalInterval;

    private float heartRate = 0;
    private int preX = 0;

    private const int WAVE_ARR_LEN = 100;
    private float[] wave = new float[WAVE_ARR_LEN];

    // 一大格所代表的量, x为时间(s), y为电压(mV)
    private const float GRID_X_SEC = 0.04f * 5;
    private const float GRID_Y_MV = 0.1f * 5;

    // 正常波形, 参考周期 = 0.8s, 参考峰值 = 1mV
    private const float REGULAR_WAVE_REFER_T = 0.8f; // time s
    private const float REGULAR_WAVE_REFER_F = 1 / REGULAR_WAVE_REFER_T * 60; // frequency bpm
    private const float REGULAR_WAVE_REFER_P = 1; // peak val mV
    private readonly float[] REGULAR_WAVE_ARR = { 0.025f, 0.025f, 0.025f, 0.025f, 0.022f, 0.031f, 0.058f, 0.099f, 0.133f, 0.161f, 0.181f, 0.19f, 0.191f, 0.182f, 0.163f, 0.135f, 0.102f, 0.06f, 0.032f, 0.022f, 0.025f, 0.025f, 0.029f, 0.018f, -0.023f, -0.068f, -0.037f, 0.059f, 0.255f, 0.578f, 0.983f, 1.0f, 0.498f, -0.097f, -0.273f, -0.13f, 0.002f, 0.037f, 0.025f, 0.025f, 0.025f, 0.025f, 0.025f, 0.025f, 0.025f, 0.025f, 0.025f, 0.025f, 0.025f, 0.025f, 0.025f, 0.025f, 0.025f, 0.024f, 0.032f, 0.052f, 0.078f, 0.1f, 0.12f, 0.14f, 0.159f, 0.179f, 0.196f, 0.212f, 0.225f, 0.237f, 0.246f, 0.25f, 0.248f, 0.24f, 0.225f, 0.206f, 0.184f, 0.161f, 0.134f, 0.101f, 0.06f, 0.031f, 0.022f, 0.025f, 0.025f, 0.023f, 0.03f, 0.052f, 0.082f, 0.101f, 0.111f, 0.11f, 0.098f, 0.078f, 0.06f, 0.043f, 0.031f, 0.025f, 0.025f, 0.025f, 0.025f, 0.025f, 0.025f, 0.025f };

    // 插值计算波形值
    float GetWaveY(float[] waveArr, float x, float peakVal) {
        if (x < 0) {
            x += ((-x / WAVE_ARR_LEN) + 1) * WAVE_ARR_LEN;
        }
        x %= WAVE_ARR_LEN;
        int xl = Mathf.FloorToInt(x) % WAVE_ARR_LEN;
        int xr = (xl + 1) % WAVE_ARR_LEN;
        return Mathf.Lerp(waveArr[xl], waveArr[xr], x - xl) * peakVal;
    }

    void Start() {
        // 获取必要属性
        bigGridCount = material.GetVector("_bigGridCount");
        totalInterval = GRID_X_SEC * bigGridCount.x; // 总时长 = 大格子时间 * 大格子数量.x
        material.SetFloatArray("_ECG_Arr", wave);
        renderer.material = material;
    }

    delegate void FillLambda(int end);

    // 展示单独一种波的循环
    void ShowSingleWave(float[] waveArr, float interval, float peakVal) {
        // 根据shader时间获取当前扫描进度，相当于获取了x坐标
        float shaderTime1 = Shader.GetGlobalVector("_Time")[1];
        float scanProcess = shaderTime1 % totalInterval / totalInterval; // 值域[0, 1]
        int x = (int)(scanProcess * WAVE_ARR_LEN);

        // 根据波形周期计算此时波形进度
        float waveProcess = (shaderTime1 % interval) / interval;

        // 更新此时x位置波形数据
        wave[x] = GetWaveY(waveArr, waveProcess * WAVE_ARR_LEN, peakVal);

        // 补上上一帧和这一帧中间未打的点
        FillLambda Fill = (int end) => {
            for (; preX < end; ++preX) {
                float preShaderTime1 = shaderTime1 - (totalInterval / WAVE_ARR_LEN) * (end - preX);
                float preWaveProcess = (preShaderTime1 % interval) / interval;
                wave[preX] = GetWaveY(waveArr, preWaveProcess * WAVE_ARR_LEN, peakVal);
            }
        };
        Fill(x);
        if (preX > x) {
            Fill(WAVE_ARR_LEN);
            preX = 0;
            Fill(x);
        }

        // 赋值给shader
        material.SetFloatArray("_ECG_Arr", wave);
        Debug.Log("scanProcess: " + (scanProcess * 100) + "%");
    }

    void Update() {
        ShowSingleWave(REGULAR_WAVE_ARR, REGULAR_WAVE_REFER_T, REGULAR_WAVE_REFER_P);
    }
}
```

# Python代码：用于生成特定长度的波形数组

代码中的REGULAR_WAVE_ARR数组，来源于一个Python脚本

```Python
import matplotlib.pyplot as plt
import numpy as np
from scipy.signal import savgol_filter
import pyperclip

# 原始数据，用图片画格子数出来的
# 正常波形, 参考心率 = 75bpm, 参考峰值 = 1mV
# y = [0.2, 0.2,                                         # |
#      0.2, 1.0, 1.5, 1.5, 1.0, 0.2,                     # P
#      0.2, 0.2,                                         # |
#     -0.8,                                              # Q
#      2.0, 10.0,                                        # R
#     -3.0, 0.2,                                         # S
#      0.2, 0.2, 0.2, 0.2, 0.2, 0.2,                     # |
#      0.2, 0.7, 1.1, 1.5, 1.8, 2.0, 1.9, 1.5, 1.0, 0.2, # T
#      0.2,                                              # |
#      0.2, 0.8, 0.9, 0.5, 0.2,                          # U
#      0.2, 0.2]                                         # |

# 房颤 = (正常 - P + f杂波) & 心律不齐, 参考心率 = 75bpm, 参考峰值 = 1mV
# f杂波频率350~600bpm, 此参考波形数据未添加杂乱和心律不齐两种特征, 请自行使用sin和rand等函数实现
y = [0.2, 0.2,                                         # |
     0.2, 0.2, 0.2, 0.2, 0.2, 0.2,                     # | (P)
     0.2, 0.2,                                         # |
    -0.8,                                              # Q
     2.0, 10.0,                                        # R
    -3.0, 0.2,                                         # S
     0.2, 0.2, 0.2, 0.2, 0.2, 0.2,                     # |
     0.2, 0.7, 1.1, 1.5, 1.8, 2.0, 1.9, 1.5, 1.0, 0.2, # T
     0.2,                                              # |
     0.2, 0.8, 0.9, 0.5, 0.2,                          # U
     0.2, 0.2]                                         # |

# 室颤 = 杂波

y = np.array(y)
x = np.arange(0, len(y), 1)

# 线性插值扩展数组长度
lenEx = 100
xEx = np.arange(0, len(y), len(y) / lenEx)
y = np.interp(xEx, x, y)

# 平滑曲线。参数1为输入曲线；参数2为卷积窗口值，越大越平滑，越小越接近原始曲线；参数3为方程次数，这里用三次方程拟合
y_smooth = savgol_filter(y, 5, 3, mode = 'nearest')
y_normalize = y_smooth / max(y_smooth) # 归一化

# 调整小数精度为3, 并以C#代码格式打印目标数据
csOutput = "{ " + "f, ".join(str(round(i, 3)) for i in y_normalize) + "f };"
pyperclip.copy(csOutput)
print("### 输出数据已复制至剪贴板！ ###")
print(csOutput)
print("### 输出数据已复制至剪贴板！ ###")

# 绘制目标曲线
plt.ylim(min(y), max(y))
plt.xlim(min(xEx), max(xEx))
plt.plot(xEx, y, '1-', label = 'raw')
plt.plot(xEx, y_smooth, 'r-', label = 'smooth')
plt.plot(xEx, y_normalize, 'b-', label = 'normalize')
plt.legend()
plt.show()
```

波形的原始数据是直接用截图，数格子数出来的，上面的注释P、Q、R、S、T、U分别对应着心电图的特征线段，这里需要学习专业知识，当然也可以简单看看下图的描述（此波形为正常波形）

![](/img/ECG_regular.jpg)

# 使用方法

1. 创建一个3D Object: Plane，面向屏幕，铺满屏幕

2. 创建一个Material，选择Shader: Custom/ECG

3. Plane替换上述材质

4. Plane挂载ECG.cs脚本，脚本上挂载上述材质，挂载自己的Renderer

5. 可修改C#脚本内的心电图数组数据来输出不同的波形。数据可由上述Python代码来插值生成平滑曲线

# 总结

这些代码虽然是废案，但是其实也可以使用。如果要再优化的话，或许应该考虑一下数据来源部分——如何才能更好地模拟游戏中各种情况下病人的心电数据？这里还有很多需要学习的地方，专业知识还需要再多学习，最好至少能摸一次心电监护仪，有专业人士讲解等等...

总之这些代码暂时先这样吧，等之后有需要了再调整。