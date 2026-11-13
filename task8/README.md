# task8 — 实验八：基础路由 + NAT

作业要求：

> ① 基础路由配置完成后，PCA ping 211.1.1.2 **通**、ping 211.2.2.2 **不通**。
> ② 配置 NAT 之后，PCA ping 211.2.2.2 **成功**。

这个目录就是按这两步搭的：

| 目录 | 阶段 | 内容 |
|---|---|---|
| `task8/`（根目录） | **① 基础路由** | 接口 IP + 双向静态路由 + 公网侧私有源过滤 |
| `task8/nat/` | **② 加 NAT** | 同①，只在 r1 上多一条 SNAT |
| `task8/with-switches/` | 带交换机的 Linux 版 | 阶段①，需要 Kali/Linux |

---

## 1. 拓扑与接口

```
 192.168.1.0/24              211.1.1.0/24               211.2.2.0/24
 PCA .10 ── DCS3600 ── DCR2611 ══════ DCR2611 ── DCRS552 ── PCB .2
                       F0/0 .1   S0/2 .1  .2 S0/3   F0/0 .1
```

| 设备 | 图中接口 | Kathara | IP |
|---|---|---|---|
| r1（左 DCR2611） | F0/0 | eth0 | 192.168.1.1/24 |
| r1 | S0/2 | eth1 | 211.1.1.1/24 |
| r2（右 DCR2611） | S0/3 | eth1 | 211.1.1.2/24 |
| r2 | F0/0 | eth0 | 211.2.2.1/24 |
| PCA | — | eth0 | 192.168.1.10/24，网关 192.168.1.1 |
| PCB | — | eth0 | 211.2.2.2/24，网关 211.2.2.1 |

两台交换机各只接「一个路由器 + 一台 PC」，没有任何 VLAN 策略，
所以默认版用直连链路代替（数据面完全等价）。带交换机的版本在 `with-switches/`。

---

## 2. 为什么 ① 会是「通一个、不通另一个」

这里有一个容易被忽略的点，先讲清楚，否则配出来对不上作业。

### 211.1.1.2 为什么需要回程路由

211.1.1.2 是 **r2 自己的接口地址**。PCA ping 它时：

1. 请求经 r1 转发到 r2；
2. r2 生成回包，源地址是 **211.1.1.2**，目的地址是 **192.168.1.10**；
3. r2 要发这个回包，必须**有一条到 192.168.1.0/24 的路由**。

所以「基础路由配置」里，**r2 必须有回程路由**，否则连 211.1.1.2 都 ping 不通。

### 但有了回程路由，211.2.2.2 也会通

如果 r2 有回程路由，PCA → 211.2.2.2 的去程、PCB 的回程都成立，两个 ping 全通。
这就是你之前遇到的「第一题我直接就 ping 通 211.2.2.2 了」。

### 作业描述的真实含义

作业里 ① 的「211.2.2.2 不通」，在现实网络中对应的是：
**公网路由器不会转发源地址是私有地址（192.168.x.x）的包**。
这是 ISP 普遍部署的入口过滤（RFC1918 filtering / uRPF），也是 NAT 存在的根本原因。

所以我们在 r2 上加一条过滤规则，精确复现这个现象：

```bash
iptables -A FORWARD -o eth0 -s 192.168.1.0/24 -j DROP
```

- 只挂在 **FORWARD 链**（只拦“转发”的包），
  所以 r2 自己回应 211.1.1.2 的包不受影响；
- 在真机上对应的是「在 F0/0 **出方向**应用 ACL」，
  这样也不会误伤到路由器自己的回应。

加上 NAT 后，源地址变成 211.1.1.1，这条规则不再命中，211.2.2.2 就通了。
**这就是实验要证明的东西**：NAT 不是为了“能出去”，而是为了让回程认得你。

---

## 3. 阶段①：基础路由（根目录）

### Kathara 配置

`r1.startup`：

```bash
ip addr add 192.168.1.1/24 dev eth0      # F0/0
ip link set eth0 up
ip addr add 211.1.1.1/24 dev eth1        # S0/2
ip link set eth1 up
ip route replace 211.2.2.0/24 via 211.1.1.2
```

`r2.startup`：

```bash
ip addr add 211.2.2.1/24 dev eth0        # F0/0
ip link set eth0 up
ip addr add 211.1.1.2/24 dev eth1        # S0/3
ip link set eth1 up
ip route replace 192.168.1.0/24 via 211.1.1.1

# 模拟公网拒收私有源地址（见第 2 节）
iptables -A FORWARD -o eth0 -s 192.168.1.0/24 -j DROP
```

### DCN 真机命令

r1：

```
enable
configure terminal
interface fastethernet 0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
 exit
interface serial 0/2
 ip address 211.1.1.1 255.255.255.0
 no shutdown
 exit
ip route 211.2.2.0 255.255.255.0 211.1.1.2
end
write
```

r2：

```
enable
configure terminal
interface serial 0/3
 ip address 211.1.1.2 255.255.255.0
 no shutdown
 exit
interface fastethernet 0/0
 ip address 211.2.2.1 255.255.255.0
 no shutdown
 exit
ip route 192.168.1.0 255.255.255.0 211.1.1.1

! 模拟公网对私有源地址的过滤（F0/0 出方向）
access-list 100 deny ip 192.168.1.0 0.0.0.255 any
access-list 100 permit ip any any
interface fastethernet 0/0
 ip access-group 100 out
 exit
end
write
```

