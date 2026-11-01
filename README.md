# home-231

家庭 L2 网段 `192.168.231.0/24` 的现态档案。

主路由 TP-LINK（192.168.231.1）承担 PPPoE 拨号与 IPv6 RA；旁路由 OneCloud 板 `onecloud`（192.168.231.11）由 dnsmasq 接管 DHCP/DNS、sing-box（tailscale endpoint）承担分流与 tailnet 互联，宣告 `192.168.231.0/24` 到 tailnet；腾讯云 VPS `cn-server-8fdc19` 为 tailnet 对端（tailscaled）。

## 内容

- [AGENTS.md](AGENTS.md) — 架构文档：拓扑图、链路关系、设备与软件实测值
- onecloud.json — 旁路由 sing-box 配置
- tailscale.json — 旧 tailscaled 节点配置基线
- config.json — 本机 Windows 客户端 sing-box 配置导出

## 维护约定

只记现态：当前拓扑、配置值与软件列表，不写变更历史。`config.json` 与 `1.json` 含明文 password/uuid，仓库须保持私有。
