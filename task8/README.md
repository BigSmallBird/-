# task8 — DCR2611 静态路由实验

把图里的 Cisco 设备映射成 Kathara 容器，配置两台路由器的接口地址和静态路由，
让 PC A 与 PC B 互通。

## 1. 拓扑与设备对应

```
  192.168.1.0/24                211.1.1.0/24                211.2.2.0/24
  PC A ---- DCS3600 ---- DCR2611 ====== DCR2611 ---- DCRS552 ---- PC B
 .10        (交换)       .1  F0/0    S0/2  .1   .2 S0/3  F0/0 .1        .2
```

| 图中设备 | Kathara 名称 | 说明 |
|---|---|---|
| 左侧 DCR2611 | `r1` | 路由器，开启 IPv4 转发 |
| 左侧 DCS3600 | — | 二层交换机，本版本用直连链路代替 |
| PC A | `pca` | 192.168.1.10/24，网关 192.168.1.1 |
| 右侧 DCR2611 | `r2` | 路由器，开启 IPv4 转发 |
| 右侧 DCRS552 | — | 二层交换机，本版本用直连链路代替 |
| PC B | `pcb` | 211.2.2.2/24，网关 211.2.2.1 |

### 接口对照表

| 设备 | 图中接口 | Kathara 接口 | IP |
|---|---|---|---|
| r1 | F0/0 | eth0 | 192.168.1.1/24 |
| r1 | S0/2 | eth1 | 211.1.1.1/24 |
| r2 | S0/3 | eth1 | 211.1.1.2/24 |
| r2 | F0/0 | eth0 | 211.2.2.1/24 |
| pca | — | eth0 | 192.168.1.10/24 |
| pcb | — | eth0 | 211.2.2.2/24 |

## 2. 为什么默认版没有交换机

DCS3600 和 DCRS552 每台只接**一个路由器 + 一台 PC**，交换机上没有任何 VLAN、
ACL、生成树之类的策略。这种情况下交换机只是把两条线接在一起，Kathara 的
collision domain（同一个 `netA` / `netB`）本身就是同一件事，
所以默认版把「路由器—交换机—PC」压成了一条直连链路。

**三层配置和实验结果完全一样**：接口地址、掩码、网关、静态路由一个字都不变。

如果你需要在报告里体现交换机（抓包、`show mac-address-table`、VLAN 实验），
用 `with-switches/` 版本，但那里必须跑在 Linux 上，见第 6 节。

## 3. 静态路由

两边各自只知道直连的两条网段，所以要手工告诉它们对方网段怎么走：

| 设备 | 目的网段 | 掩码 | 下一跳 |
|---|---|---|---|
| r1 | 211.2.2.0 | 255.255.255.0 | 211.1.1.2（r2 的 S0/3） |
| r2 | 192.168.1.0 | 255.255.255.0 | 211.1.1.1（r1 的 S0/2） |
| pca | 0.0.0.0 | 0.0.0.0 | 192.168.1.1（默认网关） |
| pcb | 0.0.0.0 | 0.0.0.0 | 211.2.2.1（默认网关） |

对应 Cisco 命令：

```
r1(config)# ip route 211.2.2.0 255.255.255.0 211.1.1.2
r2(config)# ip route 192.168.1.0 255.255.255.0 211.1.1.1
```

对应 Linux（`*.startup` 里用的就是这条）：

```
ip route replace 211.2.2.0/24 via 211.1.1.2
```

## 4. 启动

```bash
cd ~/计算机网络实验/task8
/opt/homebrew/Caskroom/miniconda/base/envs/kathara-env/bin/python -m kathara lstart
```

容器启动后 startup 脚本会自动配好地址和路由，通常几秒钟。

## 5. 验证

```bash
K="/opt/homebrew/Caskroom/miniconda/base/envs/kathara-env/bin/python -m kathara"

# 1) 先看本机路由表，确认静态路由生效
$K exec r1 -- ip route
$K exec r2 -- ip route

# 2) PC A ping PC B（跨两个路由器）
$K exec pca -- ping -c 4 211.2.2.2

# 3) 看路径，应该经过 192.168.1.1 -> 211.1.1.2 -> 211.2.2.1
$K exec pca -- traceroute -n 211.2.2.2

# 4) 反向也要通
$K exec pcb -- ping -c 4 192.168.1.10
```

期望结果：

- `ping` 4 个包全通，`0% packet loss`；
- `traceroute` 出现两跳：`192.168.1.1`、`211.1.1.2`（最后一跳是目标）。

如果 ping 不通，按顺序查：

```bash
$K exec r1 -- ip addr          # 接口地址有没有配上
$K exec r1 -- ip route         # 静态路由在不在
$K exec r1 -- sysctl net.ipv4.ip_forward   # 必须是 1
```

## 6. 带交换机的版本（Linux 专用）

`with-switches/` 用 `kathara/openvswitch` 还原两台交换机：

```bash
cd ~/计算机网络实验/task8/with-switches
/opt/homebrew/Caskroom/miniconda/base/envs/kathara-env/bin/python -m kathara lstart
```

> ⚠️ **不要在 macOS/OrbStack 上跑这个版本。**
> OrbStack 的 `katharanp` bridge 插件会把这些端口做 hairpin 反射，
> 两个 OVS 端口属于同一个 VLAN 时会形成无限广播环路：几秒钟内计数涨到上亿，
> CPU 打满，机器发烫。
>
> 因此 `sw1.startup` / `sw2.startup` 开头加了一道保护：检测到 OrbStack 内核
> 就直接 `exit 1`，OVS 根本不会配起来，不会起风。
>
> 想真正看到交换机行为，请在独立的 Kali/Linux 虚拟机（Docker Engine + 
> `kathara/katharanp_vde` 插件）或 Linux 物理机上跑。

## 7. 停止

```bash
cd ~/计算机网络实验/task8
/opt/homebrew/Caskroom/miniconda/base/envs/kathara-env/bin/python -m kathara lclean
```

> 如果机器突然发烫，先执行 `lclean`，再用 `docker ps` 确认没有残留容器。
