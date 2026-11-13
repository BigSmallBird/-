# task3 — 同网段 / 跨网段互访实验

## 一、三张图分别在讲什么

这三张图是**同一个目标**（让两个不同 IP 网段的 PC 互相通信）的三种实现方式，
课本按"由简到繁"排列：

### 图 1：单臂路由（Router-on-a-Stick）

```
PC A ──┐
       ├── DCS3600 ══TRUNK══ DCR2611
PC B ──┘   (VLAN100/200)
```

- 只有**一根**物理链路从交换机连到路由器；
- 交换机上 VLAN 100 = 1 号口（PC A）、VLAN 200 = 9 号口（PC B）、24 号口设为 TRUNK；
- 路由器那一个 F0/0 上划出两个**子接口**：
  `F0/0.100` 配 192.168.1.1/24，`F0/0.200` 配 192.168.2.1/24；
- 两个 VLAN 的流量都挤在这根 trunk 上，靠 802.1Q 标签区分。

Cisco 关键配置：

```
DCS3600(config)# vlan 100
DCS3600(config-vlan)# vlan 200
DCS3600(config)# interface ethernet 0/0/1
DCS3600(config-if)# switchport access vlan 100
DCS3600(config)# interface ethernet 0/0/9
DCS3600(config-if)# switchport access vlan 200
DCS3600(config)# interface ethernet 0/0/24
DCS3600(config-if)# switchport mode trunk

DCR2611(config)# interface f0/0.100
DCR2611(config-subif)# encapsulation dot1Q 100
DCR2611(config-subif)# ip address 192.168.1.1 255.255.255.0
DCR2611(config)# interface f0/0.200
DCR2611(config-subif)# encapsulation dot1Q 200
DCR2611(config-subif)# ip address 192.168.2.1 255.255.255.0
```

**特点**：省钱，路由器只有一个口也能连多个 VLAN；缺点是所有跨网段流量都走这一根线。

### 图 2：多接口路由（传统方式，本目录 `image.png`）

```
PC A 192.168.1.10 ┐                    ┌ PCC 192.168.2.10
PC B 192.168.1.11 ┴ DCRS552 ─ DCR2611 ─ DCS3600 ┴ PCD 192.168.2.11
                     TP0 192.168.1.1      TP1 192.168.2.1
```

- 路由器有**两个物理接口**，一个网段一个；
- 每个网段一台二层交换机，交换机上不划 VLAN，所有口同一广播域；
- 最直白：**一个网段 = 一个接口 = 一个广播域**。

**这台实验就是按这个拓扑做的。**

### 图 3：三层交换机（VLANIF / SVI）

```
PC A VLAN100 ─┐                    ┌─ DCRS552（三层交换机）
              ├── DCS3600 ═TRUNK══┤    Vlanif100 192.168.1.1
PC B VLAN200 ─┘   (100,200)        └    Vlanif200 192.168.2.1
```

- 二层交换机只做接入，把 VLAN 100/200 通过 trunk 送到三层交换机；
- 三层交换机上每个 VLAN 建一个**虚拟接口**（VLANIF），它就是该网段的网关；
- 等价于把这台三层交换机当成"很多个路由器接口"用，但转发在硬件里做，快得多。
- 这是现在企业网最常用的做法，逐渐取代图 1 的单臂路由。

Cisco 关键配置：

```
DCS3600(config)# interface ethernet 0/0/24
DCS3600(config-if)# switchport mode trunk

DCRS552(config)# vlan 100
DCRS552(config-vlan)# vlan 200
DCRS552(config)# interface ethernet 0/0/24
DCRS552(config-if)# switchport mode trunk
DCRS552(config)# interface vlan 100
DCRS552(config-if)# ip address 192.168.1.1 255.255.255.0
DCRS552(config)# interface vlan 200
DCRS552(config-if)# ip address 192.168.2.1 255.255.255.0
DCRS552(config)# ip routing
```

## 二、这个实验要证明什么

