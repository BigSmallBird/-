# task4 — VLAN 间互通：单臂路由 vs 三层交换机

两种做法解决同一个问题：

```
PCA 192.168.1.10/24 VLAN100 ── 交换机 ── PCB 192.168.2.10/24 VLAN200
```

两个 VLAN = 两个广播域 = 两个 IP 网段。要互通，必须有一台三层设备做各自网段的网关。

| | ① 单臂路由 | ② 三层交换机 |
|---|---|---|
| 网关在哪 | 路由器 F0/0 的子接口 | 三层交换机的 `interface vlan` |
| 占用物理口 | 1 个（靠 802.1Q 复用） | 0 个（虚拟接口） |
| 转发性能 | 软件转发，越低端越慢 | 硬件转发，快 |
| 现状 | 老方案，实验常考 | 现在企业网主流 |

---

## ① 单臂路由（Router-on-a-Stick）

### 拓扑

```
   PCA 192.168.1.10/24                        PCB 192.168.2.10/24
        VLAN100                                     VLAN200
          │ 1 号口                                    │ 9 号口
        ┌─┴──────────────────────────────────────────┴─┐
        │                  DCS3600                     │
        └──────────────────────┬───────────────────────┘
                        24 号口 TRUNK
                               │
                        F0/0   │   (子接口 F0/0.1 / F0/0.2)
                          ┌────┴────┐
                          │ DCR2611 │
                          └─────────┘
```

### DCS3600（二层交换机）完整命令

```
enable
configure terminal

! 创建 VLAN
vlan 100
exit
vlan 200
exit

! 1 号口接 PCA，划入 VLAN100
interface ethernet 0/0/1
 switchport mode access
 switchport access vlan 100
 exit

! 9 号口接 PCB，划入 VLAN200
interface ethernet 0/0/9
 switchport mode access
 switchport access vlan 200
 exit

! 24 号口连路由器，设成 TRUNK
interface ethernet 0/0/24
 switchport mode trunk
 switchport trunk allowed vlan all
 exit

end
write
```

> `switchport trunk allowed vlan all` 是默认值，可以不写。
> 有的 DCN 版本限制 VLAN 列表时用分号：`switchport trunk allowed vlan 100;200`。

### DCR2611（路由器）完整命令

```
enable
configure terminal

! 物理接口先打开（不配 IP）
interface fastethernet 0/0
 no shutdown
 exit

! 子接口 1：封装 802.1Q VLAN100，配 VLAN100 的网关
interface fastethernet 0/0.1
 encapsulation dot1Q 100
 ip address 192.168.1.1 255.255.255.0
 no shutdown
 exit

! 子接口 2：封装 802.1Q VLAN200，配 VLAN200 的网关
interface fastethernet 0/0.2
 encapsulation dot1Q 200
 ip address 192.168.2.1 255.255.255.0
 no shutdown
 exit

end
write
```

**要点**：子接口号（.1 / .2）只是本地编号，**真正决定 VLAN 的是 `encapsulation dot1Q` 后面的数字**。

### PC 配置

```
PCA：IP 192.168.1.10   掩码 255.255.255.0   网关 192.168.1.1
PCB：IP 192.168.2.10   掩码 255.255.255.0   网关 192.168.2.1
```

### 验证命令

```
DCR2611# show ip interface brief
DCR2611# show ip route
DCS3600# show vlan
DCS3600# show interfaces ethernet 0/0/24 switchport
```

### Kathara 对应（`on-a-stick/`）

| Cisco | Kathara（`on-a-stick/`） |
|---|---|
| `switchport access vlan 100` | `ovs-vsctl add-port sw1 eth0 tag=100` |
| `switchport mode trunk` | `ovs-vsctl add-port sw1 eth2 trunks=100,200` |
| `encapsulation dot1Q 100` | `ip link add link eth0 name eth0.100 type vlan id 100` |
| `ip address 192.168.1.1 255.255.255.0` | `ip addr add 192.168.1.1/24 dev eth0.100` |

---

## ② 三层交换机（VLANIF / SVI）

### 拓扑

```
   PCA 192.168.1.10/24                        PCB 192.168.2.10/24
        VLAN100                                     VLAN200
          │ 1 号口                                    │ 9 号口
        ┌─┴──────────────────────────────────────────┴─┐
        │                  DCS3600                     │
        └──────────────────────┬───────────────────────┘
                        24 号口 TRUNK
                               │
                        E0/0/24│
                          ┌────┴─────┐
                          │ DCRS552  │  interface vlan 100 / 200
                          └──────────┘
```

### DCS3600（二层交换机）完整命令

与①完全相同：建 VLAN，1 号口 access VLAN100，9 号口 access VLAN200，24 号口 trunk。

```
enable
configure terminal
vlan 100
exit
vlan 200
exit
interface ethernet 0/0/1
 switchport mode access
 switchport access vlan 100
 exit
interface ethernet 0/0/9
 switchport mode access
 switchport access vlan 200
 exit
interface ethernet 0/0/24
 switchport mode trunk
 switchport trunk allowed vlan all
 exit
end
write
```

### DCRS552（三层交换机）完整命令

```
enable
configure terminal

! 三层交换机上也要有这两个 VLAN
vlan 100
exit
vlan 200
exit

! 上联口设为 TRUNK，放行 VLAN100 / 200
interface ethernet 0/0/24
 switchport mode trunk
 switchport trunk allowed vlan all
 exit

! 虚接口 100：VLAN100 的网关
interface vlan 100
 ip address 192.168.1.1 255.255.255.0
 no shutdown
 exit

! 虚接口 200：VLAN200 的网关
interface vlan 200
 ip address 192.168.2.1 255.255.255.0
 no shutdown
 exit

! 打开三层转发（很多型号默认关闭）
ip routing

end
write
```

