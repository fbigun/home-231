# 家庭网络架构（实测，2026-10-31）

## 一、架构图

```mermaid
flowchart TB
    WAN["互联网"]
    ONT["光猫（桥接）"]

    subgraph TAILNET["tailnet hale-city.ts.net"]
        CN["VPS cn-server-8fdc19（tailscaled）<br/>公网 123.206.118.38<br/>tailnet 100.109.171.0"]
    end

    subgraph L2["家庭 L2 广播域 192.168.231.0/24"]
        MAIN["主路由 TP-LINK<br/>192.168.231.1"]
        AP["AC + AP（易展）"]
        ONE["旁路由 onecloud（sing-box）<br/>LAN 192.168.231.11<br/>tailnet 100.93.202.116"]
        WIFI["无线终端（手机等）"]
        WIRED["有线终端（直连主路由）"]
        PC["m4000q（Windows）<br/>192.168.231.148<br/>tailscale 100.115.171.94（备用）"]
    end

    WAN --- ONT
    ONT --- MAIN
    MAIN --- AP
    AP --- WIFI
    MAIN --- WIRED
    MAIN --- ONE
    WIFI ==>|"默认网关 / DNS"| MAIN
    WIRED ==>|"默认网关 / DNS"| MAIN
    PC ==>|"默认网关 / DNS"| MAIN
    ONE ===|"宣告 192.168.231.0/24（已批准）"| TAILNET
    CN --- WAN
    PC -.->|"计划：改指 .11"| ONE
```

### 链路关系

| 链路 | 关系 | 说明 |
|---|---|---|
| 光猫 ↔ 主路由 | PPPoE | 光猫桥接，主路由拨号；主路由已拿到移动 IPv6 前缀 |
| 主路由 ↔ AC+AP ↔ 无线终端 | 纯二层 | AP 不提供网关与 DNS |
| 主路由 ↔ 有线终端 | 纯二层 | 直连主路由 LAN 口 |
| 主路由 ↔ 旁路由 | 同段旁挂 | 旁路由不做默认网关 |
| 终端 → 主路由 | IPv4/IPv6 默认出口 | 网关与 DNS 均为 `192.168.231.1`；IPv6 下一跳 `fe80::828f…` |
| m4000q → 旁路由（计划） | 仅该设备 | 日常路径：dnsmasq 按 `tag:proxy_devices` 改发 `option 3/6 = 192.168.231.11`；备用路径：本机 Tailscale 客户端（平时禁用，应急启用） |
| 旁路由 ↔ tailnet | overlay 成员 | sing-box tailscale endpoint，tailnet 地址 `100.93.202.116`；宣告 `192.168.231.0/24` 已批准，对端入站可达 LAN（实测 sukean → .148:22000） |
| VPS cn-server-8fdc19 ↔ tailnet | overlay 成员 | tailscaled；`100.109.171.0` / `fd7a:115c:a1e0::4031:ab01`（`cn-server-8fdc19.hale-city.ts.net`）；公网 `123.206.118.38` / `2402:4e00:c032:6900:c694:3d42:32a0:1` |

### 终端在册（旁路由 DHCP 租约，实测）

| 主机名 | IPv4 | MAC | 备注 |
|---|---|---|---|
| m4000q | 192.168.231.148 | 84-7B-57-21-B8-B2 | 本机 Windows：Wi-Fi 6 → 网桥 → Hyper-V 外部交换机 |
| m4000q-1 | — | — | tailnet 重复节点（100.68.203.77），长期 offline，待从管理台删除 |
| MIS-AL00 | 192.168.231.163 | CC-B0-A8-DF-55-20 | 华为设备，随机 MAC |
| OPPO-Reno3-Pro-5G | 192.168.231.120 | 6E-5A-59-9C-CD-11 | 随机 MAC |
| midea_ac_1363 | 192.168.231.178 | BC-89-F8-C1-E4-AE | 美的空调 |
| Bouffalolab_BL602-5262ad | 192.168.231.112 | 7C-B9-4C-52-62-AD | IoT |
| AP | 192.168.231.2 | 68:77:24:8d:2d:bb | AC+AP；静态租约 `192.168.231.2` 生效中（活跃租约实测，`.117` 不再使用）；chaddr 68:77:24:8d:2d:bb，稳定 MAC 68-77-24-8D-2D-BB（client-id 与 EUI-64 LLA 可见） |
| oec | 192.168.231.99 | DA-43-B1-F8-F7-78 | 静态租约 |
| woniu | 192.168.231.100 | 00-E2-69-13-C0-64 | 静态租约；knockd WoL 目标 |

## 二、主路由 TP-LINK（192.168.231.1）

