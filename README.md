# padavan-4.4 #

This project is based on original rt-n56u with latest mtk 4.4.198 kernel, which is fetch from D-LINK GPL code.

- Features
  - Based on 4.4.198 Linux kernel
  - Support MT7621 based devices
  - Support MT7615D/MT7615N/MT7915D wireless chips
  - Support raeth and mt7621 hwnat with legency driver
  - Support qca shortcut-fe
  - Support IPv6 NAT based on netfilter
  - Support WireGuard integrated in kernel
  - Support fullcone NAT (by Chion82)
  - Support LED&GPIO control via sysfs


- Supported devices
  - CR660x
  - JCG-Q20
  - JCG-AC860M
  - JCG-836PRO
  - JCG-Y2
  - DIR-878
  - DIR-882
  - K2P
  - K2P-USB
  - NETGEAR-BZV
  - MR2600
  - MI-4
  - MI-R3G
  - MI-R3P
  - R2100
  - XY-C1

- Compilation step
  - Install dependencies
    ```sh
    # Debian/Ubuntu
    sudo apt install unzip libtool-bin curl cmake gperf gawk flex bison nano xxd \
        fakeroot kmod cpio git python3-docutils gettext automake autopoint \
        texinfo build-essential help2man pkg-config zlib1g-dev libgmp3-dev \
        libmpc-dev libmpfr-dev libncurses5-dev libltdl-dev wget libc-dev-bin

    # Archlinux/Manjaro
    sudo pacman -Syu --needed git base-devel cmake gperf ncurses libmpc \
            gmp python-docutils vim rpcsvc-proto fakeroot cpio help2man

    # Alpine
    sudo apk add make gcc g++ cpio curl wget nano xxd kmod \
        pkgconfig rpcgen fakeroot ncurses bash patch \
        bsd-compat-headers python2 python3 zlib-dev \
        automake gettext gettext-dev autoconf bison \
        flex coreutils cmake git libtool gawk sudo
    ```
  - Clone source code
    ```sh
    git clone https://github.com/meisreallyba/padavan-4.4.git
    ```
  - Prepare toolchain
    ```sh
    cd padavan-4.4/toolchain-mipsel

    # (Recommend) Download prebuilt toolchain for x86_64 or aarch64 host
    ./dl_toolchain.sh

    # or build toolchain with crosstool-ng
    # ./build_toolchain
    ```
  - Modify template file and start compiling
    ```sh
    cd padavan-4.4/trunk

    # (Optional) Modify template file
    # nano configs/templates/K2P.config
    # 注意，修改.config文件时，确保文件的最后一行为空行，否则会导致build_firmware_modify插入的配置代码不能换行，从而编译报错

    # Start compiling
    fakeroot ./build_firmware_modify K2P

    # To build firmware for other devices, clean the tree after previous build
    ./clear_tree
    ```

- Manuals
  - Controlling GPIO and LEDs via sysfs
  - How to use NAND RWFS partition
  - How to use IPv6 NAT and fullcone NAT
  - How to add new device support with device tree

