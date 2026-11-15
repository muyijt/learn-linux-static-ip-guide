# Rocky Linux 9 修改静态 IP 地址指南

本文说明如何在 VMware NAT（VMnet8）网络下，修改 Rocky Linux 9 虚拟机的静态 IP 地址。

## 一、只修改 Rocky Linux 虚拟机的 IP

如果只是更换虚拟机自己的 IP，**不需要修改 VMware 的 VMnet8 子网和 NAT 网关**。

假设网络参数如下：

| 项目 | 配置 |
|---|---|
| VMware 网络模式 | NAT（VMnet8） |
| 子网 | `192.168.200.0/24` |
| NAT 网关 | `192.168.200.2` |
| 原虚拟机 IP | `192.168.200.10` |
| 新虚拟机 IP | `192.168.200.20` |
| DNS | `223.5.5.5`、`8.8.8.8` |

### 1. 查看网卡和连接名称

在 Rocky Linux 中执行：

```bash
nmcli device status
nmcli connection show
```

确认实际连接名称。以下示例使用 `Wired connection 1`，如果你的名称不同，请替换为实际名称。

### 2. 修改静态 IP

只修改 IP 地址：

```bash
sudo nmcli connection modify "Wired connection 1" \
ipv4.addresses 192.168.200.20/24
```

重新启用连接，使配置生效：

```bash
sudo nmcli connection up "Wired connection 1"
```

### 3. 同时设置网关和 DNS（可选）

```bash
sudo nmcli connection modify "Wired connection 1" \
ipv4.method manual \
ipv4.addresses 192.168.200.20/24 \
ipv4.gateway 192.168.200.2 \
ipv4.dns "223.5.5.5 8.8.8.8" \
ipv4.ignore-auto-dns yes \
connection.autoconnect yes

sudo nmcli connection up "Wired connection 1"
```

## 二、检查配置是否生效

查看 IP 地址：

```bash
ip -4 addr
```

查看默认网关：

```bash
ip route
```

查看 DNS：

```bash
cat /etc/resolv.conf
```

测试网络连通性：

```bash
ping -c 4 192.168.200.2
ping -c 4 223.5.5.5
ping -c 4 www.baidu.com
```

注意：部分网络会屏蔽 ICMP，`ping` 失败不一定代表网络不可用，可以再用 `curl` 测试：

```bash
curl -I https://www.baidu.com
```

## 三、同步更新 Windows 和 SSH 连接

如果 Windows 宿主机需要访问虚拟机：

1. 在 Windows 中按 `Win + R`，输入 `ncpa.cpl`。
2. 找到 **VMware Network Adapter VMnet8**。
3. 确认它仍处于 `192.168.200.0/24` 网段内，通常宿主机地址为 `192.168.200.1`。
4. 在 Windows 命令提示符或 PowerShell 中测试：

```powershell
ping 192.168.200.20
```

如果已启用 SSH，之后使用新 IP 连接：

```powershell
ssh rockyadmin@192.168.200.20
```

## 四、如果要修改整个网段

例如，将网络从 `192.168.200.0/24` 改成 `192.168.100.0/24`，就不能只修改 Rocky Linux 的 IP。需要同步修改：

- VMware Virtual Network Editor 中 VMnet8 的 Subnet IP。
- VMnet8 的 NAT 网关地址。
- DHCP 地址池（如果启用了 DHCP）。
- Windows 宿主机 VMnet8 虚拟网卡地址。
- Rocky Linux 的静态 IP 和网关。

修改整个网段前，先确认新网段没有与宿主机当前网络或其他路由冲突。

## 五、建议

如果只是把虚拟机 IP 从 `192.168.200.10` 改成 `192.168.200.20`，建议只修改 Rocky Linux 的静态 IP，保持以下配置不变：

- VMnet8 子网：`192.168.200.0/24`
- NAT 网关：`192.168.200.2`
- Windows VMnet8：`192.168.200.1`

这样改动最少，也不容易影响虚拟机访问外网或 Windows 宿主机访问虚拟机。
