# Rocky Linux 9 + VMware Workstation Pro：NAT 静态 IP 配置指南

## 一、环境与目标

-   **宿主机**：Windows 10
-   **虚拟机软件**：VMware Workstation Pro（具体版本未指定）
-   **虚拟机系统**：Rocky Linux 9
-   **网络模式**：NAT（VMnet8）

配置目标：

1.  Rocky Linux 9 使用固定静态 IP。
2.  Rocky Linux 9 可以访问互联网。
3.  Windows 10 宿主机可以访问 Rocky Linux。
4.  Rocky Linux 重启后 IP 地址保持不变。

## 二、网络规划

本指南使用以下示例网段：

  配置项                    IP 地址
  ------------------------- --------------------
  VMware VMnet8 子网        `192.168.200.0/24`
  Windows VMnet8 虚拟网卡   `192.168.200.1`
  VMware NAT 网关           `192.168.200.2`
  Rocky Linux 9             `192.168.200.10`
  子网掩码                  `255.255.255.0`
  首选 DNS                  `223.5.5.5`
  备用 DNS                  `8.8.8.8`

**重要：** 配置前先确认 Windows 物理局域网没有使用
`192.168.200.0/24`。如果已经使用该网段，应改用其他不冲突的私有网段，并同步修改下面所有相关
IP。

------------------------------------------------------------------------

# 第一部分：Windows 10 配置 VMware 网络

## 1. 打开虚拟网络编辑器

1.  打开 VMware Workstation Pro。
2.  点击菜单 **Edit → Virtual Network Editor**。
3.  点击右下角 **Change Settings**，以管理员权限修改网络设置。

## 2. 配置 VMnet8

在网络列表中选择 `VMnet8`。如果不存在，可通过 **Add Network** 创建。

设置如下：

  配置项        设置
  ------------- -----------------
  VMnet         `VMnet8`
  Type          `NAT`
  Subnet IP     `192.168.200.0`
  Subnet mask   `255.255.255.0`

确认勾选：

-   `Connect a host virtual adapter to this network`
-   `Use local DHCP service to distribute IP address`（可以勾选）

点击 **Apply**。

## 3. 配置 NAT 网关

选中 VMnet8，点击 **NAT Settings**，将网关设置为：

``` text
Gateway IP: 192.168.200.2
```

点击确定并应用。

## 4. 配置 DHCP 地址池

点击 **DHCP Settings**，建议设置：

``` text
Start IP Address: 192.168.200.128
End IP Address:   192.168.200.254
```

这样 Rocky Linux 使用的 `192.168.200.10` 不会落入 DHCP 动态分配范围。

点击确定。

------------------------------------------------------------------------

# 第二部分：配置 Rocky Linux 9 虚拟机

## 5. 设置虚拟机网络适配器

在 VMware 中选中 Rocky Linux 虚拟机，点击：

**VM → Settings → Network Adapter**

选择 **NAT**，并确认勾选：

-   `Connected`
-   `Connect at power on`

点击 OK，启动 Rocky Linux 9。

## 6. 查看 Rocky Linux 网卡名称和连接名称

进入 Rocky Linux 终端，执行：

``` bash
ip addr
```

例如输出中有：

``` text
2: ens160:
    inet 192.168.200.128/24
```

这里的 `ens160` 是网卡名称。实际名称也可能是 `ens192`、`ens33`
或其他名称，请以实际输出为准。

查看网络设备：

``` bash
nmcli device status
```

查看网络连接：

``` bash
nmcli connection show
```

例如：

``` text
NAME                UUID                                  TYPE      DEVICE
Wired connection 1  xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  ethernet  ens160
```

记下实际连接名称。以下命令以 `Wired connection 1` 为例。

## 7. 配置静态 IP

确保 NetworkManager 正常运行：

``` bash
sudo systemctl enable --now NetworkManager
```

执行以下命令配置静态地址：

``` bash
sudo nmcli connection modify "Wired connection 1" \
ipv4.method manual \
ipv4.addresses 192.168.200.10/24 \
ipv4.gateway 192.168.200.2 \
ipv4.dns "223.5.5.5 8.8.8.8" \
ipv4.ignore-auto-dns yes \
connection.autoconnect yes
```