| 对比项 | 同网段 | 不同网段 |
|---|---|---|
| 例子 | PC A `192.168.1.10` → PC B `192.168.1.11` | PC A `192.168.1.10` → PC C `192.168.2.10` |
| 走哪 | 二层直接转发，交换机查 MAC 表 | 发给自己网关 → 路由器查路由表 → 转发 |
| 网关 | 用不到 | 必须配，不配就 ping 不通 |
| `ping` 的 TTL | **64**（没经过路由器） | **63**（经过 1 台路由器，TTL 减 1） |
| `traceroute` | 1 跳直达 | 第 1 跳是网关 |

**TTL 64 / 63 就是两张截图的判据** —— 这是最直观、老师最认的证据。

## 三、目录结构

```
task3/
├── lab.conf          ← 带 OVS 交换机的版本（Linux/Kali 用，macOS 会被保护拦住）
├── sw1/2.startup     交换机：三个端口同属 VLAN 1
├── r1.startup        路由器两个接口 192.168.1.1 / 192.168.2.1
├── pca/pcb/pcc/pcd.startup
├── image.png         课本原图
├── safe/             ← macOS/OrbStack 用的版本（无交换机，直接用 collision domain）
│   ├── lab.conf
│   └── *.startup
└── README.md
```

## 四、在 macOS 上跑（用 safe/）

```bash
cd ~/计算机网络实验/task3/safe
K=/opt/homebrew/Caskroom/miniconda/base/envs/kathara-env/bin/python
$K -m kathara lstart --noterminals
```

### 截图 1：同网段

```bash
$K -m kathara exec pca -- ping -c 4 192.168.1.11
```

```
64 bytes from 192.168.1.11: icmp_seq=1 ttl=64 time=0.138 ms
...
4 packets transmitted, 4 received, 0% packet loss
```

### 截图 2：不同网段

```bash
$K -m kathara exec pca -- ping -c 4 192.168.2.10
$K -m kathara exec pca -- traceroute -n 192.168.2.10
```

```
64 bytes from 192.168.2.10: icmp_seq=1 ttl=63 time=0.157 ms
...
4 packets transmitted, 4 received, 0% packet loss
```

```
 1  192.168.1.1  0.531 ms      ← 先到网关
 2  192.168.2.10 0.285 ms      ← 再到目标
```

### 辅助截图（可选，加分）

```bash
$K -m kathara exec pca -- ip route            # 看到 default via 192.168.1.1
$K -m kathara exec r1  -- ip route            # 两条直连路由
$K -m kathara exec r1  -- sysctl net.ipv4.ip_forward   # = 1
```

## 五、为什么 safe 版没有交换机

图 2 里的 sw1/sw2 每台只接 **2 台 PC + 1 个路由器口**，三个口都在同一个 VLAN，
既没划 VLAN 也没配 ACL —— 它做的事只有一个：把这四条线接成一个广播域。

在 Kathara 里，**一个 collision domain 本来就是同一个二层广播域**，
所以 `netA + netB + netAB` 合并成 `netAB` 后，二层行为完全等价，
而三层配置（IP、掩码、网关）一个字都不用改。

## 六、要还原真实交换机（Linux/Kali）

用上一层 `task3/lab.conf`（`sw1`/`sw2` 是 `kathara/openvswitch`）：

```bash
cd ~/计算机网络实验/task3
kathara lstart
```

> ⚠️ **不能在 macOS/OrbStack 上跑。** OrbStack 的 katharanp bridge 插件会
> hairpin 反射，多个同 VLAN 端口会形成广播环路，几秒钟 CPU 打满、机器发烫。
> `sw1.startup` / `sw2.startup` 开头已加保护：检测到 OrbStack 直接 `exit 1`，
> OVS 不会起来，也不会起风。请在独立 Kali/Linux 虚拟机里跑这个版本。

## 七、清理

```bash
$K -m kathara lclean -d ~/计算机网络实验/task3/safe
```