| 项目 | 实测值 |
|---|---|
| 角色 | PPPoE 拨号、IPv4 默认网关、IPv6 唯一通告 RA 的设备 |
| LAN IPv4 | 192.168.231.1/24 |
| LAN MAC | 80-8F-1D-25-D7-AE |
| 链路本地 | fe80::828f:1dff:fe25:d7ae |
| 通告前缀 | 2409:8a28:4e5:3d00::/64 [onlink,auto]，valid 86400 / pref 14400 |
| RDNSS | 2409:8a28:4e5:3d00:828f:1dff:fe25:d7ae |
| 出口公网 | IPv4 `39.182.8.9`、IPv6 `2409:8a28:4e5:3d00:…`，均归属浙江杭州移动（`myip.ipip.net` 实测）；未与主路由 WAN 显示值比对，不能判定 IPv4 是否独占 |
| 管理页 | http://tplogin.cn （旁路由 dnsmasq 有 `address=/tplogin.cn/192.168.231.1`） |

设置要点

- WAN：光猫桥接 + 主路由 PPPoE 拨号；IPv6 已拿到移动前缀，状态健康。
- 运营商是中国移动（`2409:8000::/20` 归移动，`2408:8000::/20` 才是联通）。
- **前缀会变**（/64 整段轮换，不是子网重编）：任何配置都不要写死 GUA，匹配键一律用 IPv4 地址或链路本地地址。
- DHCP 服务：关闭，由旁路由接管（旁路由 `dhcp-authoritative`，下发 `option 3/6 = 192.168.231.1`）。
- IPv6：无状态/SLAAC + 主路由通告 RA，当前实际通告的是 2409:8a28:4e5:3d00::/64。
- IPv6 RA 唯一来源是主路由，旁路由不通告 RA。

## 三、旁路由 onecloud（192.168.231.11）

| 项目 | 实测值 |
|---|---|
| 设备 | OneCloud（Amlogic meson8b，armv7l），Armbian 26.8.3 trixie，kernel 6.12.28-current-meson |
| 登录 | `ssh root@192.168.231.11`（`~/.ssh/config` 中别名 `tailscale`，key `~/.ssh/ed25519`） |
| eth0 IPv4 | 192.168.231.11/24 |
| eth0 MAC | f6:4b:57:a2:ed:1c（`/etc/netplan/20-eth-fixed-mac.yaml` 固定，因此 fe80 地址稳定） |
| eth0 IPv6 | GUA 2409:8a28:4e5:3d00:f44b:57ff:fea2:ed1c/64（SLAAC，由 networkd 用户态收 RA，路由 `proto ra metric 1024`）；静态 ULA fd00:231::1/64；LLA fe80::f44b:57ff:fea2:ed1c |
| IPv6 学习 | `/etc/netplan/armbian.yaml`：`dhcp6: false`（不用 DHCPv6-PD），`eth0.accept_ra=0`（交给 systemd-networkd） |
| 转发 | `net.ipv4.ip_forward=1`，`net.ipv6.conf.all.forwarding=1`（`/etc/sysctl.d/99-forwarding.conf`） |
| 防火墙 | 空规则集：无持久化 ip(6)tables、无 nftables 规则集；FORWARD 策略 ACCEPT |
| sing-box tailscale endpoint | active/enabled；NodeID `nAfhqG1tt221CNTRL`（auth key 注册）；`advertise_routes=["192.168.231.0/24"]`、`accept_routes=true` 已批准；tun-in 栈 **gvisor**（mixed/system 栈会吞掉本机出站 TCP，实测后选定；内核接口名 `tun0`，`tun-in` 是配置里的 tag，仅 IPv4 地址 172.19.0.1/30）；state 在 `/var/lib/sing-box/ts` |
| DHCP 池 | 192.168.231.101–200/255.255.255.0/12h，`dhcp-authoritative`，`domain=lan`、`expand-hosts`、`dhcp-fqdn` |
| 静态租约 | woniu 192.168.231.100（00:E2:69:13:C0:64）；oec 192.168.231.99（DA:43:B1:F8:F7:78）；AP 192.168.231.2（68:77:24:8d:2d:bb，按 DHCP chaddr 匹配） |
| DNS | :53，上游 223.5.5.5、119.29.29.29、2409:8028:2000::1111、2409:8028:2000::2222；`listen-address=192.168.231.11,127.0.0.1`；`/etc/resolv.conf` → `nameserver 127.0.0.1`；尾网段域名经 sing-box `taildns` 解析（dnsmasq 转发 → tun-in → hijack-dns，实测 `sukean` → 100.123.38.36） |
| 下发给普通客户端 | `option 3 = 192.168.231.1`、`option 6 = 192.168.231.1`、`option 138 = 192.168.231.1`、`option 60 = TP-LINK`（即默认全部直连主路由） |
| 代理设备标记 | `tag:proxy_devices` → `option:router/dns-server = 192.168.231.11`；当前唯一绑定该 tag 的 `dhcp-host` 行是注释状态，集合为空 |
| IPv6 RA | 关闭：`enable-ra`、`dhcp-range=::,constructor:eth0,ra-only,slaac,64,1d`、`dhcp-option=eth0,option6:dns-server,fd00:231::1` 三行处于注释状态；备份 `/etc/dnsmasq.conf.d-ra-backup/` 待清理 |
| 其他服务 | sshd active；cron、rsyslog、unattended-upgrades、systemd-resolved/timesyncd、armbian-* 一组 enabled（systemd-resolved 无 :53 监听，:53 由 dnsmasq 独占）；knockd active（端口序列 51002,58004,54781 → `wakeonlan 00:E2:69:13:C0:64`）；NetworkManager 未安装 |