**要点**：

- `interface vlan 100` 不是物理口，是交换机内部的一个虚拟接口，VLAN100 里所有 PC 的网关就是它；
- 必须有 `ip routing`，否则 VLANIF 有 IP 也不会互相转发；
- 这条 trunk 与①的 trunk 完全相同，区别只是对端从"路由器的子接口"变成了"交换机自己的 VLANIF"。

### PC 配置

```
PCA：IP 192.168.1.10   掩码 255.255.255.0   网关 192.168.1.1
PCB：IP 192.168.2.10   掩码 255.255.255.0   网关 192.168.2.1
```

### 验证命令

```
DCRS552# show ip interface brief
DCRS552# show ip route
DCRS552# show vlan
DCRS552# show interfaces vlan 100
```

### Kathara 对应（`l3-switch/`）

| Cisco | Kathara（`l3-switch/`） |
|---|---|
| `interface vlan 100` | `ip link add link eth0 name eth0.100 type vlan id 100` |
| `ip address 192.168.1.1 255.255.255.0` | `ip addr add 192.168.1.1/24 dev eth0.100` |
| `ip routing` | `sysctl net.ipv4.ip_forward=1` |

---

## 三、目录结构

```
task4/
├── on-a-stick/      ← ① 单臂路由，真交换机（Linux/Kali）
│   ├── lab.conf
│   ├── sw1.startup  OVS：eth0 tag=100，eth1 tag=200，eth2 trunk
│   ├── r1.startup   路由器：eth0.100 / eth0.200
│   └── pca / pcb.startup
│
├── l3-switch/       ← ② 三层交换机，真交换机（Linux/Kali）
│   ├── lab.conf
│   ├── sw1.startup  OVS：二层交换机
│   ├── l3sw.startup 三层交换机：eth0.100 / eth0.200 + ip_forward
│   └── pca / pcb.startup
│
├── macos/           ← 无交换机版，macOS/OrbStack 可跑
│   ├── lab.conf
│   ├── r1.startup   同时对应①②的三层设备
│   └── pca / pcb.startup（自己打 802.1Q 标签）
│
└── README.md
```

## 四、怎么跑

### Linux / Kali（真交换机，推荐交作业用）

```bash
cd ~/计算机网络实验/task4/on-a-stick
kathara lstart

kathara exec pca -- ping -c 4 192.168.2.10

kathara lclean
```

```bash
cd ~/计算机网络实验/task4/l3-switch
kathara lstart

kathara exec pca -- ping -c 4 192.168.2.10

kathara lclean
```

查看 OVS 的 VLAN 配置（对应 `show vlan` / `show interfaces switchport`）：

```bash
kathara exec sw1 -- ovs-vsctl show
kathara exec sw1 -- ovs-vsctl list port
```

### macOS / OrbStack（无交换机等价版）

```bash
cd ~/计算机网络实验/task4/macos
K=/opt/homebrew/Caskroom/miniconda/base/envs/kathara-env/bin/python
$K -m kathara lstart --noterminals
$K -m kathara exec pca -- ping -c 4 192.168.2.10
$K -m kathara lclean
```

> `on-a-stick/` 和 `l3-switch/` 里用了 OVS 交换机容器，在 OrbStack 上会触发
> 广播风暴，`sw1.startup` 已加内核检测直接 `exit 1`。Mac 上请用 `macos/`。

## 五、已经验证过什么

在 `macos/` 上实际跑过（与本机等价的数据面）：

```
$ kathara exec r1 -- ip -4 -o addr
2: eth0.100    inet 192.168.1.1/24          ← 对应 f0/0.1 / Vlanif100
3: eth0.200    inet 192.168.2.1/24          ← 对应 f0/0.2 / Vlanif200

$ kathara exec pca -- ping -c 4 192.168.2.10
64 bytes from 192.168.2.10: icmp_seq=1 ttl=63 time=0.102 ms
64 bytes from 192.168.2.10: icmp_seq=2 ttl=63 time=0.174 ms
64 bytes from 192.168.2.10: icmp_seq=3 ttl=63 time=0.221 ms
64 bytes from 192.168.2.10: icmp_seq=4 ttl=63 time=0.206 ms
4 packets transmitted, 4 received, 0% packet loss
```

ttl=63 = 经过了 1 台三层设备，说明流量确实是从 VLAN100 → 网关 → VLAN200。

抓包确认标签是真的：

```
$ kathara exec pcb -- tcpdump -e -n -i eth0
ethertype 802.1Q (0x8100), vlan 100:
  192.168.1.10 > 192.168.1.1: ICMP echo request
```

并且在 PCB 的 VLAN200 接口（`eth0.200`）上抓不到这些 VLAN100 的帧 —— VLAN 隔离生效。

> `on-a-stick/` 和 `l3-switch/` 两个 Linux 版只做了 `lstart --print` 配置校验
> （均通过），没有在本机执行 —— 因为它们的 OVS 部分只能在 Linux 上跑。

## 六、截图清单

| 编号 | 内容 | 命令 |
|---|---|---|
| ① | PCA ping PCB 通，TTL=63 | `kathara exec pca -- ping -c 4 192.168.2.10` |
| ① 附 | 路由器子接口 | `kathara exec r1 -- ip -4 addr` |
| ② | PCA ping PCB 通，TTL=63 | `kathara exec pca -- ping -c 4 192.168.2.10` |
| ② 附 | 三层交换机 VLANIF | `kathara exec l3sw -- ip -4 addr` |

两份作业要的"完整命令"在上面的 Cisco 配置块里，可以直接抄进报告。
