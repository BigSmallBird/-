# task5/rip — RIP 动态路由

拓扑与实验五完全一样，只是把两台路由器的**静态路由**换成 **RIP**。

```
   PCA 192.168.1.10/24                     PCB 192.168.3.10/24
        │                                        │
   F0/0 192.168.1.1                      F0/0 192.168.3.1
      ┌─┴─┐        s0/2        s0/2        ┌─┴─┐
      │ r1├──── 192.168.2.1 ──── 192.168.2.2┤ r2│
      └───┘          192.168.2.0/24          └───┘
```

## 一、RIP 是什么

- **距离矢量**协议：只知道自己直连什么、邻居告诉它什么。
- **度量是跳数**：经过几台路由器。最大 15 跳，**16 = 不可达**。
- **每 30 秒**把整张路由表发给邻居（组播 224.0.0.9，RIPv2）。
- **管理距离 120**：比静态路由（1）和直连（0）低，所以静态路由存在时 RIP 学到的不会生效。
- RIPv1 是有类的、不携带掩码；**RIPv2 无类**、支持 VLSM，写实验一律用 v2。

## 二、Cisco / DCN 配置

### RouterA（r1）

```
enable
configure terminal

! 先把实验五的静态路由删掉，否则它会压过 RIP
no ip route 192.168.3.0 255.255.255.0 192.168.2.2

router rip
 version 2
 no auto-summary
 network 192.168.1.0
 network 192.168.2.0
 exit

end
write
```

### RouterB（r2）

```
enable
configure terminal

no ip route 192.168.1.0 255.255.255.0 192.168.2.1

router rip
 version 2
 no auto-summary
 network 192.168.2.0
 network 192.168.3.0
 exit

end
write
```

### 三个要点

1. **只宣告直连网段**
   - r1 直连 192.168.1.0/24 和 192.168.2.0/24 → 就写这两个；
   - 192.168.3.0/24 是远端，**不要写**，写了也没用，RIP 只按接口宣告。
2. **`network` 是"在这个接口上启用 RIP 并宣告这个网段"**，不是"去这个网段的路线"。
3. **`no auto-summary`**：关掉自动汇总。RIPv2 默认会在主类边界汇总，跨不同主类网段时容易出问题。

### 验证

```
show ip route
show ip protocols
show ip rip database
```

期望看到 `R` 开头的路由：

```
R    192.168.3.0/24 [120/1] via 192.168.2.2, Serial0/2
```

`[120/1]` = 管理距离 120 / 度量 1 跳。

调试用：

```
debug ip rip
no debug all
```

## 三、Kathara 里怎么配（task5/rip/）

Kathara 的路由器用 `kathara/frr` 镜像（内置 FRRouting），对应关系如下：

| Cisco | Kathara（FRR） |
|---|---|
| `router rip` | `frr.conf` 里的 `router rip` 段 |
| `version 2` | `version 2` |
| `network 192.168.1.0` | `network 192.168.1.0/24` |
| `no auto-summary` | `no auto-summary` |
| 开机启动 RIP 进程 | 启用 `ripd=yes` 并 `service frr start` |

`r1.startup` 关键部分：

```bash
# 镜像默认 ripd=no，必须打开
sed -i 's/^ripd=no/ripd=yes/' /etc/frr/daemons

cat > /etc/frr/frr.conf <<'EOF'
frr version 8.4
frr defaults traditional
hostname r1
service integrated-vtysh-config
!
router rip
 version 2
 no auto-summary
 network 192.168.1.0/24
 network 192.168.2.0/24
!
EOF

chown frr:frrvty /etc/frr/frr.conf
chmod 640 /etc/frr/frr.conf
service frr start
```

> 注意：**没有任何 `ip route` 静态路由**，这正是和实验五的区别。

运行：

```bash
cd ~/计算机网络实验/task5/rip
kathara lstart
```

查看：

```bash
kathara exec r1 -- vtysh -c "show ip route"
kathara exec r1 -- vtysh -c "show ip rip status"
kathara exec pca -- ping -c 4 192.168.3.10
```

## 四、实测结果

`r1` 学到的（`R` = RIP，`[120/2]` = 管理距离 120 / 度量 2）：

```
R>* 192.168.3.0/24 [120/2] via 192.168.2.2, eth1
```

`r2` 学到的：

```
R>* 192.168.1.0/24 [120/2] via 192.168.2.1, eth1
```

> FRR 的度量和 Cisco 的显示差 1（Cisco 在同一场景显示 `[120/1]`），
> 这是实现计数方式不同，不影响结论。

端到端：

```
$ kathara exec pca -- ping -c 4 192.168.3.10
64 bytes from 192.168.3.10: icmp_seq=3 ttl=62 time=0.208 ms
64 bytes from 192.168.3.10: icmp_seq=4 ttl=62 time=0.163 ms
4 packets transmitted, 4 received, 0% packet loss
```

## 五、常见坑

| 现象 | 原因 |
|---|---|
| 配了 RIP 但 `show ip route` 里没有 `R` | 忘了开 `ripd`（Cisco 上是没进 `router rip` 配置模式） |
| 有 `R` 路由但 ping 不通 | 静态路由没删，或者接口没 up |
| 学不到远端网段 | `network` 写成了远端网段；只写直连网段 |
| 跨网段汇总出问题 | 没写 `no auto-summary` |
| 刚配完不通 | RIP 30 秒发一次更新，等一等；FRR 启动时会立即发一次 |
| 两台路由器版本不一致 | 统一 `version 2` |

## 六、和静态路由的对比（实验五 vs 实验六）

| | 静态路由（task5） | RIP（task5/rip） |
|---|---|---|
| 配置对象 | 每条远端网段手工写一行 | 只宣告直连网段 |
| 拓扑变化 | 要手工改 | 自动重新收敛 |
| 度量 | 无 | 跳数 |
| 管理距离 | 1 | 120 |
| 适合 | 小网络、结构稳定 | 中小网络、有冗余链路 |