> 不同 DCN 版本的 ACL 应用命令可能略有差别（`ip access-group` / `firewall`），
> 以设备的 `?` 提示为准。

### 实测结果

```
$ kathara exec pca -- ping -c 3 211.1.1.2
3 packets transmitted, 3 received, 0% packet loss

$ kathara exec pca -- ping -c 3 211.2.2.2
3 packets transmitted, 0 received, 100% packet loss
```

r2 上的过滤规则确实在命中：

```
$ kathara exec r2 -- iptables -L FORWARD -v -n
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target prot opt in     out     source               destination
    3   252 DROP   0   --  *      eth0    192.168.1.0/24       0.0.0.0/0
```

---

## 4. 阶段②：配置 NAT（`nat/`）

只有 r1 多一条：

```bash
# 把从 eth1（S0/2，外网口）出去的包的源地址，统一换成 211.1.1.1
iptables -t nat -A POSTROUTING -o eth1 -j SNAT --to-source 211.1.1.1
```

### DCN 真机命令（在 r1 上）

```
enable
configure terminal

! 允许被转换的内网地址
access-list 1 permit 192.168.1.0 0.0.0.255

interface fastethernet 0/0
 ip nat inside
 exit
interface serial 0/2
 ip nat outside
 exit

ip nat inside source list 1 interface serial 0/2 overload

end
write
```

- `ip nat inside`：内网口（F0/0）
- `ip nat outside`：外网口（S0/2）
- `overload`：端口复用（NAPT），多个内网主机共用一个公网地址

### 实测结果

```
$ kathara exec pca -- ping -c 3 211.2.2.2
3 packets transmitted, 3 received, 0% packet loss

$ kathara exec pca -- ping -c 3 211.1.1.2
3 packets transmitted, 3 received, 0% packet loss
```

在 r2 的外网口抓包，源地址已经变成公网地址：

```
$ kathara exec r2 -- timeout 5 tcpdump -n -i eth1 icmp
IP 211.1.1.1 > 211.2.2.2: ICMP echo request
IP 211.2.2.2 > 211.1.1.1: ICMP echo reply
```

PC A 的真实地址 192.168.1.10 **完全不出现在公网上**，这就是 NAT 的效果。
PCB 看到的源是 211.1.1.1，回包自然回到 r1，再由连接跟踪还原给 PCA。

> TTL：PCA → PCB 经过两台路由器，收到时是 **62**。

---

## 5. 目录结构

```
task8/
├── README.md
├── lab.conf            ┐
├── r1.startup          │ 阶段①：基础路由
├── r2.startup          │ （含公网私有源过滤）
├── pca.startup         │
├── pcb.startup         ┘
├── nat/                ┐ 阶段②：同① + r1 上的 SNAT
│   ├── lab.conf        │
│   ├── r1.startup      │
│   ├── r2.startup      │
│   ├── pca.startup     │
│   └── pcb.startup     ┘
├── with-switches/      ← Linux 版（真 OVS 交换机），阶段①
└── 截屏2026-11-13 22.13.53.png
```

---

## 6. 怎么跑 + 截图清单

### 阶段①

```bash
cd ~/计算机网络实验/task8
kathara lstart

kathara exec pca -- ping -c 4 211.1.1.2      # 截图 1
kathara exec pca -- ping -c 4 211.2.2.2      # 截图 1（不通）
kathara exec r2 -- iptables -L FORWARD -v -n # 可选：证明过滤在生效

kathara lclean
```

### 阶段②

```bash
cd ~/计算机网络实验/task8/nat
kathara lstart

kathara exec pca -- ping -c 4 211.2.2.2      # 截图 2
kathara exec r1 -- iptables -t nat -L POSTROUTING -v -n   # 可选

kathara lclean
```

| 截图 | 内容 | 判据 |
|---|---|---|
| ① | PCA ping 211.1.1.2 | 3/4 以上 received |
| ① | PCA ping 211.2.2.2 | 0 received / 100% loss |
| ② | PCA ping 211.2.2.2 | 全部 received |

> 作业要求在真机上截图时，就是两台 PC 的 cmd 窗口；
> Kathara 版用来先验证配置逻辑，输出可以对照。

---

## 7. 常见问题

| 现象 | 原因 |
|---|---|
| 阶段① 两个都通 | 用的还是旧的 `r2.startup`，过滤规则没加；或手动删过规则 |
| 阶段① 两个都不通 | r2 少了回程静态路由（211.1.1.2 也 ping 不通就是这个原因） |
| 阶段② 还是不通 | r1 的 SNAT 没生效；`ip_forward` 不是 1 |
| 阶段② 通了但抓包源还是 192.168.1.10 | 规则加错了链/接口（应在 POSTROUTING，出接口 eth1） |
| 想还原“两个都通” | 删掉 r2 的 `iptables -A FORWARD ... DROP` |

关掉所有实验：

```bash
cd ~/计算机网络实验/task8 && kathara lclean
```

---

## 8. 带交换机的版本（Linux 专用）

`with-switches/` 用 `kathara/openvswitch` 还原 DCS3600 / DCRS552，配置与阶段①相同。
要在 Kali/Linux 上跑：

```bash
cd ~/计算机网络实验/task8/with-switches
kathara lstart
```

> ⚠️ **不要在 macOS/OrbStack 上跑。** OVS + OrbStack 的 hairpin 反射会形成
> 广播风暴。`sw1.startup` / `sw2.startup` 开头有内核检测，检测到 OrbStack 会直接退出。

做阶段②时，在 `with-switches/r1.startup` 末尾加上同一行 SNAT 即可。
