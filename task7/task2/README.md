# task7/task2 — 交换机 DHCP 服务（图 2）

```
              DCRS552 三层交换机
        ┌────────────────────────────┐
        │  interface vlan 1          │
        │  192.168.1.1/24            │
        │  DHCP 服务器                │
        └───┬────────────────┬───────┘
         端口 2            端口 10
           │                │
        ┌──┴─┐           ┌──┴─┐
        │PCA │           │PCB │   IP：自动获取
        └────┘           └────┘
```

和 task1（图 1）做的是同一件事，只是**服务器从路由器换成了三层交换机**。
两个端口都在默认 VLAN 1，所以是一台交换机的 DHCP，不需要跨网段。

| 项目 | 值 |
|---|---|
| 网段 | 192.168.1.0/24 |
| 网关 | 192.168.1.1（交换机的 **VLAN 1 接口**） |
| 地址池 | 192.168.1.100 ～ 192.168.1.200 |
| DNS | 192.168.1.1 |
| 租期 | 12 小时 |

---

## 一、DCRS552 完整命令

```
enable
configure terminal

! ---------- 1. 给 VLAN 1 配网关地址 ----------
! 三层交换机没有"物理口 IP"的概念，
! 网关是配在 VLAN 接口上的
interface vlan 1
 ip address 192.168.1.1 255.255.255.0
 no shutdown
 exit

! ---------- 2. 打开 DHCP 服务 ----------
! 有的 DCN 版本默认开启，敲了也不报错
service dhcp

! ---------- 3. 排除不参与分配的地址 ----------
ip dhcp excluded-address 192.168.1.1 192.168.1.99

! ---------- 4. 建地址池 ----------
ip dhcp pool LAN
 network 192.168.1.0 255.255.255.0
 default-router 192.168.1.1
 dns-server 192.168.1.1
 lease 0 12 0
 exit

end
write
```

### 端口配置

图里 PCA 接端口 2、PCB 接端口 10，默认就在 VLAN 1，其实不用配。
写完整一点：

```
configure terminal

interface ethernet 0/0/2
 switchport mode access
 switchport access vlan 1
 exit

interface ethernet 0/0/10
 switchport mode access
 switchport access vlan 1
 exit

end
write
```

> 端口编号不同型号写法可能不同（`0/0/2`、`0/1/2`），以 `show interface brief`
> 里实际列出的为准。

### 和 task1（路由器 DHCP）的区别

| | 图 1 路由器 | 图 2 三层交换机 |
|---|---|---|
| DHCP 服务器 | DCR-2621 | DCRS552 |
| 网关配在哪 | `interface fastethernet 0/0` | `interface vlan 1` |
| DHCP 命令 | `ip dhcp pool ...` | **完全相同** |
| 需要 `ip routing` 吗 | 不需要 | **单 VLAN 也不需要** |

一句话：**DHCP 的命令两边一样，区别只在网关接口是物理口还是 VLAN 接口。**

> `ip routing` 只有存在**多个 VLAN / 多个网段**、需要三层设备转发时才必须打开。
> 这张图只有 VLAN 1，所以不配也能通。做“多 VLAN 各一个地址池”时才要加。

### 查看

```
show ip dhcp binding          ! 已经分配出去的租约
show ip dhcp pool             ! 池子状态
show interface vlan 1         ! 网关接口地址
show vlan                     ! 端口属于哪个 VLAN
```

---

## 二、PC 端（Windows）

两块网卡都设成**自动获得 IP、自动获得 DNS**：

```cmd
ipconfig /release
ipconfig /renew
ipconfig /all
```

期望：

```
   DHCP Enabled. . . . . . . . . . . : Yes

   IPv4 Address. . . . . . . . . . . : 192.168.1.100(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Lease Obtained. . . . . . . . . . : 2026年11月14日 10:20:31
   Lease Expires . . . . . . . . . . : 2026年11月14日 22:20:31

   Default Gateway . . . . . . . . . : 192.168.1.1
   DHCP Server . . . . . . . . . . . : 192.168.1.1
   DNS Servers . . . . . . . . . . . : 192.168.1.1
```

**截图关键点**：

1. `DHCP Enabled: Yes` —— 证明是自动获取；
2. `DHCP Server` 是 `192.168.1.1` —— 证明地址来自这台交换机；
3. `Default Gateway` 有值 —— 证明 option 3 下发了；
4. `DNS Servers` 有值 —— 证明 option 6 下发了；
5. 出现 `169.254.x.x` 就是没拿到，见下面问题表。

---