参数说明：

  参数                           作用
  ------------------------------ --------------------------
  `ipv4.method manual`           使用静态 IPv4 地址
  `ipv4.addresses`               配置固定 IP 和前缀长度
  `ipv4.gateway`                 配置默认网关
  `ipv4.dns`                     配置 DNS 服务器
  `ipv4.ignore-auto-dns yes`     忽略 DHCP 自动下发的 DNS
  `connection.autoconnect yes`   开机自动连接

**注意：** 如果连接名称不是 `Wired connection 1`，请替换为
`nmcli connection show` 中显示的实际名称。

重新激活连接：

``` bash
sudo nmcli connection up "Wired connection 1"
```

如果通过虚拟机控制台操作，执行后网络短暂断开属于正常现象。

## 8. 检查静态 IP 配置

查看 IP：

``` bash
ip -4 addr
```

应看到类似：

``` text
inet 192.168.200.10/24
```

查看路由：

``` bash
ip route
```

应有类似默认路由：

``` text
default via 192.168.200.2 dev ens160
```

查看 DNS：

``` bash
cat /etc/resolv.conf
```

应能看到配置的 DNS 地址，例如：

``` text
nameserver 223.5.5.5
nameserver 8.8.8.8
```

## 9. 测试 Rocky Linux 外网访问

依次测试：

**测试 NAT 网关：**

``` bash
ping -c 4 192.168.200.2
```

**测试公网 IP：**

``` bash
ping -c 4 223.5.5.5
```

**测试 DNS 解析：**

``` bash
ping -c 4 www.baidu.com
```

**测试 HTTPS 访问：**

``` bash
curl -I https://www.baidu.com
```

如果 `curl` 返回 HTTP 响应，说明 HTTPS 访问基本正常。

注意：有些网络会屏蔽 ICMP，因此 ping 不通不一定代表互联网不可用，应结合
`curl` 判断。

------------------------------------------------------------------------

# 第三部分：Windows 10 访问 Rocky Linux

## 10. 检查 Windows VMnet8 虚拟网卡

在 Windows 10 中按 `Win + R`，输入：

``` text
ncpa.cpl
```

回车打开网络连接窗口。

找到：

`VMware Network Adapter VMnet8`

右键选择 **状态 → 详细信息**，确认 IPv4 地址为：

``` text
192.168.200.1
```

也可以在 Windows CMD 中执行：

``` cmd
ipconfig
```

找到 VMware Network Adapter VMnet8 对应的 IPv4 地址。

如果没有 VMnet8 虚拟网卡，回到 VMware Virtual Network Editor，选中
VMnet8 并勾选：

`Connect a host virtual adapter to this network`

然后应用设置。

## 11. 测试 Windows 到 Rocky Linux 的网络连通性

在 Windows CMD 或 PowerShell 中执行：

``` cmd
ping 192.168.200.10
```

如果收到响应，说明宿主机与虚拟机网络互通。

如果 ping 不通，可能是 Rocky Linux 防火墙或其他安全策略限制了
ICMP；可以继续通过 SSH 测试。

## 12. 安装并启动 SSH 服务

在 Rocky Linux 9 中执行：

``` bash
sudo dnf install -y openssh-server
```

启动并设置开机自启：

``` bash
sudo systemctl enable --now sshd
```

检查服务状态：

``` bash
sudo systemctl status sshd
```

确认状态为：

``` text
Active: active (running)
```

在 firewalld 中开放 SSH 服务：

``` bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

检查 SSH 端口监听：

``` bash
sudo ss -lntp | grep :22
```

## 13. 从 Windows 远程连接 Rocky Linux

在 Windows 10 PowerShell 中执行：

``` powershell
ssh 用户名@192.168.200.10
```

例如：

``` powershell
ssh rockyadmin@192.168.200.10
```

首次连接时，SSH 可能提示确认主机密钥。确认目标主机无误后输入
`yes`，再输入对应用户密码。

建议使用普通管理员账户进行远程管理，不建议直接开放 root 密码远程登录。

如需创建普通管理员账户，在 Rocky Linux 中执行：

``` bash
sudo useradd -m rockyadmin
sudo passwd rockyadmin
sudo usermod -aG wheel rockyadmin
```

然后从 Windows 连接：

``` powershell
ssh rockyadmin@192.168.200.10
```

------------------------------------------------------------------------

# 第四部分：最终验证清单

  检查项目             命令或位置                        预期结果
  -------------------- --------------------------------- ---------------------
  Rocky Linux IP       `ip addr`                         `192.168.200.10/24`
  默认网关             `ip route`                        `192.168.200.2`
  NAT 网关连通         `ping -c 4 192.168.200.2`         正常
  公网访问             `curl -I https://www.baidu.com`   返回 HTTP 响应
  Windows VMnet8       `ipconfig`                        `192.168.200.1`
  Windows 访问 Rocky   `ping 192.168.200.10`             允许 ICMP 时正常
  SSH 服务             `systemctl status sshd`           `active (running)`
  SSH 远程连接         `ssh 用户名@192.168.200.10`       成功登录

------------------------------------------------------------------------

# 第五部分：常见问题排查

## 问题 1：Rocky Linux 可以 ping 通网关，但不能访问外网

检查默认路由：

``` bash
ip route
```

应存在：

``` text
default via 192.168.200.2
```

检查 DNS：

``` bash
cat /etc/resolv.conf
```

必要时重新设置 DNS：

``` bash
sudo nmcli connection modify "Wired connection 1" \
ipv4.dns "223.5.5.5 8.8.8.8"

sudo nmcli connection up "Wired connection 1"
```

如果公网 IP 也无法访问，应检查 VMware NAT 服务、宿主机网络和 Windows
防火墙等。

## 问题 2：Rocky Linux 可以访问外网，但 Windows 无法访问 Rocky Linux

检查：

1.  VMnet8 是否启用了宿主机虚拟网卡。
2.  Windows VMnet8 与 Rocky Linux 是否处于同一网段。
3.  虚拟机网络适配器是否选择 NAT。
4.  Rocky Linux 防火墙是否允许 SSH。
5.  Windows 防火墙或其他安全软件是否阻止了通信。

查看 Rocky Linux 防火墙允许的服务：

``` bash
sudo firewall-cmd --list-services
```

输出中应包含：

``` text
ssh
```

## 问题 3：SSH 可以连接，但 root 无法登录

建议创建普通管理员账户：

``` bash
sudo useradd -m rockyadmin
sudo passwd rockyadmin
sudo usermod -aG wheel rockyadmin
```

然后使用该账户登录：

``` powershell
ssh rockyadmin@192.168.200.10
```

## 问题 4：Rocky Linux 重启后 IP 发生变化

查看连接：

``` bash
nmcli connection show
```

检查连接的 IPv4 配置：

``` bash
nmcli connection show "Wired connection 1" | grep ipv4
```

确认静态配置包含：

``` text
ipv4.method: manual
ipv4.addresses: 192.168.200.10/24
```

如果连接名称不同，请替换为实际名称。

------------------------------------------------------------------------

# 第六部分：后续服务访问地址

网络配置完成后，可以在 Rocky Linux
上部署各类服务。以下为常见默认端口示例：

  服务          宿主机访问地址
  ------------- ------------------------------
  SSH           `192.168.200.10:22`
  Nginx         `http://192.168.200.10`
  Spring Boot   `http://192.168.200.10:8080`
  MySQL         `192.168.200.10:3306`
  Redis         `192.168.200.10:6379`

实际访问还取决于服务是否已安装启动、监听地址是否允许远程访问，以及 Rocky
Linux 防火墙是否开放相应端口。

如果以后需要让局域网内其他电脑直接访问 Rocky Linux，可以进一步考虑
VMware NAT 端口转发或桥接模式。

## 配置建议

对于目前的需求，优先使用单网卡 NAT 方案即可：

-   不必增加第二块虚拟网卡。
-   不必使用桥接模式。
-   Rocky Linux 使用固定静态 IP。
-   Windows 通过 VMnet8 访问 Rocky Linux。
-   Rocky Linux 通过 VMware NAT 访问外网。

这种网络结构简单，适合后续安装 Docker、部署数据库、运行 Java
项目和搭建开发测试环境。