设置要点

1. IPv6 不由旁路由通告，RA 与 RDNSS 全部由主路由下发；脚本与论证存档 `ipv6-fix/`。
2. 只由 dnsmasq + sing-box 两个软件承担分流：dnsmasq 负责"哪些设备走旁路由"，sing-box 负责转发；IPv6 保持直连不进隧道（RA 无法按设备区分）。入站形态 **TUN 先行**（`tun-in` auto_route，栈 gvisor），redirect `lan-redir:7892` 留作回退；tun-in 只配了 IPv4 地址，与"IPv6 直连"天然一致。
3. LAN→尾网段由 sing-box 的 tailscale endpoint 承担（`100.64.0.0/10 + fd7a::/48 → tailnet`，NAT 层搞定回程，无需 SNAT），出站实测通；`advertise_routes=192.168.231.0/24` 已批准，tailnet 对端入站可达 LAN（实测 sukean → .148:22000）。

## 四、旁路由已安装软件

| 软件 | 版本 | 安装方式 | 状态 | 位置 / 备注 |
|---|---|---|---|---|
| dnsmasq | 2.91-1+deb13u2 | apt | active / enabled | `/etc/dnsmasq.conf`，DHCP + DNS；RA 相关三行注释关闭；`dhcp-host=68:77:24:8d:2d:bb,192.168.231.2`（AP 静态绑定） |
| sing-box | 1.14.4（armhf deb，dpkg 状态 `hi` 已 hold） | apt/deb | active / enabled | unit `/usr/lib/systemd/system/sing-box.service`（User=sing-box，CAP_NET_ADMIN/RAW/NET_BIND_SERVICE/SYS_PTRACE，`-D /var/lib/sing-box -C /etc/sing-box`）；`/etc/sing-box/config.json`（600 root；auth key 已写入并消费，勿提交 git）；tailscale endpoint（state `/var/lib/sing-box/ts`）+ tun-in（gvisor）+ lan-redir:7892 并存；启动引导依赖直连 UDP 解析器 `boot=223.5.5.5`（`local` 链在旧 MagicDNS 残留时死锁）；升级方式：sagernet 源在板上拉不动（后端 403），本机下载 armhf deb、MD5 对过 apt 元数据后 scp + `dpkg -i` |
| systemd-networkd + netplan | 随 Armbian | — | active / enabled | `/etc/netplan/armbian.yaml`、`/etc/netplan/20-eth-fixed-mac.yaml` |
| knockd | 0.8-2+b4 | apt | active / enabled | WoL 触发（woniu） |
| systemd-resolved / systemd-timesyncd | 随 systemd | apt | enabled | resolved 存在但无 :53 监听（dnsmasq 独占）；timesyncd 对时 pool.ntp.org 实测同步正常（chrony 无增益，不装） |
| armbian-config / armbian-firmware / bsp-cli-onecloud | 26.8.x | Armbian 镜像 | — | 板级工具 |
| tcpdump | 4.99.5 | apt | — | 诊断用 |

卸载后的残留备份在 `/root/migration-backup-20261031/`，待完成清理。

## 五、文档维护约定

- 只写现态：当前拓扑、配置值与软件列表；不写变更记录、带日期的"已完成 / 已修复 / 裁剪记录"叙述。
- 历史只留两种形式：现态事实（"三行处于注释状态"）或待办（"残留备份在 `/root/migration-backup-20261031/`，待清理"）；论证放专文，正文只留指针。
- 已不存在的对象（卸载的软件、关闭的链路）直接从图表删除，不留占位。
- 改一处先全文搜同类（日期、"已"、"曾"、"本次"、"新增"、"改名"），一次处理干净。
- 落盘前给全文预览，等一句同意再写。