# 个人自用 Padavan 固件高级配置教程
> [!NOTE]
> Padavan 不像 OpenWrt，版本相对较老，折腾的人少，教程也偏少，并且限制比较多。因此整理保留了这份自用教程。
> - **测试设备**：基于 MT7621 的路由器（如：京东云路由鲁班、小米 CR660x 等）。
---
## 1. NAND 存储类型设备挂载 `/opt`
对于 NAND 存储类型的设备，如何正确挂载 `/opt` 分区，请参考[恩山论坛帖子](https://www.right.com.cn/forum/forum.php?mod=viewthread&tid=8313063&extra=page%3D1&page=1)。
*(相关备份贴文已放置在仓库的 `自动挂载RWFS文件系统.pdf` 中)*
### 修复因扩容 rwfs 导致的 OpenWrt 可用空间极小问题
**问题描述**
在刷入 OpenWrt / ImmortalWrt 固件后，可能会发现系统的可用空间（`/overlay` 分区）非常小（例如仅剩十多兆），尽管设备本身有 128MB 的物理闪存。
**原因分析**
这是因为此前在padavan系统中扩容的 `rwfs` 用户数据分区，在刷写第三方固件后变成了**未挂载的遗留闲置卷**，白白占用了大量空间。
如果在 SSH 下运行 `ubinfo -a`，通常会看到 117.7 MiB 的逻辑空间被切分成了 3 个卷（我的CR660x路由器是128M闪存，其他路由器的分区分布可能不同）：
- `rootfs` (Volume 1)：只读系统核心段（约占 6MiB）
- `rootfs_data` (Volume 2)：当前的可用空间（约占 17MiB）
- `rwfs` (Volume 0)：原厂遗留闲置卷，罪魁祸首（约占 91MiB）
**解决办法**
删除这个毫无用处的 `rwfs` 遗留卷，并在OpenWrt的webui中重新平刷固件重新分配空间即可：
1. 通过 SSH 登录路由器，执行以下命令强制删除 `rwfs` 卷：
   ```bash
   # 注意此处 -n 0 代表删除 Volume ID 为 0 的卷（即 rwfs 所在的编号）
   ubirmvol /dev/ubi0 -n 0
---
## 2. 双拨号实现分流（一拨 IPv6，二拨 IPv4）
> [!TIP]
> **双拨的基本逻辑：**
> 1. **虚拟网卡 (macvlan)**：由于物理 WAN 口（如 `eth3`）只能被系统原生拨号占用，我们需要通过 `macvlan` 技术在同一个物理网卡上虚拟出一个带有新 MAC 地址的网卡（`vwan1`），以骗过运营商的限制，允许进行第二次独立拨号。
> 2. **原生拨号 (ppp0)**：交由 Padavan 传统的 Web 界面接管并正常配置，主要用来获取 IPv6 地址。由于双拨会导致路由冲突，我们会通过脚本将其默认路由的优先级降低（增大跃点数 / Metric），使其仅仅作为备胎或仅用于 IPv6 流量。
> 3. **二次拨号 (ppp1)**：使用我们生成的虚拟网卡进行命令行后台拨号，获取独立的 IPv4 地址。通过配置策略路由，确保其具有较高优先级（较低的跃点数），并增加 NAT 转发规则，使局域网的 IPv4 流量主要通过这条高速线路转发。
> 4. **后台守护**：辅以监控脚本，在断网或路由改变时自动复活连接或恢复路由表，防止系统的默认程序误杀二次拨号进程。
### 阶段一：在 WAN 上行/下行启动后执行的脚本
在路由器的 **“高级设置 -> 自定义设置 -> 脚本 -> 在 WAN 上行/下行启动后执行”** 中粘贴以下代码：
```bash
# ==== ppp1 自动复活防误杀代码 开始 ====
if [ "$1" == "up" ]; then
    if ! ps | grep -v grep | grep "options.wan1" > /dev/null; then
        WAN_IF="eth3"
        VIRTUAL_WAN="vwan1"
        FAKE_MAC="d6:35:38:44:77:33" # 改成你想设定的MAC地址
        
        ip link del $VIRTUAL_WAN 2>/dev/null
        ip link add link $WAN_IF name $VIRTUAL_WAN type macvlan 2>/dev/null
        ip link set $VIRTUAL_WAN address $FAKE_MAC 2>/dev/null
        ip link set $VIRTUAL_WAN up 2>/dev/null
        
        pppd file /tmp/ppp/options.wan1
    fi
fi
# ==== ppp1 自动复活防误杀代码 结束 ====
# ==== 路由跃点数 (Metric) 修改与防欺骗对策 ====
if [ "$1" == "up" ] && [ "$2" == "ppp0" ]; then
    # 等待3秒，确保系统的 ppp0 路由和 IP 已经完全生成并写入内核
    sleep 3
    
    # ----------------------------------------------------
    # 魔法 1：修改 ppp0 的默认路由跃点数为 20 (降级为备胎)
    # ----------------------------------------------------
    # 先删除系统默认分配的 0 跃点路由
    ip route del default dev ppp0 2>/dev/null
    # 重新添加跃点数为 20 的路由
    ip route add default dev ppp0 metric 20 2>/dev/null
    # ----------------------------------------------------
    # 魔法 2：解决双拨回程非对称路由的“防欺骗”对策
    # ----------------------------------------------------
    IP_PPP0=$(ip addr show ppp0 2>/dev/null | grep -w inet | awk '{print $2}')
    if [ -n "$IP_PPP0" ]; then
        ip rule del table 100 2>/dev/null
        ip route flush table 100 2>/dev/null
        
        ip rule add from $IP_PPP0 table 100
        ip route add default dev ppp0 table 100
    fi
fi
# ===============================================
```
### 阶段二：在路由器启动后执行的脚本
在路由器的 **“高级设置 -> 自定义设置 -> 脚本 -> 在路由器启动后执行”** 中粘贴以下代码（请记得替换账号密码）：
```bash
# ==== 第二个拨号（带MAC伪装）的脚本配置 开始 ====
USER_V4="（填入你的宽带账号）"
PASS_V4="（填入你的宽带密码）"
# 1. 获取物理 WAN 口名称 (如 eth3)
WAN_IF="eth3" 
# 2. 创建虚拟网卡(macvlan)并赋予随机伪装MAC
VIRTUAL_WAN="vwan1"
FAKE_MAC="d6:35:38:44:77:33" # 记得与上一段脚本中的 MAC 保持一致
ip link del $VIRTUAL_WAN 2>/dev/null
ip link add link $WAN_IF name $VIRTUAL_WAN type macvlan 2>/dev/null
ip link set $VIRTUAL_WAN address $FAKE_MAC 2>/dev/null
ip link set $VIRTUAL_WAN up 2>/dev/null
# 3. 生成拨号配置 (保持 nodefaultroute，不让 pppd 乱加路由，由我们自己管理)
cat > /tmp/ppp/options.wan1 <<EOF
plugin rp-pppoe.so nic-$VIRTUAL_WAN
user "$USER_V4"
password "$PASS_V4"
nomppe nomppc
noauth
nodefaultroute
unit 1
persist
holdoff 10
maxfail 0
lcp-echo-interval 10
lcp-echo-failure 5
mtu 1492
mru 1492
EOF
# 4. 启动拨号进程
if ! ps | grep -v grep | grep "options.wan1" > /dev/null; then
    pppd file /tmp/ppp/options.wan1
fi
# 5. 启动轻量级后台守护进程（负责 ppp1 的高优先级路由和NAT）
(
    sleep 5
    while true; do
        # 只要 ppp1 在线
        if ip link show ppp1 2>/dev/null | grep -q "UP"; then
            # 确保 NAT 规则存在
            iptables -t nat -C POSTROUTING -o ppp1 -j MASQUERADE 2>/dev/null || \
            iptables -t nat -A POSTROUTING -o ppp1 -j MASQUERADE
            
            # 检查 ppp1 的 Metric 10 路由是否存在，不存在则添加
            if ! ip route show default dev ppp1 2>/dev/null | grep -q "metric 10"; then
                ip route add default dev ppp1 metric 10 2>/dev/null
            fi
        fi
        sleep 5
    done
) &
# ==== 第二个拨号（带MAC伪装）的脚本配置 结束 ====
```
---
## 3. 在 Padavan 上部署 Tailscale 内网穿透
> [!IMPORTANT]
> 前提条件：已经配置好 `/opt` 挂载，我们将把 Tailscale 直接装到该目录下以防重启后丢失进度。
### 第一步：下载并释放 Tailscale 核心程序
通过 SSH 登录终端，依次执行以下命令：
```bash
cd /opt
mkdir -p /opt/bin
# 1. 使用 curl 的 -k (跳过SSL验证) 强行下载官方 MIPSLE 静态编译包 (以 1.66.4 为例)
curl -k -O https://pkgs.tailscale.com/stable/tailscale_1.66.4_mipsle.tgz
# 2. 把压缩包解开
tar -xzvf tailscale_1.66.4_mipsle.tgz
# 3. 提取最核心的两个执行文件（守护进程与命令行客户端）
cp tailscale_1.66.4_mipsle/tailscaled /opt/bin/
cp tailscale_1.66.4_mipsle/tailscale /opt/bin/
chmod +x /opt/bin/tailscale*
# 4. 删除安装包和临时文件夹，释放空间
rm -rf tailscale_1.66.4_mipsle*
```
### 第二步：启动守护进程并做到状态持久化
很多教程重启后 Tailscale 会掉登录状态，那是因为状态文件没写入真实的存储中。将其存到 `/opt` 分区：
```bash
# 1. 在 /opt 里创建一个专门用来存登录授权证书的持久化文件夹
mkdir -p /opt/tailscale_state
# 2. 正式启动 Tailscale 核心后台服务！
/opt/bin/tailscaled --state=/opt/tailscale_state/tailscaled.state --statedir=/opt/tailscale_state/ --socket=/var/run/tailscaled.sock --port=41641 &
```
*(敲完这句如果你看到一些类似启动日志的东西在跳，直接敲一下回车键就能回到控制台)*
### 第三步：生成登录链接与组网
后台服务启动后，输入以下命令获取扫码登录的认证链接：
```bash
# 发起登录请求
/opt/bin/tailscale --socket=/var/run/tailscaled.sock up --netfilter-mode=off
```
此时，屏幕上会弹出一个类似 `https://login.tailscale.com/a/xxxxxx` 的链接，将链接复制进浏览器打开并登录账号即可设备组网。
### 第四步：注入开机自启及防火墙规则
前往 **Web 管理后台 -> “高级设置 -> 自定义设置 -> 脚本”**
1. 在 **在路由器启动后执行** 下方空白处插入：
```bash
# ==== 启动内网穿透 Tailscale ====
/opt/bin/tailscaled --state=/opt/tailscale_state/tailscaled.state --statedir=/opt/tailscale_state/ --socket=/var/run/tailscaled.sock --port=41641 &
```
2. 个人需求是让路由器下面的局域网设备也能直接访问 `100.x.x.x` 的网络，并在校园网等环境打开 udp 打通隧道。
在 **在防火墙规则启动后执行 (Run After Firewall Rules Restarted)** 的空白处插入：
```bash
# ======== Tailscale 网络防火墙配置 ========
iptables -I FORWARD -i tailscale0 -j ACCEPT
iptables -I FORWARD -o tailscale0 -j ACCEPT
iptables -t nat -I POSTROUTING -o tailscale0 -j MASQUERADE
# 允许外部访问 41641 用于 P2P 连接
iptables -I INPUT -p udp --dport 41641 -j ACCEPT
# ============================================
```
---
## 4. 解决特定设备动态 IPv6 端口开放问题（UGNAS 等设备）
> [!NOTE]
> **背景说明**：正常设备的 IPv6 启用 SLAAC 后会有固定的后 64 位后缀，但部分厂商（如 UGNAS）强制开启了隐私扩展使得 IP 尾部随意变化。如果在 Padavan 关闭 SLAAC 仅开启 DHCPv6 将导致 Android 手机无法获取 IPv6！
> **解决思路**：虽 IP 动态，但设备的物理 MAC 地址固定，通过定期扫描路由器的 **IPv6 邻居表 (ndp)** 匹配物理网卡获取瞬时 IPv6 地址，动态增删防火墙放行规则。
### 部署动态防火墙脚本
进入 SSH，新建脚本 `/etc/storage/user_update_ugnas_ipv6_fw.sh`，内容如下：
```bash
#!/bin/bash
# 配置区域（填入设备的真实网卡 MAC 地址和需要开放的同伴端口）
MAC="（填入设备的mac地址，例如 AA:BB:CC:DD:EE:FF）"
TCP_PORTS="（例如: 80,443,5000）"
UDP_PORTS="（例如: 5000）"
LASTIP_FILE="/tmp/user_last_ugnas_ipv6"
# Get the current REACHABLE IPv6 address (pick the first one)
cur_ipv6="$(ip -6 neigh show | grep -i $MAC | grep REACHABLE | grep '^240e:' | awk '{print $1}' | head -n 1)"
# Exit if no IPv6 address found
[ -z "$cur_ipv6" ] && exit 0
# Read last IPv6 address
[ -f "$LASTIP_FILE" ] && last_ipv6="$(cat $LASTIP_FILE)" || last_ipv6=""
# 如果发现 IP 没有变化，直接退出（不产生额外负载）
if [ "$cur_ipv6" = "$last_ipv6" ]; then
    exit 0
fi
# 如果旧规则存在，先予以抹除
if [ -n "$last_ipv6" ]; then
    ip6tables -D FORWARD -d "$last_ipv6" -p tcp -m state --state NEW -m tcp -m multiport --dports $TCP_PORTS -j ACCEPT 2>/dev/null
    ip6tables -D FORWARD -d "$last_ipv6" -p udp -m state --state NEW -m udp -m multiport --dports $UDP_PORTS -j ACCEPT 2>/dev/null
fi
# 面向新的动态 IPv6 添加放通规则
ip6tables -A FORWARD -d "$cur_ipv6" -p tcp -m state --state NEW -m tcp -m multiport --dports $TCP_PORTS -j ACCEPT
ip6tables -A FORWARD -d "$cur_ipv6" -p udp -m state --state NEW -m udp -m multiport --dports $UDP_PORTS -j ACCEPT
# Save the current IPv6 address for next run
echo "$cur_ipv6" > "$LASTIP_FILE"
```
然后利用 cron 使得系统每分钟检查一次，将命令填入 Crontab 计划任务：
```cron
*/1 * * * * /bin/bash /etc/storage/user_update_ugnas_ipv6_fw.sh 
```
---
## 5. 配置 Dynv6 DDNS 动态域名解析
> [!TIP]
> 针对通过 IPv6 外网访问的需求，使用 dynv6 的 Api 进行更新解析。
在 `/etc/storage` 创建一 个 `.sh` 脚本（如 `/etc/storage/user_dynv6_update.sh`）：
```bash
#!/bin/sh                                                                                                               
                                                                                                                        
# --- 配置区域 ---                                                                                                      
TOKEN="（填充你的Token）"                                                                                  
DOMAIN="（填充你的域名）"                                                                                         
# 临时文件用于记录上一次的IP，避免由于 IP 未变时的重复无效请求                                                                              
IP_FILE="/tmp/luban_last_ipv6.txt"                                                                                      
# --- --- --- ---                                                                                                       
                                                                                                                        
# 获取当前 IPv6 地址 (采用外部辅助验证)                                                                                    
CURRENT_IP=$(curl -s 6.ipw.cn)                                                                                          
                                                                                                                        
# 检查获取是否成功（判断是否包含冒号）                                                                                  
if echo "$CURRENT_IP" | grep -q ":"; then                                                                               
    echo "当前 IPv6: $CURRENT_IP"                                                                                       
else                                                                                                                    
    echo "错误：无法获取 IPv6 地址"                                                                                     
    exit 1                                                                                                              
fi                                                                                                                      
                                                                                                                        
# 检查 IP 是否有变化                                                                                                    
if [ -f "$IP_FILE" ]; then                                                                                              
    LAST_IP=$(cat "$IP_FILE")                                                                                           
    if [ "$CURRENT_IP" = "$LAST_IP" ]; then                                                                             
        echo "IP 未变化，跳过更新。"                                                                                    
        exit 0                                                                                                          
    fi                                                                                                                  
fi                                                                                                                      
                                                                                                                        
# 更新 dynv6                                                                                                            
URL="http://dynv6.com/api/update?hostname=$DOMAIN&token=$TOKEN&ipv6=$CURRENT_IP"                                        
RESPONSE=$(curl -s "$URL")                                                                                              
                                                                                                                        
if [ "$RESPONSE" = "addresses updated" ] || [ "$RESPONSE" = "addresses unchanged" ]; then                               
    echo "dynv6 更新成功: $RESPONSE"                                                                                    
    echo "$CURRENT_IP" > "$IP_FILE"                                                                                     
else                                                                                                                    
    echo "dynv6 更新失败，服务器响应: $RESPONSE"                                                                        
    exit 1                                                                                                              
fi                                                    
```
然后利用 cron，将命令填入 Crontab 计划任务来定时心跳执行：
```cron
*/1 * * * * /etc/storage/user_dynv6_update.sh 
```
---
## 6. 修复由 SSL 证书引起的 `x509: certificate signed by unknown authority` 报错
> [!WARNING]
> 在 Padavan 系统上运行一些采用 Golang 编写的应用程序并尝试主动外网发起 TLS(HTTPS) 请求时，可能会因为原生系统底层 CA 根证书严重老化缺失，报出类似证书机构未知的致命故障。
**针对此问题，可以尝试以下两种修复方法（选择任一即可应用）：**
### 方案一：通过 Entware 安装系统证书更新包（如果您已安装 Entware）
推荐使用该原生包管理进行安装，使用其自带的 CA 证书替换链。
1. 在终端执行以下命令直接获取最新证书包：
   ```bash
   opkg update
   opkg upgrade
   opkg install ca-bundle
   opkg install ca-certificates
   ```
2. 需要运行您的代码/服务之前，在环境中执行以下声明（注：请确认您的 Entware 资源所部署的确切路径，以下为通常默认路径）：
   ```bash
   export SSL_CERT_FILE=/opt/etc/ssl/certs/ca-certificates.crt
   export SSL_CERT_DIR=/opt/etc/ssl/certs
   ```
### 方案二：手动下载单一公钥凭证并显式挂载（快捷有效）
直接获取最新的公开 `pem` 文件作为受信任的授权。
1. 将官方根证书库下载到你的持久性存储区域内（如 `/opt/etc/`）：
   ```bash
   curl -k -o /opt/etc/cacert.pem https://curl.haxx.se/ca/cacert.pem
   ```
2. 在运行您的程序之前，在环境中提前进行注入挂载声明即可解决拦截：
   ```bash
   export SSL_CERT_FILE=/opt/etc/cacert.pem
   ```
将对应方案获取后的 `export` 环境变量语句补到相关程序或启动脚本的首行即可彻底解决问题！
