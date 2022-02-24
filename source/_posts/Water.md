---
title: URP自定义后处理特效第二篇：水波扭曲
date: 2022-02-24 14:30:00
toc: true
tags:
- Unity
- URP
- PostEffect
categories:
- Unity
banner_img: /img/Water/Water.png
banner_img_set: /img/Water/Water.png
---

# 一、效果展示

![](/img/Water/Water.gif)

# 二、思路

## 2.1 确定参数

### 2.1.1 扭曲图与相关参数

![](/img/Water/Noise.png)

使用的扭曲噪声图是这张，用其R和G通道来做主贴图扭曲的参数，计算公式定为：```float2 mainTexOffset = -1 * (mapColor * _Power - (_Power * 0.5))```，其中```mapColor```是取自于扭曲噪声图的颜色，取值位置随时间改变，改变的规则为：```float2 mapOffset = float2(frac(_Time.y * _DistortionSpeed), 0)```，```_DistortionSpeed```为扭曲图x轴移动速度，```_Power```为扭曲强度

### 2.1.2 遮罩

如果不添加遮罩，那么扭曲是对整张图进行扭曲的，于是我们需要有一张遮罩图，黑色为无扭曲效果，白色为有扭曲效果，而灰色则表示在这两者之间。为了达到水面分界线清晰的效果，我们需要对其进行一个step操作，颜色亮度低于0.5的视为黑色，否则视为白色，于是有此公式：```float mask = step(0.5, tex2D(_Mask, i.uv))```，```mask```的结果只有0和1两种

### 2.1.3 水的颜色

对于```mask```为0的地方，不进行改色，而对于```mask```为1的地方则需要添加水的颜色，于是颜色的计算可以这样写：```float4 finalColor = (mainTexColor * !mask + mainTexColor + _Color * mask) * 0.5```，其中```mainTexColor```为原本贴图的颜色，```_Color```为指定的水的颜色

### 2.1.4 水面正弦波

如果只完成了上述操作，那么只能创造出一个矩形的水区域。为了添加水面波动，我们需要加上一个正弦波。公式我们定义为：```float2 waveUv = i.uv + float2(0, _Peak * (sin(_WaveFrequency * i.uv.x + waveOffset) - 1))```，其中```_Peak```为正弦波峰值，```_WaveFrequency```为正弦波频率，```waveOffset```为正弦波偏移量，其值随时间变化，公式为：```float waveOffset = _Time.y * _WaveSpeed```，其中```_WaveSpeed```为正弦波移动速度

### 2.1.5 Shader代码

综上，我们的最终代码是这样的：

```hlsl
Shader "Custom/Water" {
    Properties {
        _MainTex("Main Texture", 2D) = "" {}
        _Map("Distortion Map", 2D) = "" {}
        _Mask("Mask", 2D) = "Black" {}
        _Power("Distortion Power", float) = 0
        _DistortionSpeed("Distortion Speed", float) = 0.25
        _WaveSpeed("wave Speed", float) = 1.5
        _Color("Color", color) = (0, 0, 0, 0)
        _Peak("Peak", float) = 0.01
        _WaveFrequency("Wave Frequency", float) = 15
        _Enable("Enable", float) = 0
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
            sampler2D _Map;
            sampler2D _Mask;
            float _Scale;
            float _Power;
            float _DistortionSpeed;
            float _WaveSpeed;
            float4 _Color;
            float _Peak;
            float _WaveFrequency;
            float _Enable;

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

            float WhenNeq(float x, float y) {
                return abs(sign(x - y));
            }

            fixed4 frag(VertexToFragment i) : COLOR {
                // 噪声图偏移量，根据时间偏移，只偏移x
                float2 mapOffset = float2(frac(_Time.y * _DistortionSpeed), 0);
                // 采样处于该uv下的噪声图颜色
                float4 mapColor = tex2D(_Map, frac(i.uv + mapOffset));
                // 主贴图偏移量
                float2 mainTexOffset = _Enable * (-1 * (mapColor * _Power - (_Power * 0.5)));
                // 波浪uv，根据时间偏移，只偏移y
                float waveOffset = _Time.y * _WaveSpeed;
                float2 waveUv = i.uv + float2(0, _Peak * (sin(_WaveFrequency * i.uv.x + waveOffset) - 1));
                // 波浪遮罩
                float waveMask = step(0.5, tex2D(_Mask, waveUv));
                // 遮罩
                float mask = step(0.5, tex2D(_Mask, i.uv));
                mask = max(waveMask, mask);
                // 主贴图偏移
                mainTexOffset = i.uv + mainTexOffset * mask;
                // 主贴图颜色
                float4 mainTexColor = tex2D(_MainTex, mainTexOffset);
                // 最终颜色为两颜色相加后除以2所得
                // 对于遮罩范围外的颜色，为(mainTexColor * (1 + 1 + 0) + _Color * 0 * 0) * 0.5 = mainTexColor;
                // 对于遮罩范围内的颜色，为(mainTexColor * (1 + 0 + 0) + _Color * 1 * 1) = (mainTexColor + _Color) * 0.5
                return (mainTexColor * (1 + !mask + !_Enable) + _Color * mask * _Enable) * 0.5;
            }
            ENDCG
        }
    }
}
```

