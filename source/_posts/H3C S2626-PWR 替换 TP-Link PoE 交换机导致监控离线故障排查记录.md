---
layout: ops
title: H3C S2626-PWR 替换 TP-Link PoE 交换机导致监控离线故障排查记录
uuid: f0b9d018eed24652a7f1de5d28de89b3
date: 2026-06-03 17:54:01
tags:
---

# H3C S2626-PWR 替换 TP-Link PoE 交换机导致监控离线故障排查记录

## 一、故障背景

原网络环境：

* 路由器划分监控 VLAN 104
* 两台 TP-Link TL-SL1218MP PoE 交换机级联
* 海康录像机（NVR）
* 多个海康摄像头

网络拓扑：

```text
路由器
    │
TP-Link①
    │
TP-Link②（故障）
    │
海康录像机
```

由于其中一台 TP-Link TL-SL1218MP 损坏，更换为 H3C S2626-PWR。

更换后拓扑：

```text
路由器
    │
TP-Link①
    │
H3C S2626-PWR
    │
海康录像机
```

故障现象：

* TP-Link 下挂摄像头正常；
* H3C 下挂摄像头全部离线；
* 摄像头有 PoE 供电，但录像机无画面。

---

# 二、初步检查

### 1. 检查 PoE 状态

执行：

```shell
display poe interface
```

发现：

```text
Eth1/0/9 ~ Eth1/0/24
Status : delivering-power
```

说明：

* PoE 正常；
* 摄像头已经成功供电。

---

### 2. 检查端口状态

执行：

```shell
display interface brief
```

发现：

```text
Eth1/0/9~24  UP
PVID = 1
```

说明：

* 摄像头链路正常；
* 所有端口默认处于 VLAN1。

---

### 3. 检查监控 VLAN

已知：

```text
VLAN ID：104
IP：192.168.104.x
Gateway：192.168.104.1
Mask：255.255.255.0
```

执行：

```shell
display vlan 104
```

返回：

```text
VLAN(s) do(es) not exist.
```

说明：

H3C 上尚未创建 VLAN104。

---

# 三、创建 VLAN104

进入系统视图：

```shell
system-view
```

创建 VLAN：

```shell
vlan 104
```

查看：

```shell
display vlan 104
```

确认创建成功。

---

# 四、配置端口

## 摄像头端口（1~24）

```shell
interface range Ethernet1/0/1 to Ethernet1/0/24

port link-type access

port access vlan 104
```

查看：

```shell
display interface brief
```

确认：

```text
Eth1/0/1~24

PVID 104
```

---

## 初始错误配置

认为：

* GE1/0/25 接上级交换机；
* GE1/0/26 接海康录像机；

因此配置：

```shell
interface GigabitEthernet1/0/25
port link-type trunk
port trunk permit vlan 104

interface GigabitEthernet1/0/26
port link-type trunk
port trunk permit vlan 104
```

后改为：

```shell
interface GigabitEthernet1/0/25
port link-type access
port access vlan 104

interface GigabitEthernet1/0/26
port link-type access
port access vlan 104
```

但是：

监控依旧全部离线。

---

# 五、排查发现异常

执行：

```shell
display interface brief
```

发现：

```text
GE1/0/25 DOWN
GE1/0/26 DOWN
```

执行：

```shell
display mac-address vlan 104
```

发现：

* 没有 25、26 口 MAC 地址；
* 无流量通过。

说明：

25、26 口没有建立物理链路。

---

# 六、现场验证

将原本插在：

```text
GE1/0/25
GE1/0/26
```

上的网线，

改插：

```text
Eth1/0/23
Eth1/0/24
```

结果：

监控立即恢复。

说明：

* VLAN104 配置正确；
* 摄像头正常；
* NVR 正常；
* H3C 正常；
* 问题集中在 GE1/0/25、26。

---

# 七、最终定位原因

查看：

```shell
display interface GigabitEthernet1/0/25
```

输出：

```text
Media type is optical fiber
Port hardware type is No connector
```

说明：

GE1/0/25 工作于：

```text
Optical Fiber（光口模式）
```

而非：

```text
Copper（电口模式）
```

因此：

* RJ45 电口被禁用；
* 插网线后始终 DOWN；
* 无法建立链路。

---

# 八、最终原因

H3C S2626-PWR 的：

```text
GE1/0/25
GE1/0/26
```

为：

### Combo 光电复用口

结构：

```text
25 电口 ←→ 25 光口
26 电口 ←→ 26 光口
```

两者只能使用一种介质。

当前交换机被设置为：

```text
Optical Fiber Mode
```

导致：

RJ45 电口不可用。

---

# 九、最终解决方案

不再使用 25、26 Combo 口。

采用：

```text
Eth1/0/23 ← 上联 TP-Link
Eth1/0/24 ← 海康录像机
```

全部端口加入 VLAN104：

```shell
interface range Ethernet1/0/1 to Ethernet1/0/24

port link-type access

port access vlan 104
```

保存配置：

```shell
save
```

监控全部恢复正常。

---

# 十、经验总结

### 1. PoE 有电 ≠ 网络正常

需要同时检查：

```shell
display poe interface

display interface brief

display mac-address
```

---

### 2. 先看物理层，再看 VLAN

如果：

```text
Interface DOWN
```

优先排查：

* 网线；
* 接口；
* 光电模式；
* Shutdown；

不要急于修改 VLAN。

---

### 3. H3C 老型号 Combo 口容易踩坑

典型型号：

* S2626-PWR
* S2626-EI
* S3100
* S3600
* S5120

25、26 口通常为：

```text
Combo 光电复用口
```

需要确认：

```shell
display interface GigabitEthernet1/0/25
```

关注：

```text
Media type
```

是否为：

```text
optical fiber
```

还是：

```text
twisted pair
```

---

### 4. 更换交换机时优先查看

```shell
display interface brief

display poe interface

display vlan

display mac-address
```

可以快速定位：

* VLAN 问题；
* PoE 问题；
* 物理链路问题；
* Combo 口问题；
* 端口故障问题。

---

# 最终结论

此次故障并非：

* VLAN 配置错误；
* 海康录像机故障；
* 摄像头故障；
* PoE 供电问题；

真正原因是：

> H3C S2626-PWR 的 GE1/0/25、GE1/0/26 为 Combo 光电复用口，当前处于光口模式，导致 RJ45 电口失效，造成 H3C 下挂摄像头全部离线。

通过将上联和录像机改接普通百兆端口（23、24），监控系统恢复正常。
