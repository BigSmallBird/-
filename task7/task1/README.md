# task7/task1 — 路由器 DHCP 服务（图 1）

```
                 Router A 2621
                    F0/0 192.168.1.1/24
                       │ (DHCP 服务器)
                 ┌─────┴─────┐
                 │ Switch A  │  DCS3600（二层，透明转发）
                 └──┬─────┬──┘
              E0/3 │     │ E0/6
                ┌──┴─┐ ┌─┴──┐
                │PCA │ │PCB │   两台都是“自动获取 IP”
                └────┘ └────┘
```

一台路由器用 **DHCP 地址池**给下面两台 PC 自动分配：

| 项目 | 值 |
|---|---|
| 网段 | 192.168.1.0/24 |
| 网关（路由器 F0/0） | 192.168.1.1 |
| 地址池 | 192.168.1.100 ～ 192.168.1.200 |
| DNS | 192.168.1.1 |
| 租期 | 12 小时 |

---

## 一、Router A（DCR-2621）完整命令

```
enable
configure terminal

! 1. 先给 F0/0 配上网关地址
interface fastethernet 0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
 exit

! 2. 打开 DHCP 服务
service dhcp

! 3. 把路由器自己和保留地址排除在地址池外
ip dhcp excluded-address 192.168.1.1 192.168.1.99

! 4. 建地址池
ip dhcp pool LAN
 network 192.168.1.0 255.255.255.0
 default-router 192.168.1.1
 dns-server 192.168.1.1
 lease 0 12 0
 exit

end
write
```

### 每行在干什么

| 命令 | 作用 | 对应 DHCP 报文里的 |
|---|---|---|
| `service dhcp` | 在路由器上启用 DHCP 服务器进程 | — |
| `ip dhcp excluded-address` | 不参与自动分配的地址 | — |
| `ip dhcp pool LAN` | 建一个地址池，名字随意 | — |
| `network ...` | 告诉服务器“这个池属于哪个网段、掩码多少” | 决定从哪个池分配 |
| `default-router` | 下发给 PC 的网关 | **option 3** |
| `dns-server` | 下发给 PC 的 DNS | **option 6** |
| `lease 0 12 0` | 租期 0 天 12 小时 0 分 | 租约时间 |

> **最容易错的点**：`network` 里的网段必须和 F0/0 的 IP **同网段**。
> 如果 F0/0 是 192.168.1.1/24，池子写成 192.168.2.0，PC 永远拿不到地址。

### 查看

```
show ip dhcp binding          ! 已经分配出去的租约
show ip dhcp pool             ! 池子配置和剩余地址
show running-config | include dhcp
```

---

## 二、Switch A（DCS3600）配置

这张图里交换机的所有口都在**同一个 VLAN（默认 VLAN 1）**，所以严格来说不用配，
插上就能通。想写得完整一点：

```
enable
configure terminal

interface ethernet 0/0/3          ! 接 PCA
 switchport mode access
 switchport access vlan 1
 exit

interface ethernet 0/0/6          ! 接 PCB
 switchport mode access
 switchport access vlan 1
 exit

interface ethernet 0/0/7          ! 接路由器
 switchport mode access
 switchport access vlan 1
 exit

end
write
```

**重点是三个口必须在同一个 VLAN。** 如果 PCA 的口被划到 VLAN 10、路由器的口在 VLAN 1，
DHCP DISCOVER 广播根本到不了路由器，PC 只能拿到 169.254.x.x 的自动私有地址。

---

## 三、PC 端（Windows）

网卡属性里把 **IP 地址、子网掩码、默认网关、DNS 服务器全部选“自动获得”**，然后：

```cmd
ipconfig /release
ipconfig /renew
ipconfig /all
```

`ipconfig /all` 里要能看出这几行：

```
   DHCP Enabled. . . . . . . . . . . : Yes
   Autoconfiguration Enabled . . . . : Yes

   IPv4 Address. . . . . . . . . . . : 192.168.1.100(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Lease Obtained. . . . . . . . . . : 2026年11月14日 10:20:31
   Lease Expires . . . . . . . . . . : 2026年11月14日 22:20:31

   Default Gateway . . . . . . . . . : 192.168.1.1
   DHCP Server . . . . . . . . . . . : 192.168.1.1
   DNS Servers . . . . . . . . . . . : 192.168.1.1
```

截图要点：

- 必须有 `DHCP Enabled: Yes`；
- `DHCP Server` 有地址（证明是从路由器拿的）；
- `Default Gateway` 和 `DNS Servers` 都有值（证明 option 3 / option 6 下发了）；
- 出现 `169.254.x.x` = 没拿到 DHCP，见下面“常见问题”。

---

## 四、Kathara 版（本机已跑通）

`kathara/base` 里没有 ISC `dhcpd`，用的是自带的 **dnsmasq**，它一个进程同时干 DHCP 和 DNS。