## 2.2 遮罩计算与优化

另外我们还需要有水波扭曲范围的计算，代码为：

```csharp
public class Water : MonoBehaviour {
    [SerializeField] private BoxCollider2D boxCollider2D;
    public Vector2 LeftBottom { get => (Vector2)transform.position + boxCollider2D.offset - 0.5f * boxCollider2D.size; }
    public Vector2 RightTop { get => (Vector2)transform.position + boxCollider2D.offset + 0.5f * boxCollider2D.size; }

    private void Start() {
        WaterManager.Instance.AddItem(GetHashCode(), this);
    }
}
```

其返回碰撞盒的左下角与右上角坐标，以得到碰撞盒的范围

最后我们有一个统一管理所有Water组件的WaterManager，在这个管理器中将统一计算出所有Water组件碰撞范围，并绘制遮罩图。注意，为了性能优化，我们需要缩小遮罩图大小，我这里给的缩小值为0.1，即缩小了整整10倍，但对其最终效果的影响几乎没有！下面是管理器的具体代码：

```csharp
public class WaterManager : MonoSingleton<WaterManager> {
    private Dictionary<int, Water> items = new Dictionary<int, Water>(); // key = hashCode
    private Material material = null;
    private Texture2D mask = null;
    [SerializeField] private float scale = 0.05f;
    private bool enable = false;
    public bool BoolEnable { get => enable; }
    public float FloatEnable { get => enable ? 1 : 0; }

    public void AddItem(int hashCode, Water item) {
        items.Add(hashCode, item);
    }

    public void RemoveItem(int hashCode) {
        items.Remove(hashCode);
    }

    private void FillBlackToMask() {
        for (int y = 0; y < mask.height; ++y) {
            for (int x = 0; x < mask.width; ++x) {
                mask.SetPixel(x, y, Color.black);
            }
        }
        mask.Apply();
    }

    public void SetMaterial(Material material) {
        this.material = material;
        mask = new Texture2D((int)(Screen.width * scale), (int)(Screen.height * scale));
        FillBlackToMask();
    }

    private void FillWhiteToMask(Vector2 leftBottom, Vector2 rightTop) {
        leftBottom = Camera.main.WorldToScreenPoint(leftBottom) * scale;
        rightTop = Camera.main.WorldToScreenPoint(rightTop) * scale;
        int minX = (int)leftBottom.x;
        minX = (minX < 0) ? 0 : minX;
        int maxX = (int)rightTop.x;
        maxX = (maxX > mask.width) ? mask.width : maxX;
        int minY = (int)leftBottom.y;
        minY = (minY < 0) ? 0 : minY;
        int maxY = (int)rightTop.y;
        maxY = (maxY > mask.height) ? mask.height : maxY;
        for (int y = minY; y < maxY; ++y) {
            for (int x = minX; x < maxX; ++x) {
                mask.SetPixel(x, y, Color.white);
            }
        }
    }

    private void Update() {
        if (!material) {
            return;
        }
        FillBlackToMask();
        enable = false;
        List<int> nullKey = new List<int>();
        foreach (KeyValuePair<int, Water> i in items) {
            if (i.Value == null) {
                nullKey.Add(i.Key);
            } else {
                enable = true;
                FillWhiteToMask(i.Value.LeftBottom, i.Value.RightTop);
            }
        }
        foreach (int n in nullKey) {
            items.Remove(n);
        }
        if (enable) {
            mask.Apply();
            material.SetTexture("_Mask", mask);
        }
    }
}
```