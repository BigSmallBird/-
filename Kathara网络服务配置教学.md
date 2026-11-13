# Kathara 网络实验配置教学手册

> 适用环境：macOS（Apple Silicon）+ OrbStack + Kathara 3.8.x  
> 已存在的 conda 环境：`kathara-env`（不要另建环境）  
> 本文命令统一写作 `python -m kathara ...`，在“实验场景目录”中执行。  
> 已验证镜像：`kathara/base`、`kathara/frr`、`kathara/openvswitch`（均为 arm64）。  
> 本文中的路由、OSPF、DHCP、DNS、HTTP、iptables 防火墙/NAT/DNAT 示例均在本机实际跑通过；macOS 特有的限制在第 11 节单独说明。

---

## 目录

1. [先理解 Kathara 的 6 个核心概念](#1-先理解-kathara-的-6-个核心概念)
2. [场景目录与 lab.conf 语法](#2-场景目录与-labconf-语法)
3. [startup 脚本与设备文件夹](#3-startup-脚本与设备文件夹)
4. [基础网络配置：IP、路由、VLAN](#4-基础网络配置ip路由vlan)
5. [路由器配置：Linux 静态路由与 FRR/OSPF](#5-路由器配置linux-静态路由与-frrosopf)
6. [防火墙：iptables 过滤、NAT、DNAT](#6-防火墙iptables-过滤natdnat)
7. [链路追踪与故障排查](#7-链路追踪与故障排查)
8. [DHCP 服务](#8-dhcp-服务)
9. [DNS 服务](#9-dns-服务)
10. [其他基础服务：HTTP、SSH、iperf3、端口映射](#10-其他基础服务httpsdhiperf3端口映射)
11. [综合实验：一个完整的小型企业网](#11-综合实验一个完整的小型企业网)
12. [自定义镜像](#12-自定义镜像)
13. [macOS + OrbStack 专题](#13-macos--orbstack-专题)
14. [命令速查与排错清单](#14-命令速查与排错清单)
15. [附录：镜像内工具清单](#15-附录镜像内工具清单)

---

## 1. 先理解 Kathara 的 6 个核心概念

1. **一个实验 = 一个文件夹**。文件夹里有 `lab.conf`（拓扑）、每台设备的 `<设备名>.startup`（启动脚本），还可以有 `<设备名>/` 目录（预置文件）。
2. **`lab.conf` 只描述拓扑**：哪台设备、用哪个镜像、哪个网卡接到哪条链路。IP 地址、路由、服务配置都写在 startup 或设备目录里。
3. **同一个链路名 = 同一个广播域**。`pca[0]=netA` 和 `sw1[0]=netA` 表示 pca 的 eth0 与 sw1 的 eth0 接在同一条线上。链路名不同就必须靠路由器互通。
4. **startup 是容器里的 root bash 脚本**。容器里没有 systemd，所以要用 `service xxx start`（镜像自带 SysV 脚本时）或直接运行守护进程。
5. **每次 `lstart` 都重新执行 startup**。实验配置的“源文件”在宿主机上，容器只是运行实例；改完配置要 `lclean` 再 `lstart`（或 `lrestart`）。
6. **macOS/OrbStack 有额外限制**：网络插件必须用 `kathara/katharanp`；Open vSwitch 只能演示“不同 VLAN 隔离”，不能做同 VLAN 多端口交换、trunk、router-on-a-stick，否则会出现 hairpin 广播风暴（第 13 节详述）。

---

## 2. 场景目录与 lab.conf 语法

### 2.1 目录结构

```
my-lab/
├── lab.conf                  # 必需：拓扑
├── lab.dep                   # 可选：设备启动依赖
├── shared.startup            # 可选：在每台设备上都会执行
├── shared.shutdown           # 可选：停止时在每台设备上执行
├── pc1.startup               # 每台设备的启动脚本
├── r1.startup
├── r1/                       # 可选：设备文件系统，内容会覆盖容器根目录 /
│   └── etc/frr/frr.conf
├── pc1.shutdown              # 可选：该设备停止时执行
├── shared/                   # 自动创建：挂载到所有设备的 /shared
└── lab.ext                   # 外部网络（仅 Linux，macOS 不可用）
```

### 2.2 lab.conf 的基本格式

每一行都是：

```
设备名[参数]=值
```

- **设备名**：只能是小写字母、数字、下划线，长度 1–30；不能用 `shared`、`_test`（保留名）。大写会直接报 `SyntaxError`。
- **参数是数字**：表示网卡编号。编号必须从 0 开始连续（0、1、2……），Kathara 会按编号排序后映射为 `eth0`、`eth1`……
- **参数不是数字**：表示设备的元属性（meta），例如 `image`、`mem`。
- **值**：网卡行的值是“链路名”（collision domain）。链路名只能由字母/数字/下划线组成；同名链路 = 同一个二层广播域。
- **注意**：`pc1[eth0]=netA` 不是“指定网卡名”，`eth0` 会被当成 meta（而且 Kathara 不认这个 meta）。正确写法是 `pc1[0]=netA`。
- **等号两边不能有空格**；行尾注释必须至少有一个空格再接 `#`。整行以 `#` 开头是注释。

最小示例：

```ini
# 两台 PC 接在同一条链路
pc1[0]=netA
pc2[0]=netA
```

设备名相同、接口编号相同的行不能重复写；接口编号必须连续：

```ini
# 错误：只有 0 和 2，缺 1
r1[0]=netA
r1[2]=netB

# 正确
r1[0]=netA
r1[1]=netB
r1[2]=netC
```

### 2.3 常用 meta 属性

| 属性 | 示例 | 说明 |
|---|---|---|
| `image` | `r1[image]="kathara/frr"` | 使用的 Docker 镜像；不写就用全局默认 `kathara/base` |
| `mem` | `pc1[mem]="256m"` | 内存限制；纯数字视为 MB |
| `cpus` | `pc1[cpus]="0.5"` | CPU 核数，可为小数 |
| `privileged` | `pc1[privileged]="true"` | 特权容器；一般不需要，默认能力已够做网络实验 |
| `port` | `srv[port]="8080:80/tcp"` | 把宿主机 8080 映射到容器 80；只写 `80` 时宿主机默认用 3000 |
| `volume` | `pc1[volume]="/Users/me/share|/share|rw"` | 挂载宿主机目录；格式 `宿主机路径|容器路径|模式`，模式 `ro/rw/rx` |
| `env` | `pc1[env]="FOO=bar"` | 设置环境变量；可写多行 |
| `sysctl` | `r1[sysctl]="net.ipv4.ip_forward=1"` | 只允许 `net.*`；在容器创建时生效 |
| `exec` | `pc1[exec]="ip route add 10.9.0.0/24 via 10.10.0.1"` | startup 执行完后运行的命令；可写多行 |
| `ulimit` | `pc1[ulimit]="nofile=1024:2048"` | 软/硬限制 |
| `shell` | `pc1[shell]="/bin/bash"` | 交互 shell |
| `num_terms` | `pc1[num_terms]="2"` | 打开的终端数量 |
| `ipv6` | `pc1[ipv6]="true"` | 启用 IPv6 |
| `entrypoint` / `args` | 一般不用 | 覆盖镜像入口/参数 |
| `bridged` | 一般不用 | 接到宿主机网桥（Linux 场景）；macOS 不适合 |
| `type` | `sw1[type]="switch"` | **只是注释性标签，没有任何行为** |

几个容易误解的点：

- `type="switch"` **不会**让设备变成交换机。真正决定行为的是 `image`（例如 `kathara/openvswitch`）和 startup 里的命令。
- `sysctl`、`env`、`port`、`volume` 可以写多行，每行增加一个条目；`image`、`mem`、`cpus` 这类标量属性重复写会覆盖。
- `port` 的协议支持 `tcp/udp/sctp`，默认 tcp；`8080:80/udp` 表示宿主机 8080/UDP → 容器 80/UDP。
- `volume` 的宿主机路径建议写绝对路径；相对路径会按执行命令时的当前目录解析，容易出错。

### 2.4 LAB_ 元数据

可以给场景加说明（都是可选）：

```ini
LAB_NAME="VLAN and Routing Lab"
LAB_DESCRIPTION="Two VLANs plus inter-VLAN routing"
LAB_VERSION="1.0"
LAB_AUTHOR="Your Name"
LAB_EMAIL="you@example.com"
LAB_WEB="https://example.com"
```

### 2.5 常用 CLI

```bash
# 在实验目录中启动；--noterminals 不弹终端窗口
python -m kathara lstart --noterminals

# 只启动/排除某些设备
python -m kathara lstart --noterminals pc1 r1
python -m kathara lstart --noterminals --exclude pc2

# 先做语法检查（dry run）
python -m kathara lstart --print

# 停止实验
python -m kathara lclean

# 重启场景（容器会重建，startup 会重新执行）
python -m kathara lrestart --noterminals

# 查看运行中的设备 / 场景信息
python -m kathara list
python -m kathara linfo

# 进入设备 shell；-l 会先打印 startup 日志
python -m kathara connect -l pc1

# 在设备里执行单条命令；--wait 等 startup 执行完再执行
python -m kathara exec --wait pc1 -- ip addr show eth0

# 检查宿主机环境（Docker、插件等）
python -m kathara check
```

如果出现 `MachineAlreadyExistsError`，说明上一次容器没有清干净：

```bash
python -m kathara lclean
python -m kathara lstart --noterminals
```

> 本机使用 conda 环境时，也可以先 `conda activate kathara-env`，然后直接用 `python -m kathara`。不要新建环境。

---

## 3. startup 脚本与设备文件夹

### 3.1 startup 的执行机制

Kathara 启动设备时，实际按下面的顺序执行（这是 Docker 管理器的行为）：

1. 卸载 Docker 自动挂载的 `/etc/resolv.conf` 和 `/etc/hosts`，这样设备目录里的同名文件才能覆盖它们。
2. 如果存在 `<设备名>/` 目录，把它打包解压到容器根目录 `/`。
3. 如果存在 `/etc/frr` 或 `/etc/quagga`，自动把它们 chown 给对应用户并设置权限。
4. 执行 `shared.startup`（如果存在），输出写到 `/var/log/shared.log`。
5. 执行 `<设备名>.startup`（如果存在），输出写到 `/var/log/startup.log`。
6. 执行 `lab.conf` 里该设备的 `exec` 命令，并把 `++ 命令` 追加到 startup.log。
7. `touch /tmp/EOS`，表示启动流程结束。

关键点：

- startup 由 root 执行；Kathara 会在脚本第一行前插入 `set -x`，所以日志里能看到每条命令。
- 容器里**没有 systemd**。能用 `service xxx start` 的前提是镜像里有 `/etc/init.d/xxx`；否则直接运行守护进程。
- startup 里的普通命令按顺序同步执行；要让服务常驻，用 `service` 或 `nohup ... &`。
- 查看日志：
  ```bash
  python -m kathara exec pc1 -- cat /var/log/startup.log
  python -m kathara connect -l pc1
  ```
- 开机脚本末尾如果某条命令失败，后面的命令仍会继续（除非你写 `set -e`）。需要容错时可写 `cmd || true`。

### 3.2 设备文件夹：预置配置文件

`<设备名>/` 目录会被复制到容器根目录。例如：

```
r1/
└── etc/frr/frr.conf

srv/
├── var/www/html/index.html
└── etc/resolv.conf
```

这样做的好处：

- 长配置文件不用塞进 startup 的 heredoc；
- 文件在宿主机上可直接编辑、版本管理；
- `/etc/resolv.conf`、`/etc/hosts` 因为前面被卸载，可以被设备目录中的文件覆盖。

注意文件权限：解压后的文件属主是 root。需要非 root 用户读取的配置，保证 `644` 即可；如果服务以专用用户运行（如 `named`、`frr`），必要时在 startup 里 `chown`。FRR 目录会被 Kathara 自动 chown，见 5.3。

### 3.3 shared 目录与 shared.startup

- 实验目录下的 `shared/` 会自动创建，并以读写方式挂载到每台设备的 `/shared`。适合在设备和宿主机之间传文件。
- `shared.startup` 会在**每台设备**上执行，适合做所有设备的公共配置（例如设置 root 密码、统一 DNS 等）。
- `shared.shutdown` 会在每台设备停止时执行。
- 不想挂载 shared 时，`lstart --no-shared`（或改配置里的 `shared_mount`）。

---

## 4. 基础网络配置：IP、路由、VLAN

### 4.1 配置 IP 和默认网关

`pc1.startup`：

```bash
ip addr add 192.168.1.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.1.1
```

常用检查命令：

```bash
ip -br addr                 # 简洁查看所有接口地址
ip route                    # 路由表
ip route get 10.20.0.10     # 查看去某个目的地实际选哪条路、从哪个源地址出
ip neigh                    # ARP 邻居表
```

删除/替换路由：

```bash
ip route del default
ip route replace default via 10.10.0.1
```

### 4.2 多网段静态路由完整实验

拓扑：

```
pc1 ---- net10 ---- r1 ---- net12 ---- r2 ---- net20 ---- pc2
      10.10.0.0/24      10.12.0.0/24      10.20.0.0/24
```

`lab.conf`：

```ini
pc1[0]=net10
r1[0]=net10
r1[1]=net12
r2[0]=net12
r2[1]=net20
pc2[0]=net20

# 两个路由器都要开启转发
r1[sysctl]="net.ipv4.ip_forward=1"
r2[sysctl]="net.ipv4.ip_forward=1"
```

`pc1.startup`：

```bash
ip addr add 10.10.0.10/24 dev eth0
ip link set eth0 up
ip route add default via 10.10.0.1
```

`r1.startup`：

```bash
ip addr add 10.10.0.1/24 dev eth0
ip addr add 10.12.0.1/24 dev eth1
ip link set eth0 up
ip link set eth1 up

# 去往右侧 10.20.0.0/24 的下一跳是 r2 的 10.12.0.2
ip route add 10.20.0.0/24 via 10.12.0.2
```

`r2.startup`：

```bash
ip addr add 10.12.0.2/24 dev eth0
ip addr add 10.20.0.1/24 dev eth1
ip link set eth0 up
ip link set eth1 up

# 去往左侧 10.10.0.0/24 的下一跳是 r1 的 10.12.0.1
ip route add 10.10.0.0/24 via 10.12.0.1
```

`pc2.startup`：

```bash
ip addr add 10.20.0.10/24 dev eth0
ip link set eth0 up
ip route add default via 10.20.0.1
```

验证：

```bash
python -m kathara exec --wait pc1 -- ping -c 3 10.20.0.10
python -m kathara exec pc1 -- traceroute -n 10.20.0.10
```

预期 traceroute：

```
 1  10.10.0.1
 2  10.12.0.2
 3  10.20.0.10
```

**关于 `ip_forward`**：

- 推荐在 `lab.conf` 用 `r1[sysctl]="net.ipv4.ip_forward=1"`，Kathara 会在创建容器时通过 Docker 的 `--sysctl` 设置，本机实测有效。
- 也可以在 startup 里写 `sysctl -w net.ipv4.ip_forward=1`。在 macOS/OrbStack 上这条命令可能报 `permission denied`，但本机默认 `ip_forward` 已经是 1；先检查：
  ```bash
  python -m kathara exec r1 -- cat /proc/sys/net/ipv4/ip_forward
  ```
  输出 1 就说明转发已开启。

### 4.3 VLAN 与 Open vSwitch：读懂 task2

你当前的 task2 就是“两台 PC 分别接 OVS 的不同 VLAN”：

`lab.conf`：

```ini
sw1[type]="switch"
sw1[image]="kathara/openvswitch"

sw1[0]=netA
sw1[1]=netB
pca[0]=netA
pcb[0]=netB
```

`sw1.startup`：

```bash
service openvswitch-switch start
ovs-vsctl add-br sw1
ovs-vsctl set bridge sw1 stp_enable=false
ovs-vsctl add-port sw1 eth0 tag=100
ovs-vsctl add-port sw1 eth1 tag=200
ip link set sw1 up
```

逐行解释：

- `service openvswitch-switch start`：容器没有 systemd，必须先启动 OVS 守护进程，否则 `ovs-vsctl` 会连不上数据库。
- `ovs-vsctl add-br sw1`：创建网桥 `sw1`（会同时产生一个同名内部接口）。
- `add-port sw1 eth0 tag=100`：把容器的 eth0 加入网桥，作为 VLAN 100 的 **access 口**；不带 tag 的帧进入时打上 100，带 100 的帧发出时去掉标签。
- `add-port sw1 eth1 tag=200`：同理，VLAN 200。
- `ip link set sw1 up`：把网桥内部接口拉起来。

`pca.startup` / `pcb.startup` 都配置在 `192.168.1.0/24`：

```bash
ip addr add 192.168.1.10/24 dev eth0
ip link set eth0 up
```

结果：

```bash
python -m kathara exec pca -- ping -c 3 192.168.1.20
# 100% packet loss —— 这是正确的，两个 access 口在不同 VLAN
```

查看 VLAN 配置和 MAC 表：

```bash
python -m kathara exec sw1 -- ovs-vsctl show
python -m kathara exec sw1 -- ovs-appctl fdb/show sw1
python -m kathara exec sw1 -- ovs-ofctl dump-ports sw1
```

如果把它们改成同一个 `tag`，在 Linux 主机上会互通；但在 **macOS + OrbStack** 下会触发 hairpin 环路和广播风暴，**不要做**。原因见第 13 节。

> 想在 macOS 上演示“同 VLAN 互通”或 trunk/router-on-a-stick，建议在 Linux 虚拟机（或 Linux 主机）里跑 Kathara。macOS 下可以用“每个 VLAN 一根独立网线接到路由器不同网卡”的方式绕开交换机，见第 11 节综合实验。

---

## 5. 路由器配置：Linux 静态路由与 FRR/OSPF

### 5.1 Linux 路由器

任何 `kathara/base` 设备都可以当路由器：

1. 多个接口各配一个网段；
2. `net.ipv4.ip_forward=1`；
3. 写静态路由（或运行路由协议）。

这就是 4.2 节的例子。Linux 路由器排错的核心命令：

```bash
ip route
ip route get 10.20.0.10
cat /proc/sys/net/ipv4/ip_forward
iptables -L FORWARD -n -v     # 如果开了防火墙
```

### 5.2 FRR 镜像

`kathara/frr` 基于 Debian 12，内置 FRRouting 10.5.3、`vtysh`、`/etc/init.d/frr`。注意：

- 默认 `/etc/frr/daemons` 里几乎所有协议都是 `=no`（`bgpd`、`ospfd`、`ripd` 等）。
- 要用哪个协议，先在 startup 里把它改成 `=yes`。
- 配置文件是 `/etc/frr/frr.conf`，属主必须是 `frr:frr`、权限 `640`；否则守护进程可能不读它。
- 启动命令：`service frr start`（本机实测可用）。
- 查看配置：`vtysh -c "show running-config"`。

### 5.3 用设备文件夹放 FRR 配置（推荐）

目录：

```
ospf-lab/
├── lab.conf
├── pc1.startup
├── pc2.startup
├── r1.startup
├── r2.startup
├── r1/etc/frr/frr.conf
└── r2/etc/frr/frr.conf
```

Kathara 会在执行 startup 之前自动执行：

```bash
chown -R frr:frr /etc/frr/
chmod 640 /etc/frr/*
```

所以设备目录里的 `frr.conf` 不需要自己 chown。如果你是在 startup 里用 heredoc 现场写 `frr.conf`，那就必须自己补：

```bash
chown frr:frr /etc/frr/frr.conf
chmod 640 /etc/frr/frr.conf
```

### 5.4 OSPF 完整实验

拓扑（与 4.2 节相同）：

```
pc1 --- net10 --- r1 --- net12 --- r2 --- net20 --- pc2
     10.10.0.0/24      10.12.0.0/24      10.20.0.0/24
```

`lab.conf`：

```ini
pc1[0]=net10
r1[0]=net10
r1[1]=net12
r2[0]=net12
r2[1]=net20
pc2[0]=net20

r1[image]="kathara/frr"
r2[image]="kathara/frr"
r1[sysctl]="net.ipv4.ip_forward=1"
r2[sysctl]="net.ipv4.ip_forward=1"
```

`pc1.startup`：

```bash
ip addr add 10.10.0.10/24 dev eth0
ip link set eth0 up
ip route add default via 10.10.0.1
```

`pc2.startup`：

```bash
ip addr add 10.20.0.10/24 dev eth0
ip link set eth0 up
ip route add default via 10.20.0.1
```

`r1.startup`：

```bash
ip addr add 10.10.0.1/24 dev eth0
ip addr add 10.12.0.1/24 dev eth1
ip link set eth0 up
ip link set eth1 up

# 打开 OSPF 守护进程
sed -i 's/^ospfd=no/ospfd=yes/' /etc/frr/daemons
service frr start
```

`r1/etc/frr/frr.conf`：

```
frr version 10.5
frr defaults traditional
hostname r1
!
router ospf
 network 10.10.0.0/24 area 0
 network 10.12.0.0/24 area 0
!
line vty
!
```

`r2.startup`：

```bash
ip addr add 10.12.0.2/24 dev eth0
ip addr add 10.20.0.1/24 dev eth1
ip link set eth0 up
ip link set eth1 up

sed -i 's/^ospfd=no/ospfd=yes/' /etc/frr/daemons
service frr start
```

`r2/etc/frr/frr.conf`：

```
frr version 10.5
frr defaults traditional
hostname r2
!
router ospf
 network 10.12.0.0/24 area 0
 network 10.20.0.0/24 area 0
!
line vty
!
```

验证（**注意 OSPF 在广播链路上要先选 DR/BDR，等待约 30–40 秒才会到 Full**）：

```bash
python -m kathara exec r1 -- vtysh -c "show ip ospf neighbor"
python -m kathara exec r1 -- vtysh -c "show ip route"
python -m kathara exec r1 -- vtysh -c "show ip ospf database"
python -m kathara exec pc1 -- ping -c 3 10.20.0.10
```

预期：

- `show ip ospf neighbor` 最终显示 `Full/DR` 或 `Full/BDR`；
- `show ip route` 里有 `O>* 10.20.0.0/24 [110/20] via 10.12.0.2` 这样的路由；
- pc1 能 ping 通 pc2。

刚启动 15 秒左右看到 `2-Way/DROther` 是正常的，再等一会儿。

### 5.5 RIP 与 BGP 的最小配置

RIP：

```bash
sed -i 's/^ripd=no/ripd=yes/' /etc/frr/daemons
service frr start
```

`frr.conf`：

```
frr version 10.5
frr defaults traditional
hostname r1
!
router rip
 version 2
 network 10.10.0.0/24
 network 10.12.0.0/24
!
line vty
!
```

BGP（两台路由器直连的最小例子）：

```
frr version 10.5
frr defaults traditional
hostname r1
!
router bgp 65001
 neighbor 10.12.0.2 remote-as 65002
 network 10.10.0.0/24
!
line vty
!
```

r2 对应 `router bgp 65002`、`neighbor 10.12.0.1 remote-as 65001`。启动前同样要 `sed -i 's/^bgpd=no/bgpd=yes/' /etc/frr/daemons`。

### 5.6 FRR 常用命令

```bash
vtysh -c "show running-config"
vtysh -c "show ip route"
vtysh -c "show ip ospf neighbor"
vtysh -c "show ip ospf database"
vtysh -c "show ip bgp summary"
vtysh -c "show logging"
```

交互式配置：

```bash
vtysh
configure terminal
router ospf
 network 10.10.0.0/24 area 0
end
```

**不要依赖 `write memory` 做持久化**：Kathara 每次 `lstart` 都会重建容器，正确做法是把配置放在宿主机的 `frr.conf` 或 startup 里，让它每次启动都重新应用。

---

## 6. 防火墙：iptables 过滤、NAT、DNAT

### 6.1 基本概念

iptables 有四张常用表：

- `filter`：默认表，做放行/拒绝（INPUT、FORWARD、OUTPUT）。
- `nat`：地址转换（PREROUTING、POSTROUTING）。
- `mangle`：改包（TTL、TOS 等），基础实验很少用。
- `raw`：连接跟踪之前处理，基础实验很少用。

规则按“第一条匹配即生效”处理；链的默认策略由 `-P` 设置。

> 本机 `kathara/base` 提供 `iptables`、`iptables-legacy`、`iptables-save/restore`，**没有 `nft` 命令**。用 `iptables` 即可。

### 6.2 查看与清理

```bash
iptables -L -n -v                  # filter 表所有链，带包/字节计数
iptables -L INPUT -n -v --line-numbers
iptables -t nat -L -n -v           # NAT 表

iptables -F                        # 清空 filter 表规则（不清策略）
iptables -t nat -F                 # 清空 nat 表规则
```

### 6.3 主机防火墙模板

以一台服务器为例，默认拒绝入站，只放行已建立连接、ICMP、SSH、HTTP：

```bash
iptables -F
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# 本机回环必须放行
iptables -A INPUT -i lo -j ACCEPT

# 已建立的连接及其相关包（ICMP 差错等）
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 允许 ping
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# 允许 SSH 和 HTTP
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```

说明：

- `ESTABLISHED,RELATED` 很关键：没有它，你放行了请求，回包也会被 DROP。
- `-i lo` 也要放行，否则本机访问本机服务会失败。
- `conntrack` 是连接跟踪匹配模块；老教材里的 `-m state --state ...` 仍然可用，但 `conntrack` 更现代。

### 6.4 路由器上的 FORWARD 过滤

路由器转发流量经过的是 `FORWARD` 链，不是 `INPUT`。下面是一个“只允许内网访问服务器 80 端口，其他转发一律拒绝”的例子：

```bash
iptables -F FORWARD
iptables -P FORWARD DROP

# 回程流量
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 允许 10.10.0.0/24 访问 10.30.0.10 的 80/tcp
iptables -A FORWARD -s 10.10.0.0/24 -d 10.30.0.10 -p tcp --dport 80 -j ACCEPT

# 其他一律拒绝（REJECT 会回 ICMP，便于观察；DROP 则静默丢弃）
iptables -A FORWARD -j REJECT
```

要点：

- 规则顺序就是优先级；把 `REJECT` 放在 `ACCEPT` 后面就永远不会生效。
- `-P FORWARD DROP` 只影响没有匹配任何规则的包；`-A FORWARD -j REJECT` 是显式拒绝并回错误。
- 看计数器判断规则是否命中：
  ```bash
  iptables -L FORWARD -n -v
  ```

### 6.5 SNAT / MASQUERADE

典型场景：内网 `10.10.0.0/24` 通过路由器访问“外网” `203.0.113.0/24`，让外网只看到路由器的地址。

`r1.startup`：

```bash
ip addr add 10.10.0.1/24 dev eth0
ip addr add 203.0.113.1/24 dev eth1
ip link set eth0 up
ip link set eth1 up

# 开启转发（lab.conf 里也可以写 [sysctl]）
sysctl -w net.ipv4.ip_forward=1 2>/dev/null || true

# 把从 eth1 出去的内网源地址改成 eth1 的地址
iptables -t nat -A POSTROUTING -s 10.10.0.0/24 -o eth1 -j MASQUERADE
```

还要保证 FORWARD 放行：

```bash
iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

验证：在“外网”服务器上抓包，看到的源地址是路由器的 `203.0.113.1`，而不是 `10.10.0.10`。

> `MASQUERADE` 适合地址不固定的出口；出口地址固定时也可以用 `SNAT --to-source 203.0.113.1`。NAT 只对连接的第一个包做转换，所以 NAT 规则的计数器通常很小，这是正常的。

### 6.6 DNAT / 端口转发

场景：外网访问路由器的 8080，转发到内网服务器的 80。

```bash
# 进入 eth0 的 TCP 8080，改目的地址为 10.30.0.10:80
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 \
  -j DNAT --to-destination 10.30.0.10:80

# 转发链要放行转换后的目标（注意：这里看到的是 10.30.0.10:80）
iptables -A FORWARD -p tcp -d 10.30.0.10 --dport 80 -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

要点：

- DNAT 在 `PREROUTING` 做，发生在路由选择和 FORWARD 过滤之前。FORWARD 链看到的是**转换后**的目的地址和端口。
- 回程由 conntrack 自动做反向转换，不需要再写一条 SNAT。
- 如果内网服务器没有把默认网关指向路由器，回程会走错，这时需要在路由器上补 MASQUERADE（也就是“DNAT + SNAT”的 hairpin NAT）。

### 6.7 规则持久化

在 Kathara 里，最可靠的做法是把规则写进 startup，或者写成一个文件在 startup 里恢复：

```bash
# 保存当前所有表的规则
iptables-save > /etc/iptables/rules.v4

# 下次启动时恢复（会覆盖现有表）
iptables-restore < /etc/iptables/rules.v4
```

把 `rules.v4` 放在设备目录 `r1/etc/iptables/rules.v4`，startup 里只需：

```bash
iptables-restore < /etc/iptables/rules.v4
```

### 6.8 防火墙排错

```bash
iptables -L -n -v                          # 看计数
iptables -t nat -L -n -v                   # 看 NAT
iptables -S                                # 以规则语法显示
iptables -L FORWARD -n -v --line-numbers   # 带行号，方便删除
iptables -D FORWARD 3                      # 删除 FORWARD 第 3 条
```

- 如果某个方向完全不通，先看 `FORWARD` 默认策略和计数器。
- 如果“请求能出去、回包回不来”，通常是缺少 `ESTABLISHED,RELATED`。
- 如果 NAT 不生效，检查是否在正确的表/链（SNAT 在 POSTROUTING，DNAT 在 PREROUTING），以及是否开启了转发。
- 容器里被防火墙锁住也不怕：`python -m kathara exec r1 -- iptables -F` 仍然可以进去修（exec 不走网络）。

---

## 7. 链路追踪与故障排查

### 7.1 traceroute / tracepath / mtr

```bash
traceroute 10.20.0.10          # 默认 UDP 探测
traceroute -n 10.20.0.10       # 不反解域名，输出更干净
traceroute -I 10.20.0.10       # 用 ICMP Echo
traceroute -T -p 80 10.20.0.10 # 用 TCP SYN（需要 root）

tracepath 10.20.0.10           # 不需要 root，能看 PMTU
mtr -r -c 3 10.20.0.10         # 报告模式
```

预期输出（第 4.2 节拓扑）：

```
traceroute to 10.20.0.10, 30 hops max, 60 byte packets
 1  10.10.0.1
 2  10.12.0.2
 3  10.20.0.10
```

如果中间某跳显示 `* * *`：

- 该跳设备不回 TTL 超时（防火墙 DROP 了 ICMP），但只要后面的跳能出现，路径通常是通的；
- 如果从某一跳开始全是 `*`，检查该路由器的路由表、转发开关和 FORWARD 规则。

### 7.2 tcpdump

```bash
tcpdump -i eth0 -n                       # 不解析域名
tcpdump -i eth0 -n -e                    # 显示 MAC 地址
tcpdump -i eth0 -n 'icmp'
tcpdump -i eth0 -n 'tcp port 80'
tcpdump -i eth0 -n -vv 'udp port 53'
tcpdump -i eth0 -n -w /tmp/cap.pcap      # 存文件，可拿回宿主机用 Wireshark 看
tcpdump -i any -n                        # 所有接口
```

在 OVS 设备上：

```bash
tcpdump -i eth0 -n -e                    # 看某个物理口
tcpdump -i sw1 -n -e                     # 看网桥内部接口（网桥名以你创建的为准）
```

### 7.3 ARP、邻居、端口、DNS

```bash
ip neigh                       # ARP 表
arping -c 3 10.10.0.10         # 发 ARP 请求
ss -lntup                      # 本机监听的 TCP/UDP 端口
ss -anp | grep 80              # 查看连接
nc -zv 10.30.0.10 80           # 测试端口是否可达
dig @10.10.0.1 web.lab +short  # 直接问指定 DNS
```

### 7.4 OVS 排查命令

```bash
ovs-vsctl show                 # 网桥、端口、tag
ovs-vsctl list port            # 端口详情（包括 tag）
ovs-appctl fdb/show sw1        # MAC 地址表（哪个 MAC 在哪个端口/VLAN）
ovs-ofctl dump-ports sw1       # 端口收发包计数（判断是否有风暴）
ovs-ofctl dump-flows sw1       # 流表
```

### 7.5 常见故障对照表

| 现象 | 可能原因 | 检查/处理 |
|---|---|---|
| `MachineAlreadyExistsError` | 上次容器未清理 | `lclean` 后重新 `lstart` |
| lab.conf 报 `SyntaxError` | 大写设备名、等号旁有空格、接口编号不连续 | 对照 2.2 节格式 |
| `NonSequentialMachineInterfaceError` | 缺接口编号 | 改成 0、1、2 连续 |
| ping 提示 `Network is unreachable` | 没有路由/默认网关 | `ip route`，补 `ip route add default ...` |
| ping 100% 丢包，但路由存在 | 未开转发、FORWARD DROP、ARP 不通 | 看 `ip_forward`、`iptables -L FORWARD -v`、`ip neigh` |
| 能 ping 通 IP，不能解析域名 | `/etc/resolv.conf` 或 DNS 服务 | `cat /etc/resolv.conf`，`dig @DNS服务器 名字` |
| DHCP 客户端没有地址 | 服务器未就绪/网段不匹配/没在同一广播域 | 等 15–20 秒；看 dnsmasq 配置和 lease 文件 |
| FRR 邻居不是 Full | ospfd 没开、接口没 IP、network 写错、没等 DR 选举 | `ps` 看 ospfd；`vtysh -c "show ip ospf neighbor"`；等 40 秒 |
| OVS 端口计数飞涨、ping 不通 | macOS 下同 VLAN 多端口 hairpin 环路 | 见第 13 节；不要在同 VLAN 接多个 OVS 端口 |
| `service: command not found` | 自定义镜像没有 SysV 脚本 | 直接运行守护进程，或改用 `kathara/base` 派生镜像 |
| 容器启动后立刻退出 | 镜像入口/命令不兼容 | 用 `kathara/base` 作为基础镜像；检查 `entrypoint`/`args` |

---

## 8. DHCP 服务

### 8.1 用什么软件

`kathara/base` 自带 **dnsmasq**，它同时支持 DHCP 和 DNS，配置简单，非常适合 Kathara 实验。镜像里没有 `dhcpd`（ISC DHCP Server），也没有 `dhcrelay`，所以：

- 同一广播域内的 DHCP：用 dnsmasq。
- 跨网段的 DHCP：在路由器上为每个接口各开一个 dnsmasq 实例/范围（见 8.4），或者自己构建带 DHCP relay 的镜像。

### 8.2 dnsmasq 作为 DHCP 服务器

`dhcp-srv.startup`（假设服务器地址 10.10.0.2/24，网关 10.10.0.1）：

```bash
ip addr add 10.10.0.2/24 dev eth0
ip link set eth0 up

cat > /etc/dnsmasq.conf <<'EOF'
interface=eth0
bind-interfaces

# 地址池：起始、结束、租期
dhcp-range=10.10.0.100,10.10.0.150,12h

# 默认网关（option 3）
dhcp-option=3,10.10.0.1

# DNS 服务器（option 6）
dhcp-option=6,10.10.0.2

# 记录租约，方便查看
dhcp-leasefile=/var/lib/misc/dnsmasq.leases
log-dhcp
EOF

service dnsmasq start
```

客户端 `pc1.startup`：

```bash
ip link set eth0 up
dhclient -v eth0
```

验证：

```bash
# 等 15~20 秒，dnsmasq 和客户端是并行启动的
python -m kathara exec pc1 -- ip -4 addr show eth0
python -m kathara exec pc1 -- ip route
python -m kathara exec pc1 -- cat /etc/resolv.conf
python -m kathara exec dhcp-srv -- cat /var/lib/misc/dnsmasq.leases
```

- `dhclient` 会根据 option 3 自动加默认路由，根据 option 6 自动写 `/etc/resolv.conf`。
- 释放地址：`dhclient -r eth0`。
- 如果第一次 `lstart` 后马上检查没地址，属于正常竞态：dnsmasq 刚启动，客户端还在 DISCOVER，等十几秒再看。

### 8.3 固定租约（按 MAC）

在 dnsmasq 配置里加：

```ini
dhcp-host=aa:bb:cc:dd:ee:01,10.10.0.100
dhcp-host=aa:bb:cc:dd:ee:02,10.10.0.101
```

Kathara 的网卡可以指定 MAC（链路名后加 `/MAC`）：

```ini
pc1[0]=net10/aa:bb:cc:dd:ee:01
pc2[0]=net10/aa:bb:cc:dd:ee:02
```

### 8.4 多网段 DHCP（路由器场景）

dnsmasq 的 `dhcp-option` 是全局的，如果两个网段的网关不同，必须用 tag 区分。下面这段在本机综合实验中实测可用：

```ini
interface=eth0
interface=eth1
bind-interfaces

# 两个地址池，各自打上 tag
dhcp-range=set:vlan10,10.10.0.100,10.10.0.150,255.255.255.0,12h
dhcp-range=set:vlan20,10.20.0.100,10.20.0.150,255.255.255.0,12h

# 按 tag 下发不同网关和 DNS
dhcp-option=tag:vlan10,3,10.10.0.1
dhcp-option=tag:vlan10,6,10.10.0.1
dhcp-option=tag:vlan20,3,10.20.0.1
dhcp-option=tag:vlan20,6,10.20.0.1
```

其中 `set:vlan10` 给地址池打标签，`tag:vlan10` 表示只对落在这个地址池的客户端下发对应选项。

### 8.5 DHCP 常见问题

- **不在同一广播域**：DHCP DISCOVER 是广播，服务器必须和客户端在同一条链路（或中间有 DHCP relay）。dnsmasq 不会跨路由器自动转发。
- **`interface=` 写错**：dnsmasq 只会在列出的接口上提供服务；多网卡要逐个列出。配合 `bind-interfaces` 可以避免绑定到错误地址。
- **地址池和接口网段不匹配**：服务器接口必须有该网段的地址，否则 dnsmasq 不知道往哪回应。
- **服务没起来**：`python -m kathara exec dhcp-srv -- service dnsmasq status`，或直接运行 `dnsmasq -C /etc/dnsmasq.conf --log-dhcp` 看前台日志。
- **看租约**：`/var/lib/misc/dnsmasq.leases`。

---

## 9. DNS 服务

### 9.1 方案选择

- **dnsmasq**：轻量，适合“本地域名 + 转发”，可以顺便做 DHCP。实验中大多数场景够用。
- **BIND/named**：标准权威 DNS，适合练习 zone 文件、SOA/NS/A/CNAME/MX/PTR 等记录。`kathara/base` 自带 `named`、`named-checkconf`、`named-checkzone`。

### 9.2 dnsmasq 做本地 DNS

`dns.startup`：

```bash
ip addr add 10.10.0.2/24 dev eth0
ip link set eth0 up

cat > /etc/dnsmasq.conf <<'EOF'
interface=eth0
bind-interfaces

# 不读取容器的 /etc/resolv.conf，避免把 Docker 的 DNS 带进来
no-resolv

# 只解析本地域 lab，其他不转发（隔离实验）
local=/lab/

# 本地记录：域名 -> 地址（会匹配子域名）
address=/web.lab/10.10.0.10
address=/db.lab/10.10.0.20
address=/mail.lab/10.10.0.10
EOF

service dnsmasq start
```

客户端：

```bash
# 静态指定 DNS
echo "nameserver 10.10.0.2" > /etc/resolv.conf
echo "search lab" >> /etc/resolv.conf
```

测试：

```bash
python -m kathara exec pc1 -- dig web.lab +short
python -m kathara exec pc1 -- dig @10.10.0.2 web.lab
python -m kathara exec pc1 -- nslookup web.lab
```

要点：

- `address=/web.lab/10.10.0.10` 既匹配 `web.lab`，也匹配 `www.web.lab`。
- `local=/lab/` 告诉 dnsmasq：`lab` 域只查本地配置，不转发。
- 需要访问真实外网 DNS 时，用 `server=8.8.8.8` 之类的上游；纯隔离实验不要加，避免超时。
- `/etc/resolv.conf` 在 Kathara 中可以被设备目录覆盖（Kathara 启动时会先卸载 Docker 的挂载）。如果同时用 DHCP，注意 dhclient 可能会重写它。

### 9.3 BIND/named 权威 DNS

目录结构：

```
dns-lab/
├── lab.conf
├── dns.startup
├── dns/etc/bind/named.conf.local
├── dns/etc/bind/db.lab.test
└── pc1.startup
```

`lab.conf`：

```ini
dns[0]=net10
pc1[0]=net10
```

`dns.startup`：

```bash
ip addr add 10.10.0.2/24 dev eth0
ip link set eth0 up
service named start
```

`dns/etc/bind/named.conf.local`：

```
zone "lab.test" {
    type master;
    file "/etc/bind/db.lab.test";
};
```

`dns/etc/bind/db.lab.test`：

```
$TTL 300
@   IN SOA ns.lab.test. admin.lab.test. (
        2024010101 ; serial
        3600       ; refresh
        600        ; retry
        86400      ; expire
        300 )      ; negative TTL

@       IN NS  ns.lab.test.
ns      IN A   10.10.0.2
www     IN A   10.10.0.10
db      IN A   10.10.0.20
mail    IN A   10.10.0.10
ftp     IN CNAME www.lab.test.
@       IN MX  10 mail.lab.test.
```

客户端 `pc1.startup`：

```bash
ip addr add 10.10.0.10/24 dev eth0
ip link set eth0 up
echo "nameserver 10.10.0.2" > /etc/resolv.conf
```

检查配置并启动：

```bash
named-checkconf
named-checkzone lab.test /etc/bind/db.lab.test
service named start
```

测试：

```bash
python -m kathara exec pc1 -- dig @10.10.0.2 www.lab.test +short
python -m kathara exec pc1 -- dig @10.10.0.2 ftp.lab.test +short
python -m kathara exec pc1 -- dig @10.10.0.2 lab.test MX +short
```

说明：

- SOA 的 serial 每次改 zone 文件要递增，否则从服务器不会更新（单机实验影响不大，但养成习惯）。
- BIND 以 `bind` 用户运行，zone 文件要可读（644 即可）。
- 纯隔离实验可以忽略 DNSSEC 验证；如果看到 `dnssec-validation` 相关日志但不影响解析，可以不管。想彻底安静，可在 `named.conf.options` 里加 `dnssec-validation no;`。
- 反向解析（PTR）需要额外的 `in-addr.arpa` zone；基础实验可以先掌握正向解析。

### 9.4 DNS 排错

```bash
ss -lunp | grep :53              # 服务器是否在监听 53
dig @10.10.0.2 web.lab           # 绕开 /etc/resolv.conf，直接问服务器
dig web.lab                      # 使用 /etc/resolv.conf
cat /etc/resolv.conf
```

- `connection refused`：服务器没启动，或没监听该地址。
- `communications error ... timed out`：路由不通，或防火墙拦了 UDP 53。
- `NXDOMAIN`：服务器正常，但域名/zone 里没有记录。
- 客户端查询成功、服务器本机查询失败：检查 `interface=`/`bind-interfaces` 和监听地址。

---

## 10. 其他基础服务：HTTP、SSH、iperf3、端口映射

### 10.1 HTTP 服务器

**Apache（推荐，镜像自带）：**

```bash
mkdir -p /var/www/html
cat > /var/www/html/index.html <<'EOF'
<h1>Kathara Demo Server</h1>
EOF
service apache2 start
```

更推荐把网页放在设备目录里：

```
srv/var/www/html/index.html
```

Kathara 启动时会把 `/var/www/*` 权限放宽，Apache 可以直接读取。

测试：

```bash
python -m kathara exec pc1 -- curl -s http://10.30.0.10/
python -m kathara exec pc1 -- curl -v http://10.30.0.10/
```

**Python 临时 HTTP 服务器：**

```bash
cd /var/www/html
nohup python3 -m http.server 80 >/var/log/http.log 2>&1 &
```

适合临时实验；正式一点用 Apache。

### 10.2 SSH

`kathara/base` 自带 `sshd` 和 `ssh` 客户端。启动：

```bash
service ssh start
```

默认 Debian 配置通常禁止 root 密码登录。需要密码登录时，在 startup 里自行设置（仅实验环境）：

```bash
echo 'root:lab' | chpasswd
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
service ssh start
```

然后从另一台设备：

```bash
ssh root@10.10.0.10
```

生产环境当然不要这样配置；实验里只是为了方便演示。

### 10.3 iperf3 性能测试

镜像自带 `iperf3` 和 init 脚本。服务端：

```bash
service iperf3 start
# 或前台调试：iperf3 -s
```

客户端：

```bash
iperf3 -c 10.10.0.10 -t 5
iperf3 -c 10.10.0.10 -t 5 -R      # 反向
iperf3 -c 10.10.0.10 -t 5 -u -b 10M  # UDP
```

### 10.4 把服务暴露到宿主机

在 `lab.conf` 给设备加 `port`：

```ini
srv[port]="8080:80/tcp"
```

启动后在 macOS 宿主机：

```bash
curl http://localhost:8080/
```

只写容器端口时，宿主机默认用 3000：

```ini
srv[port]="80"
# 宿主机访问 http://localhost:3000/
```

### 10.5 宿主机与设备共享文件

- `shared/` ↔ 所有设备的 `/shared`（读写）。把文件放进去，所有设备都能看到。
- `volume` 可以挂载任意宿主机目录：

```ini
pc1[volume]="/Users/bigsmallbird/计算机网络实验/share|/data|rw"
```

---

## 11. 综合实验：一个完整的小型企业网

这个实验把前面所有内容串起来，已在本机完整跑通。拓扑：

```
pc1 ---- net10 ---- r1 ---- net12 ---- r2 ---- net30 ---- srv
      10.10.0.0/24      10.12.0.0/24      10.30.0.0/24
pc2 ---- net20 ----/
      10.20.0.0/24
```

角色：

- `pc1`、`pc2`：DHCP 客户端；
- `r1`：两个网段的网关 + DHCP + DNS（dnsmasq）；
- `r2`：路由器 + iptables 防火墙 + DNAT；
- `srv`：Apache HTTP 服务器（80 端口），另开一个 8080 用于演示防火墙拦截。

`lab.conf`：

```ini
pc1[0]=net10
pc2[0]=net20
r1[0]=net10
r1[1]=net20
r1[2]=net12
r2[0]=net12
r2[1]=net30
srv[0]=net30

r1[sysctl]="net.ipv4.ip_forward=1"
r2[sysctl]="net.ipv4.ip_forward=1"
```

`pc1.startup`：

```bash
ip link set eth0 up
dhclient -v eth0
```

`pc2.startup`：

```bash
ip link set eth0 up
dhclient -v eth0
```

`r1.startup`：

```bash
ip addr add 10.10.0.1/24 dev eth0
ip addr add 10.20.0.1/24 dev eth1
ip addr add 10.12.0.1/24 dev eth2
ip link set eth0 up
ip link set eth1 up
ip link set eth2 up

# 去服务器网段的静态路由
ip route add 10.30.0.0/24 via 10.12.0.2

# DHCP + DNS
cat > /etc/dnsmasq.conf <<'EOF'
interface=eth0
interface=eth1
bind-interfaces

dhcp-range=set:vlan10,10.10.0.100,10.10.0.150,255.255.255.0,12h
dhcp-range=set:vlan20,10.20.0.100,10.20.0.150,255.255.255.0,12h

dhcp-option=tag:vlan10,3,10.10.0.1
dhcp-option=tag:vlan10,6,10.10.0.1
dhcp-option=tag:vlan20,3,10.20.0.1
dhcp-option=tag:vlan20,6,10.20.0.1

local=/lab/
address=/web.lab/10.30.0.10
EOF

service dnsmasq start
```

`r2.startup`：

```bash
ip addr add 10.12.0.2/24 dev eth0
ip addr add 10.30.0.1/24 dev eth1
ip link set eth0 up
ip link set eth1 up

ip route add 10.10.0.0/24 via 10.12.0.1
ip route add 10.20.0.0/24 via 10.12.0.1

# 防火墙
iptables -F FORWARD
iptables -P FORWARD DROP
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 先拒绝“直接访问服务器 8080”，演示第一条匹配
iptables -A FORWARD -s 10.10.0.0/24 -d 10.30.0.10 -p tcp --dport 8080 -j REJECT
iptables -A FORWARD -s 10.20.0.0/24 -d 10.30.0.10 -p tcp --dport 8080 -j REJECT

# 其他访问服务器放行（80、ICMP、traceroute 的 UDP 探测等）
iptables -A FORWARD -s 10.10.0.0/24 -d 10.30.0.10 -j ACCEPT
iptables -A FORWARD -s 10.20.0.0/24 -d 10.30.0.10 -j ACCEPT

# DNAT：访问 r2 的 8080，转发到 srv 的 80
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 \
  -j DNAT --to-destination 10.30.0.10:80
```

`srv.startup`：

```bash
ip addr add 10.30.0.10/24 dev eth0
ip link set eth0 up
ip route add default via 10.30.0.1

mkdir -p /var/www/html
cat > /var/www/html/index.html <<'EOF'
<h1>Kathara Demo Server</h1>
EOF

service apache2 start

# 额外开一个 8080，用来验证防火墙
cd /var/www/html && nohup python3 -m http.server 8080 >/var/log/http8080.log 2>&1 &
```

启动并验证（**DHCP 需要等 15–20 秒**）：

```bash
python -m kathara lstart --noterminals
sleep 25

# 1. DHCP 地址、默认路由、DNS
python -m kathara exec pc1 -- bash -c 'ip -4 addr show eth0; ip route; cat /etc/resolv.conf'

# 2. DNS
python -m kathara exec pc1 -- dig web.lab +short
# 期望：10.30.0.10

# 3. HTTP
python -m kathara exec pc1 -- curl -s http://web.lab/
# 期望：<h1>Kathara Demo Server</h1>

# 4. 防火墙：直接访问 srv:8080 被拒绝
python -m kathara exec pc1 -- bash -c 'curl -s -m 5 http://web.lab:8080/; echo rc=$?'
# 期望：rc=7（连接失败）

# 5. DNAT：访问 r2 的 8080 实际访问 srv 的 80
python -m kathara exec pc1 -- curl -s http://10.12.0.2:8080/
# 期望：<h1>Kathara Demo Server</h1>

# 6. 路径追踪
python -m kathara exec pc1 -- traceroute -n -w 2 -q 1 web.lab
# 期望：10.10.0.1 -> 10.12.0.2 -> 10.30.0.10

# 7. 看防火墙计数和 NAT 规则
python -m kathara exec r2 -- iptables -L FORWARD -n -v
python -m kathara exec r2 -- iptables -t nat -L PREROUTING -n -v
```

这个实验覆盖了：静态路由、ip_forward、dnsmasq DHCP（多网段 + tag）、dnsmasq DNS、Apache HTTP、iptables FORWARD 过滤、DNAT、traceroute。

可以继续加的练习：

- 在 r2 上把 `REJECT` 改成 `DROP`，观察 curl 的行为差异；
- 给 srv 加第二条默认路由/第二个网卡，观察回程路径；
- 把 r1/r2 换成 `kathara/frr`，用 OSPF 替换静态路由；
- 给 pc1/pc2 配置固定 MAC，在 dnsmasq 里做固定租约。

---

## 12. 自定义镜像

### 12.1 什么时候需要

- 镜像里缺少某个工具（例如 `nmap`、`tcpdump` 的新版本、DHCP relay）；
- 想把一份软件预装进镜像，避免每次 startup 都 `apt-get install`（容器每次重建，装了就丢）。

### 12.2 Dockerfile 示例

```dockerfile
FROM kathara/base:latest

RUN apt-get update && apt-get install -y --no-install-recommends \
        nmap \
    && rm -rf /var/lib/apt/lists/*
```

构建（Apple Silicon 默认构建 arm64）：

```bash
docker build -t mylab/base:1.0 .
```

在 `lab.conf` 中使用：

```ini
pc1[image]="mylab/base:1.0"
```

### 12.3 自定义镜像的要求

- 基于 Linux，建议直接 `FROM kathara/base`，省去 shell、iproute2、权限等问题；
- 有 `/bin/bash`（startup 用 bash 执行）；
- 不需要 systemd；服务用 SysV 脚本或直接启动；
- 不要覆盖 Kathara 需要的启动行为（除非你明确知道 `entrypoint`/`args` 在做什么）；
- 镜像架构要和 Docker 平台一致（Apple Silicon 用 arm64）。

---

## 13. macOS + OrbStack 专题

### 13.1 网络插件必须用 bridge 插件

Kathara 默认可能使用 VDE 插件（`kathara/katharanp_vde`）。在 OrbStack 下该插件会失败（`NetworkDriver.CreateNetwork: exit status 255`）。修改：

`~/.config/kathara.conf`：

```json
{
  "network_plugin": "kathara/katharanp"
}
```

改完可以用 `python -m kathara check` 或直接启动场景验证。本文所有实验都是在这个插件下跑通的。

### 13.2 镜像名与架构

- 交换机镜像用 `kathara/openvswitch`，**不是** `kathara/ovs`（后者仓库不存在）。
- Apple Silicon 上拉取 arm64 镜像；`kathara/base`、`kathara/frr`、`kathara/openvswitch` 都有 arm64 版本。

### 13.3 OVS 的 hairpin 环路（重要）

在 OrbStack 下，Kathara 的 bridge 插件创建 `kt-*` Linux 网桥，其 veth 端口的 `hairpin_mode=1`。当 OVS 把广播帧从**同一 VLAN 的另一个端口**发出去时，Linux 网桥会把帧反射回 OVS，形成环路：

- 现象：`ovs-ofctl dump-ports` 的收发包计数在几秒内涨到几千万；
- 结果：ping 100% 丢包，CPU 被占满；
- 触发条件：同一 VLAN 在 OVS 上跨多个端口，包括：
  - 两个 access 口都 `tag=100`；
  - 一个 access 口 + 一个 trunk 口（router-on-a-stick）；
  - 任何需要 OVS 在同一 VLAN 内泛洪的拓扑。

本机复现数据：把两个端口都设成同一 VLAN 后，约十几秒内端口计数达到约 2400 万，ping 完全不通。**因此：**

- macOS 下 OVS 实验只做“不同 VLAN 之间的隔离”（就像 task2 的 100/200）；
- 不要做同 VLAN 多端口互通、trunk、STP、router-on-a-stick；
- 需要这些功能时，在 Linux 虚拟机或 Linux 主机上运行 Kathara；
- 在 macOS 上要做“多网段 + 路由”，用“每个网段一根独立链路接到路由器不同网卡”的方式（第 11 节），不要用 OVS trunk。

> 不建议通过修改宿主机 `kt-*` 网桥的 `hairpin_mode` 来绕过：这需要宿主机 root 权限，而且不是 Kathara 实验的一部分，换个环境就不成立。实验文档里应明确说明平台限制。

### 13.4 lab.ext 在 macOS 不可用

`lab.ext` 用于把场景接到宿主机外部网络（VLAN 接口、外部网桥等），Kathara 源码明确限制为 Linux：

```
lab.ext is only available on Linux systems.
```

macOS 下不要用 `lab.ext`。需要从宿主机访问实验服务时，用 `port` 映射（第 10.4 节）；需要真实外部网络时，在 Linux 虚拟机里跑。

### 13.5 sysctl 的差异

在 OrbStack 容器里执行：

```bash
sysctl -w net.ipv4.ip_forward=1
# 可能报 permission denied
```

但容器创建时通过 Docker `--sysctl` 设置是可以的，所以：

```ini
r1[sysctl]="net.ipv4.ip_forward=1"
```

是推荐写法。先检查当前值：

```bash
python -m kathara exec r1 -- cat /proc/sys/net/ipv4/ip_forward
```

本机默认就是 1。

### 13.6 终端与运行方式

- `lstart` 默认会为每台设备打开 Terminal 窗口；自动化/批量实验用 `--noterminals`。
- 需要看 startup 日志时用 `python -m kathara connect -l 设备名`，或直接 `exec ... cat /var/log/startup.log`。
- `lclean` 用 SIGKILL 停止容器（退出码 137 是正常的），之后最好确认 `docker ps` 没有残留。

---

## 14. 命令速查与排错清单

### 14.1 启停与检查

```bash
python -m kathara lstart --noterminals
python -m kathara lstart --print
python -m kathara lclean
python -m kathara lrestart --noterminals
python -m kathara list
python -m kathara linfo
python -m kathara check
```

### 14.2 设备内高频命令

```bash
ip -br addr
ip route
ip route get 10.20.0.10
ip neigh
ss -lntup
ping -c 3 10.20.0.10
traceroute -n 10.20.0.10
tcpdump -i eth0 -n -e
dig @10.10.0.1 web.lab +short
curl -v http://10.30.0.10/
iptables -L -n -v
iptables -t nat -L -n -v
```

### 14.3 服务启动方式

```bash
service openvswitch-switch start
service dnsmasq start
service named start
service apache2 start
service ssh start
service iperf3 start
service frr start
```

### 14.4 排错顺序（建议）

1. 链路层：`ip -br addr`、`ip neigh`、OVS `fdb/show`；
2. 网络层：`ip route`、`ip route get`、`ping`；
3. 转发：`cat /proc/sys/net/ipv4/ip_forward`、`iptables -L FORWARD -v`；
4. 传输层：`ss`、`nc -zv`；
5. 应用层：`dig`、`curl`、服务日志；
6. 抓包：`tcpdump`；
7. startup 日志：`/var/log/startup.log`、`/var/log/shared.log`。

---

## 15. 附录：镜像内工具清单

### 15.1 `kathara/base`（Debian 12）

已验证存在：

- 网络：`ip`、`ss`、`arping`、`bridge`、`nc`、`ip6tables`
- 抓包/追踪：`tcpdump`、`traceroute`、`tracepath`、`mtr`
- 防火墙：`iptables`、`iptables-legacy`、`iptables-save`、`iptables-restore`
- DNS/DHCP：`dnsmasq`、`named`、`named-checkconf`、`named-checkzone`、`dig`、`nslookup`、`host`、`dhclient`
- 服务：`apache2`、`ssh`/`sshd`、`iperf3`、`rsync`、`radvd`
- 其他：`python3`、`curl`、`wget`

不存在/不要假设存在：

- `nft`（用 `iptables`）
- `dhcpd`（ISC DHCP Server，用 `dnsmasq`）
- `dhcrelay`（跨网段 DHCP 需要自己构建镜像或换方案）
- `socat`

### 15.2 `kathara/frr`

- FRRouting 10.5.3
- `/etc/frr/daemons`：默认协议全 `=no`
- `/etc/frr/frr.conf`：默认没有，需要自己提供
- `vtysh`、`/etc/init.d/frr`、`service frr start`

### 15.3 `kathara/openvswitch`

- `service openvswitch-switch start`
- `ovs-vsctl`、`ovs-appctl`、`ovs-ofctl`

---

**最后提醒**：Kathara 实验的“源文件”永远在宿主机实验目录里。容器里的修改会在 `lclean` 后消失；把最终配置写回 `lab.conf`、`*.startup` 或 `<设备名>/` 目录，才算真正保存。