## 三、Kathara 版（本机已跑通）

三层交换机在 Kathara 里用一台 `kathara/base` 同时模拟三件事：

| DCRS552 的角色 | Kathara 里 |
|---|---|
| 二层交换（端口 2、10 都在 VLAN 1） | 三台设备接同一个 collision domain |
| `interface vlan 1` + IP | `ip addr add 192.168.1.1/24 dev eth0` |
| DHCP 服务器 | dnsmasq |
| `ip dhcp pool LAN` | `dhcp-range=...` |
| `default-router 192.168.1.1` | `dhcp-option=3,192.168.1.1` |
| `dns-server 192.168.1.1` | `dhcp-option=6,192.168.1.1` |
| `lease 0 12 0` | 范围末尾的 `12h` |

### 文件

```
task7/task2/
├── lab.conf
├── l3sw.startup   交换机：VLAN1 接口地址 + dnsmasq
├── pca.startup    dhclient
└── pcb.startup
```

`l3sw.startup`：

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

### 运行

```bash
cd ~/计算机网络实验/task7/task2
kathara lstart
```

等 10 秒，然后：

```bash
kathara exec pca -- ip -4 addr show eth0
kathara exec pca -- ip route
kathara exec pca -- cat /etc/resolv.conf
kathara exec l3sw -- cat /var/lib/misc/dnsmasq.leases
```

---

## 四、实测结果

```
$ kathara exec pca -- ip -4 addr show eth0
inet 192.168.1.162/24 ... scope global dynamic eth0
                              ↑ dynamic = 自动获取

$ kathara exec pca -- ip route
default via 192.168.1.1 dev eth0        ← option 3

$ kathara exec pca -- cat /etc/resolv.conf
nameserver 192.168.1.1                  ← option 6

$ kathara exec pcb -- ip -4 addr show eth0
inet 192.168.1.199/24 ... scope global dynamic eth0

$ kathara exec l3sw -- cat /var/lib/misc/dnsmasq.leases
1794635410 82:73:62:7f:84:fd 192.168.1.162 pca *
1794635410 2e:50:0e:9e:39:c9 192.168.1.199 pcb *
```

连通性：

```
$ kathara exec pca -- ping -c 3 192.168.1.199
3 packets transmitted, 3 received, 0% packet loss
```

`ipconfig /all` 对照：`dynamic` = DHCP Enabled Yes，`valid_lft` = 租期剩余，
`default via` = Default Gateway，`nameserver` = DNS Servers。

> 作业要的是 Windows 的 `ipconfig /all`，这一步要在真机 + Windows 上截图。
> Kathara 这套用来验证命令逻辑。

---

## 五、常见问题

| 现象 | 原因 | 处理 |
|---|---|---|
| PC 拿到 `169.254.x.x` | 没收到 DHCP 应答 | 查 `service dhcp`、`interface vlan 1` 是否 up、端口是否在 VLAN 1 |
| 有 IP，没网关 | 池子漏了 `default-router` | 补上，PC 上 `ipconfig /renew` |
| 有 IP，DNS 空 | 池子漏了 `dns-server` | 补上 |
| 有 IP，ping 不通网关 | `interface vlan 1` 没配地址 / 没 `no shutdown` | 检查 |
| `show ip dhcp binding` 为空 | PC 还没请求 | `ipconfig /renew` |
| 池子地址不分配 | `excluded-address` 范围把池子也排除了 | 检查两个范围是否重叠 |
| 两个 VLAN 时第二个网段拿不到地址 | 少了 `ip routing` 或 `interface vlan X` | 见下面的扩展 |

---

## 六、扩展：两个 VLAN、两个地址池（老师可能加问）

如果要求 PCA 在 VLAN 10、PCB 在 VLAN 20，各自一个网段：

```
vlan 10
 exit
vlan 20
 exit

interface vlan 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
 exit
interface vlan 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown
 exit

! 多网段必须开三层转发
ip routing

interface ethernet 0/0/2
 switchport access vlan 10
 exit
interface ethernet 0/0/10
 switchport access vlan 20
 exit

ip dhcp pool VLAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 192.168.10.1
 exit

ip dhcp pool VLAN20
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 192.168.20.1
 exit
```

这时 DHCP 仍然不需要 relay：**DHCP 服务器就在三层交换机自己身上**，
它直接在两个 VLAN 接口上收广播、各自按网段选池子。
只有服务器在别的设备上（比如一台独立服务器）时才需要 `ip helper-address`。

关闭实验：

```bash
cd ~/计算机网络实验/task7/task2 && kathara lclean
```