| Cisco | Kathara（dnsmasq） |
|---|---|
| `ip dhcp pool LAN` | `/etc/dnsmasq.conf` 里的 `dhcp-range` |
| `network 192.168.1.0 255.255.255.0` | `dhcp-range=...,255.255.255.0,...` |
| `default-router 192.168.1.1` | `dhcp-option=3,192.168.1.1` |
| `dns-server 192.168.1.1` | `dhcp-option=6,192.168.1.1` |
| `lease 0 12 0` | `dhcp-range` 末尾的 `12h` |
| PC 上选“自动获得 IP” | `dhclient eth0` |

### 拓扑说明

三台设备在同一个广播域，交换机只做透明转发，所以 `lab.conf` 里
`r1 / pca / pcb` 直接接到同一个 `netLAN`，**数据面与经过交换机完全等价**。

> 图里的交换机在本实验里不产生任何效果。如果一定要摆一台，
> 用 `kathara/openvswitch`，三个口都留在默认 VLAN（不加 `tag=`）即可；
> 但 macOS/OrbStack 上会触发广播风暴，所以这里不启用。
> 需要“带交换机”的版本请到 Linux 上跑。

### 文件

```
task7/task1/
├── lab.conf
├── r1.startup     路由器：F0/0 地址 + dnsmasq 配置 + 启动服务
├── pca.startup    dhclient 自动获取（带重试）
└── pcb.startup
```

`r1.startup`：

```bash
ip addr add 192.168.1.1/24 dev eth0
ip link set eth0 up

cat > /etc/dnsmasq.conf <<'EOF'
interface=eth0
bind-interfaces

dhcp-range=192.168.1.100,192.168.1.200,255.255.255.0,12h
dhcp-option=3,192.168.1.1
dhcp-option=6,192.168.1.1

dhcp-leasefile=/var/lib/misc/dnsmasq.leases
log-dhcp
EOF

service dnsmasq start
```

`pca.startup` / `pcb.startup`：

```bash
ip link set eth0 up

for i in $(seq 1 30); do
    if dhclient -1 eth0; then break; fi
    sleep 2
done
```

### 运行

```bash
cd ~/计算机网络实验/task7/task1
kathara lstart
```

等 10 秒（DHCP 是并行启动的，要等 DISCOVER 往返）。

---

## 五、实测结果

`pca`：

```
$ kathara exec pca -- ip -4 -o addr show eth0
eth0  inet 192.168.1.182/24 brd 192.168.1.255 scope global dynamic eth0
                                          ^^^^^^^ 自动获取

$ kathara exec pca -- ip route
default via 192.168.1.1 dev eth0        ← option 3 下发的网关

$ kathara exec pca -- cat /etc/resolv.conf
nameserver 192.168.1.1                  ← option 6 下发的 DNS
```

`pcb`：`192.168.1.107/24 dynamic`，网关、DNS 同上。

服务器上的租约表：

```
$ kathara exec r1 -- cat /var/lib/misc/dnsmasq.leases
1794634724 ce:82:29:75:61:49 192.168.1.107 pcb *
1794634724 46:0e:7d:29:0f:91 192.168.1.182 pca *
```

连通性：

```
$ kathara exec pca -- ping -c 3 192.168.1.107
3 packets transmitted, 3 received, 0% packet loss
```

### Linux 版 `ipconfig /all` 对照

| `ipconfig /all` 字段 | Kathara 里看什么 |
|---|---|
| IP Address / Subnet Mask | `ip -4 addr show eth0` 里的 `inet ...` 和 **`dynamic`** |
| Lease Obtained / Expires | 同一行的 `valid_lft` |
| Default Gateway | `ip route` 里的 `default via ...` |
| DNS Servers | `cat /etc/resolv.conf` |
| DHCP Server | `192.168.1.1`（路由器） |

> 作业要求的是 Windows 的 `ipconfig /all`，那一步必须在**真机 + Windows PC** 上截图。
> Kathara 这套用来验证命令和逻辑是否正确，输出可与真机对照。

---

## 六、常见问题

| 现象 | 原因 | 处理 |
|---|---|---|
| PC 拿到 `169.254.x.x` | 没收到 DHCP 应答 | 查交换机端口 VLAN、路由器 `service dhcp`、F0/0 是否 `no shutdown` |
| 有 IP，没有默认网关 | 没配 `default-router` / option 3 | 补上并 `ipconfig /renew` |
| 有 IP，DNS 为空 | 没配 `dns-server` / option 6 | 补上并 `ipconfig /renew` |
| 有 IP，但 ping 不通网关 | 掩码/网段不一致 | 池子的 `network` 必须与 F0/0 同网段 |
| PC 一直拿不到地址 | 交换机口不在同一 VLAN | `show vlan` 确认 |
| `show ip dhcp binding` 为空 | 租约还没建立 | 在 PC 上 `ipconfig /renew` |
| Kathara 里刚启动没地址 | dnsmasq 与客户端并行启动 | 等 10–20 秒；或看 `/var/log/startup.log` |
| `service dnsmasq status` 显示 dead | 镜像里 pid 文件路径不一致（误报） | 用 `ss -ulpn \| grep :67` 确认真的在监听 |

关闭实验：

```bash
cd ~/计算机网络实验/task7/task1 && kathara lclean
```
