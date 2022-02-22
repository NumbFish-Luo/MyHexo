---
title: 火焰类法术粒子特效设计思路解析
date: 2022-02-22 19:50:00
toc: true
tags:
- Unity
- Particle
categories:
- Unity
banner_img: /img/Particle/Title.png
banner_img_set: /img/Particle/Title.png

---

# 一、效果展示

**前排注意，本文动图较多，请耐心等待加载完毕！！**

![](/img/Particle/FireBall.gif)

![](/img/Particle/MagicMissile.gif)

![](/img/Particle/Explosion.gif)

![](/img/Particle/Torch.gif)

主要参考了《Lost Ruins》的粒子效果设计，下面进行设计思路解析（具体粒子特效属性设置请移步至项目https://github.com/NumbFish-Luo/Project01 ）

# 二、设计思路解析

## 2.1 火球术

![](/img/Particle/FireBall.gif)

![](/img/Particle/Explosion.gif)

![](/img/Particle/FireBallPrefab.png)

火球术的设计，从上到下，分成了：

- 中间拖尾粒子（MiddleParticle1）
- 中间的头部粒子（MiddleParticle2）
- 外围稀疏的粒子（OutsideParticle）
- 黑色烟雾粒子（BlackFogParticle）
- 2D光源中的红色光（Light2D/RedLight）
- 镜头光晕（Light2D/LensFlare）
- 爆炸特效中的炸开花的粒子（ExplosionEffect/Particle1）
- 随风消逝的粒子（ExplosionEffect/Particle2）
- 三段镜头爆炸光晕（ExplosionEffect/Light2D/LensFlare1-LensFlare3）
- 热扭曲特效（HeatDistortion）

![](/img/Particle/FireParticleSprite.png)

火球术粒子的贴图本身是长方形的白色虚边缘图片（上图，请在暗黑模式下观看）。默认情况下粒子不会往一个明确的方向飘，不过它具有一定的飞行速度，并且在设置中，所有粒子的simulation space改成world，shape为cone，方向从后往前（xyz的旋转角度都为0），color over lifetime设置颜色随生命周期逐渐变深变透明，rotation over lifetime和rotation by speed的范围在-360°到360°之间，打开noise进行一定程度上的扰动……通过这些设置后，火球术在移动时就会展示出漂亮的拖尾效果了！

火球术本身带有碰撞盒，当火球术击中物体时，会播放爆炸特效部分，并在特效播放结束后销毁

另外的热扭曲特效部分在这里不展开讲，请移步至 [文章链接未定]

## 2.2 魔法飞弹

![](/img/Particle/MagicMissile.gif)

![](/img/Particle/Explosion.gif)

![](/img/Particle/MagicMissilePrefab.png)

魔法飞弹的设计，参考了火球术，从上到下，分成了：

- 中间拖尾粒子（MiddleParticle1）
- 中间的头部粒子（MiddleParticle2）
- 中间稀疏的粒子（MiddleParticle2）
- 2D光源中的紫色光（Light2D/PurpleLight）
- 镜头光晕（Light2D/LensFlare）
- 爆炸特效中的炸开花的粒子（ExplosionEffect/Particle1）
- 随风消逝的粒子（ExplosionEffect/Particle2）
- 三段镜头爆炸光晕（ExplosionEffect/Light2D/LensFlare1-LensFlare3）

![](/img/Particle/MagicMissileParticleSprite.png)

魔法飞弹有两种贴图，一种是使用了上面火球术的长方形白色虚边缘图片，还有一种是白色的魔法火焰序列帧（上图，请在暗黑模式下观看）。魔法飞弹和火球术一样具有飞行速度和碰撞盒，但是没有热扭曲特效。

## 2.3 火把

![](/img/Particle/Torch.gif)

![](/img/Particle/TorchPrefab.png)

火把的设计，同样参考了火球术，不过火把本身不带有速度（虽然有也没关系），所以其粒子默认情况下是会自动往上飘的。其结构从上到下，分成了：

- 中间往上飘的粒子（MiddleParticle）
- 外围稀疏的往上飘的粒子（OutsideParticle）
- 背景的往上飘的黑色烟雾（BlackFogParticle）
- 2D光源中的两个红色光（Light2D/RedLight1-RedLight2）
- 镜头光晕（Light2D/LensFlare）
- 热扭曲特效（HeatDistortion）

## 2.4 其他

更多粒子特效后续再补充...