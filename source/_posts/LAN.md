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
> 注：文中的Message我这里翻译为“报文”，要问为什么...因为以前第一份工作的Message protocol翻译为报文协议。
>
> Message：报文
> Protocol：协议
> Serialize：序列化
> Deserialize：反序列化

# 一、基础设计

| Name       | Value                                                        |
| ---------- | ------------------------------------------------------------ |
| 同步方式   | 状态同步                                                     |
| 网络协议   | TCP/IP                                                       |
| 客户端     | 封装输入信息报文给服务器                                     |
| 服务器     | 接收客户端信息，处理游戏逻辑后把结果以报文形式反馈给指定客户端 |
| 序列化工具 | LitJSON（报文根据协议封装&解析，称之为序列化&反序列化）      |
| 报文协议   | 由于我的目标是做卡牌游戏，所以选用房间制                     |

# 二、报文协议设计

## 2.1 报文类型

| Name         | Description                                            |
| ------------ | ------------------------------------------------------ |
| HeartBeat    | 心跳，每隔一段时间发送一次，以确保网络是畅通的         |
| PlayerLogIn  | 玩家登录                                               |
| PlayerLogOut | 玩家注销                                               |
| RoomCreate   | 房间创建                                               |
| RoomEnter    | 房间进入                                               |
| RoomExit     | 房间退出                                               |
| StartGame    | 开始游戏                                               |
| Play         | 玩家操作，最复杂的报文，与游戏进行时的数据交互息息相关 |

