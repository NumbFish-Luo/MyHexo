---
title: Unity局域网笔记
date: 2021-10-02 00:30:00
toc: true
tags:
- Unity
- CSharp
- LAN
- Network
categories:
- Unity
- CSharp
- LAN
- Network
banner_img: /img/LAN.jpg
banner_img_set: /img/LAN.jpg
---
> 参考文章：我们来用Unity做一个局域网游戏--ProcessCA
> https://zhuanlan.zhihu.com/p/39005219
> https://zhuanlan.zhihu.com/p/40423071
>
> 注：文中的Message我这里翻译为“报文”
> Message：报文
> Protocol：协议
> Serialize：序列化
> Deserialize：反序列化

基础设置：
| Name       | Value                                                        |
| ---------- | ------------------------------------------------------------ |
| 同步方式   | 状态同步                                                     |
| 网络协议   | TCP/IP                                                       |
| 客户端     | 封装输入信息报文给服务器                                     |
| 服务器     | 接收客户端信息，处理游戏逻辑后把结果以报文形式反馈给指定客户端 |
| 序列化工具 | LitJSON（报文根据协议封装&解析，称之为序列化&反序列化）      |
| 报文协议   | 由于我的目标是做卡牌游戏，所以选用房间制                     |

报文协议设计：

```csharp
// 报文类型
public enum MsgType {
    None, // 空
    HeartBeat, // 心跳
    
    PlayerLogIn, // 玩家登录
    PlayerLogOut, // 玩家注销
    
    RoomCreate, // 房间创建
    RoomEnter, // 房间进入
    RoomExit, // 房间退出
    
    StartGame, // 开始游戏
    Play, // 玩家操作
}
```