示例代码：

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
    Play // 玩家操作
}
```

## 2.2 心跳报文 HeartBeat

### 2.2.1 上传

客户端每隔n秒上传1次心跳报文给服务端，若m秒内仍然未收到客户端的心跳报文，则视为离线，进行注销操作。

数据包：

| Name       | Type   | Description |
| ---------- | ------ | ----------- |
| playerName | string | 玩家名称    |
| msgId      | int    | 报文id      |

数据包中msgId是条件递增的，只有收到服务端的心跳反馈时才递增，否则msgId不变

### 2.2.2 下发

服务端接收到客户端的心跳报文后，发送心跳反馈报文给客户端。若客户端n秒内仍然未收到服务端的心跳反馈，则视为连接服务器失败。

数据包：

| Name       | Type   | Description |
| ---------- | ------ | ----------- |
| playerName | string | 玩家名称    |
| msgId      | int    | 报文id      |

## 2.3 玩家登录报文 PlayerLogIn

### 2.3.1 上传

登录请求。

数据包：

| Name       | Type   | Description |
| ---------- | ------ | ----------- |
| playerName | string | 玩家名称    |

### 2.3.2 下发

登录反馈。若客户端n秒内仍然未收到服务端的登录反馈，则视为登录服务器失败。

数据包：

| Name       | Type   | Description                                         |
| ---------- | ------ | --------------------------------------------------- |
| playerName | string | 玩家名称                                            |
| fail       | bool   | 登录是否失败                                        |
| error      | int    | 登录失败原因，0-成功，1-重复登录，...，255-未知原因 |

## 2.4 玩家注销报文 PlayerLogOut

### 2.4.1 上传

注销。该报文不需要反馈报文。

数据包：

| Name       | Type   | Description |
| ---------- | ------ | ----------- |
| playerName | string | 玩家名称    |

## 2.5 房间创建报文 RoomCreate

### 2.5.1 上传

创建房间请求。

数据包：

| Name       | Type   | Description |
| ---------- | ------ | ----------- |
| playerName | string | 玩家名称    |
| roomName   | string | 房间名称    |
| password   | string | 房间密码    |

### 2.5.2 下发

创建房间反馈。若客户端n秒内仍然未收到服务端的创建房间反馈，则视为创建房间请求超时。

数据包：

| Name       | Type   | Description                                                  |
| ---------- | ------ | ------------------------------------------------------------ |
| playerName | string | 玩家名称                                                     |
| roomName   | string | 房间名称                                                     |
| password   | string | 房间密码                                                     |
| roomId     | int    | 请求得到的房间号                                             |
| fail       | bool   | 创建房间是否失败                                             |
| error      | int    | 创建房间失败原因，0-成功，1-已达服务器最大可供房间数量，...，255-未知原因 |

## 2.6 房间进入报文 RoomEnter

### 2.6.1 上传

房间进入请求。

数据包：

| Name       | Type   | Description |
| ---------- | ------ | ----------- |
| playerName | string | 玩家名称    |
| roomName   | string | 房间名称    |
| password   | string | 房间密码    |
| roomId     | int    | 房间号      |

### 2.6.2 下发

房间进入反馈。若客户端n秒内仍然未收到服务端的房间进入反馈，则视为房间进入请求超时。

数据包：

| Name       | Type   | Description                                             |
| ---------- | ------ | ------------------------------------------------------- |
| playerName | string | 玩家名称                                                |
| roomName   | string | 房间名称                                                |
| password   | string | 房间密码                                                |
| roomId     | int    | 房间号                                                  |
| fail       | bool   | 进入房间是否失败                                        |
| error      | int    | 进入房间失败原因，0-成功, 1-密码错误，...，255-未知原因 |

## 2.7 房间退出报文 RoomExit

### 2.7.1 上传

房间退出请求。无论服务端如何处理，客户端应该无条件退出房间页面。

数据包：

| Name       | Type   | Description |
| ---------- | ------ | ----------- |
| playerName | string | 玩家名称    |
| roomName   | string | 房间名称    |
| roomId     | int    | 房间号      |

### 2.7.2 下发

房间退出反馈。通知所有仍然在该房间的客户端有玩家退出房间了。

数据包：

| Name       | Type   | Description |
| ---------- | ------ | ----------- |
| playerName | string | 玩家名称    |
| roomName   | string | 房间名称    |
| roomId     | int    | 房间号      |

## 2.8 开始游戏 StartGame

### 2.8.1 上传

开始游戏请求。若该房间内所有玩家都确定开始了，则该房间开始游戏。

数据包：

| Name       | Type   | Description  |
| ---------- | ------ | ------------ |
| playerName | string | 玩家名称     |
| roomName   | string | 房间名称     |
| roomId     | int    | 房间号       |
| ready      | bool   | 是否准备就绪 |

### 2.8.1 下发

开始游戏反馈。只有当服务端收到了房间内所有玩家都准备就绪了，才开始游戏。

数据包：

| Name         | Type     | Description                               |
| ------------ | -------- | ----------------------------------------- |
| playerName   | string   | 玩家名称                                  |
| roomName     | string   | 房间名称                                  |
| roomId       | int      | 房间号                                    |
| playerCount  | int      | 玩家数量                                  |
| playersName  | string[] | 玩家名称列表，列表长度等于playerCount     |
| playersReady | bool[]   | 玩家准备就绪列表，列表长度等于playerCount |
| start        | bool     | 是否开始游戏                              |

## 2.9 玩家操作报文 Play

### 2.9.1 上传

玩家操作报文，每隔n秒发送1次，若m秒内仍然未收到服务端反馈，则视为请求超时。

数据包：

| Name       | Type   | Description                           |
| ---------- | ------ | ------------------------------------- |
| playerName | string | 玩家名称                              |
| roomName   | string | 房间名称                              |
| roomId     | int    | 房间号                                |
| msgId      | int    | 报文id                                |
| type       | int    | 玩家操作枚举，0-无, ...，255-未知操作 |
| data       | ?      | type操作对应的数据包                  |

数据包中msgId是该玩家操作报文序列的当前标识id，此id是条件递增的，只有收到了服务端同样类型的玩家操作报文反馈才执行递增；数据包中type表示各种操作，例如出牌操作，抽卡操作等，其对应的操作数据包data也需要根据不同的操作来设计。

### 2.9.2 下发

玩家操作报文反馈。

数据包：

| Name       | Type   | Description                                     |
| ---------- | ------ | ----------------------------------------------- |
| playerName | string | 玩家名称                                        |
| roomName   | string | 房间名称                                        |
| roomId     | int    | 房间号                                          |
| msgId      | int    | 报文id                                          |
| type       | int    | 玩家操作枚举，0-无, ...，255-未知操作           |
| data       | ?      | type操作对应的数据包                            |
| fail       | bool   | 玩家操作请求是否失败                            |
| error      | int    | 玩家操作请求失败原因，0-成功，...，255-未知原因 |

