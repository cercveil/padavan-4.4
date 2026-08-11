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
  - Debian 13 编译环境注意事项：`modules.dep` 没有生成
    ```sh
    # Debian 13 自带的 kmod/depmod 版本较新，已经不支持旧的 -r 参数。
    # 如果 tools/depmod.sh 仍然使用下面这种调用方式：
    #   /sbin/depmod ... -r ${KERNELRELEASE}
    # depmod 会报 invalid option -- 'r'，导致 romfs/lib/modules/<kernel-version>/
    # 下没有生成 modules.dep。
    #
    # modules.dep 缺失后，固件里的 modprobe 不能按模块名自动加载 .ko，
    # 例如 iptables -t mangle -L 会因为 iptable_mangle.ko 无法自动加载而报：
    #   can't initialize iptables table `mangle': Table does not exist

    # 解决方案：修改 tools/depmod.sh，去掉 -r，并让 depmod 失败时中止编译：
    $depmod_bin -ae -F System.map -b "${INSTALL_MOD_PATH}" ${KERNELRELEASE} || exit $?

    # 重新完整编译后，检查 romfs 里是否已经生成 modules.dep：
    find romfs/lib/modules -maxdepth 2 -type f -name modules.dep -ls
    grep iptable_mangle romfs/lib/modules/*/modules.dep
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
> 1. 整这个的原因是我这个学校的校园网，一个免费账号的ipv4限速，ipv6不限速，另一个付费账号ipv4不限速，但是没有ipv6。且这两个账号都能在同一个有线端口上执行拨号。其实openwrt折腾这个最方便，可惜不知道什么原因，官方的op刷入后会经常断流。lean的lede更别说了，那个turboacc插件动了配置，就再也别想开启hnat了。只能选择相对来讲较为稳定的老毛子。
> 2. **虚拟网卡 (macvlan)**：由于物理 WAN 口（如 `eth3`）只能被系统原生拨号占用，我们需要通过 `macvlan` 技术在同一个物理网卡上虚拟出一个带有新 MAC 地址的网卡（`vwan1`），以骗过运营商的限制，允许进行第二次独立拨号ppp1。
> 3. **原生拨号 (ppp0)**：交由 Padavan 传统的 Web 界面接管并正常配置，主要用来获取 IPv6 地址。由于双拨会导致路由冲突，我们会通过脚本将其默认路由的优先级降低（增大跃点数 / Metric），使其仅仅作为备胎或仅用于 IPv6 流量。
> 4. **二次拨号 (ppp1)**：使用我们生成的虚拟网卡进行命令行后台拨号，获取独立的 IPv4 地址。通过配置策略路由，确保其具有较高优先级（较低的跃点数），并增加 NAT 转发规则，使局域网的 IPv4 流量主要通过这条高速线路转发。
> 5. **后台守护**：辅以监控脚本，在断网或路由改变时自动复活连接或恢复路由表，防止系统的默认程序误杀二次拨号进程。
> 6. 下面的方案对硬件加速有有限支持，同等100M流量下cpu占用比单拨要高，但是比不开硬件加速要低一些。个人建议还是禁用hwnat，然后用shortcut-fe加速，hwnat的某些bug简直离谱。或者ppp0负责ipv4，ppp1负责ipv6，然后ppp0正常走hwnat，ppp1因为校园网需要做nat66本来就没有硬件加速，这样的方案比下面的方案简单很多。
> 7. 因为padavan很多后端是针对ppp0配置的，对双ppp支持不好，因此如果需要两个ppp的v4都能入站，还需要做一些对应调整。
### 阶段一：在 WAN 上行/下行启动后执行的脚本
padavan在webui手动重连ppp0后会kill掉其他ppp进程，因此需要在这里复活ppp1，并配置ppp0接口的路由规则。
在路由器的 **“高级设置 -> 自定义设置 -> 脚本 -> 在 WAN 上行/下行启动后执行”** 中粘贴以下代码：
```bash
# ==== ppp1 自动复活与进程守护 开始 ====
# 1. ppp0 上线逻辑：配置降级跃点数与表 100
if [ "$1" == "up" ] && [ "$2" == "ppp0" ]; then
    ip route del default dev ppp0 2>/dev/null
    ip route add default dev ppp0 metric 20 2>/dev/null

    IP_PPP0=$(ip addr show ppp0 2>/dev/null | grep -w inet | awk '{print $2}')
    if [ -n "$IP_PPP0" ]; then
        while ip rule show | grep -q "lookup 100"; do
            ip rule del table 100 2>/dev/null
        done
        ip rule add from $IP_PPP0 table 100
        ip rule add fwmark 0x100 table 100
        ip route flush table 100 2>/dev/null
        ip route add default dev ppp0 table 100 2>/dev/null

        LAN_NET=$(ip route show dev br0 2>/dev/null | grep 'proto kernel' | awk '{print $1}')
        if [ -n "$LAN_NET" ]; then
            ip route add $LAN_NET dev br0 table 100 2>/dev/null
        fi
    fi
    
    echo 2 > /proc/sys/net/ipv4/conf/ppp0/rp_filter 2>/dev/null
    
    if [ -f "/tmp/ppp/options.wan1" ]; then
        if ! ps | grep -v grep | grep -q "options.wan1"; then
            logger -t "wan-script" "[信息] 检测到 ppp1 进程不存在，执行拨号..."
            pppd file /tmp/ppp/options.wan1
        fi
    fi
fi
# ==== ppp1 自动复活与进程守护 结束 ====
```
### 阶段二：在路由器启动后执行的脚本
此段脚本负责初始化第二次拨号ppp1，并启动后台守护进程，以维持 ppp1 自己被运营商强制断线重连后的规则配置。
在路由器的 **“高级设置 -> 自定义设置 -> 脚本 -> 在路由器启动后执行”** 中粘贴以下代码（请记得替换账号密码）：
```bash
# ==== 第二拨号（带MAC伪装）的初始化配置 开始 ====
USER_V4="（填入你的宽带账号）"
PASS_V4="（填入你的宽带密码）"
WAN_IF="eth3"
VIRTUAL_WAN="vwan1"
FAKE_MAC="d6:35:38:44:77:33"

# 1. 初始化虚拟网卡
if ! ip link show $VIRTUAL_WAN > /dev/null 2>&1; then
    ip link add link $WAN_IF name $VIRTUAL_WAN type macvlan 2>/dev/null
    ip link set $VIRTUAL_WAN address $FAKE_MAC 2>/dev/null
    ip link set $VIRTUAL_WAN up 2>/dev/null
fi

# 2. 生成配置
mkdir -p /tmp/ppp
cat > /tmp/ppp/options.wan1 <<EOF
plugin rp-pppoe.so nic-$VIRTUAL_WAN
user "$USER_V4"
password "$PASS_V4"
nomppe nomppc
noauth
nodefaultroute
unit 1
persist
maxfail 3
holdoff 10
lcp-echo-interval 10
lcp-echo-failure 5
mtu 1492
mru 1492
EOF

# 3. 初始执行 ppp1 拨号
if ! ps | grep -v grep | grep -q "options.wan1"; then
    pppd file /tmp/ppp/options.wan1
fi

# 4. 守护程序
(
    while true; do
        # 如果 ppp1 处于在线状态
        if ip link show ppp1 2>/dev/null | grep -q "UP"; then
            IP_PPP1=$(ip addr show ppp1 2>/dev/null | grep -w inet | awk '{print $2}')
            if [ -n "$IP_PPP1" ]; then
                
                # 检查默认路由 (metric 10) 是否存在
                if ! ip route show default dev ppp1 2>/dev/null | grep -q "metric 10"; then
                    ip route add default dev ppp1 metric 10 2>/dev/null
                fi
                
                # 检查策略规则 (是否精确指向了当前的 $IP_PPP1)
                # 如果找不到带有当前 IP 的规则，说明没有发生过第一次拨号，或者重拨，或者规则被意外清空
                if ! ip rule show | grep -q "from $IP_PPP1 lookup 200"; then
                    logger -t "ppp1-watchdog" "[修复] ppp1 策略规则失效/遗失，正在重建表 200..."
                    
                    while ip rule show | grep -q "lookup 200"; do
                        ip rule del table 200 2>/dev/null
                    done
                    
                    ip rule add fwmark 0x200 table 200 2>/dev/null
                    ip rule add from $IP_PPP1 table 200 2>/dev/null
                    ip route flush table 200 2>/dev/null
                    ip route add default dev ppp1 table 200 2>/dev/null

                    LAN_NET=$(ip route show dev br0 2>/dev/null | grep 'proto kernel' | awk '{print $1}')
                    if [ -n "$LAN_NET" ]; then
                        ip route add $LAN_NET dev br0 table 200 2>/dev/null
                    fi
                fi
            fi
        fi
        sleep 15
    done
) &
# ==== 第二拨号（带MAC伪装）的初始化配置 结束 ====
```
### 阶段三：配置防火墙规则
这里主要配置两项，一项是ppp1的nat，方便路由器下级设备上网；另一项是ppp0和ppp1的v4入站连接，主要是方便随时调试路由器。之前阶段一里面宽容了rp_filter，也是为了ppp0的入站设计的，如果不想宽容，替换那个echo指令为echo 1 > /proc/sys/net/ipv4/conf/ppp0/src_valid_mark也行。
```bash
# 配置双拨nat与入站方案
# 为 ppp1 配置基于 MASQUERADE 补丁的 FullCone NAT
iptables -t nat -C POSTROUTING -o ppp1 -j MASQUERADE --mode fullcone 2>/dev/null || \
iptables -t nat -A POSTROUTING -o ppp1 -j MASQUERADE --mode fullcone

# 2. 确保 ppp 入站流量进入端口转发/UPnP 链
iptables -t nat -C PREROUTING -i ppp1 -j vserver 2>/dev/null || \
iptables -t nat -A PREROUTING -i ppp1 -j vserver

# 1. 进门登记：只要是从 ppp0/ppp1 进来的新连接，在连接表里记下专属标记
iptables -t mangle -C PREROUTING -i ppp0 -m conntrack --ctstate NEW -j CONNMARK --set-mark 0x100 2>/dev/null || \
iptables -t mangle -A PREROUTING -i ppp0 -m conntrack --ctstate NEW -j CONNMARK --set-mark 0x100

iptables -t mangle -C PREROUTING -i ppp1 -m conntrack --ctstate NEW -j CONNMARK --set-mark 0x200 2>/dev/null || \
iptables -t mangle -A PREROUTING -i ppp1 -m conntrack --ctstate NEW -j CONNMARK --set-mark 0x200

# 2. 回包打标 (内网端口转发的回包)：从局域网返回的数据包经过 PREROUTING 时，提取标记
iptables -t mangle -C PREROUTING -j CONNMARK --restore-mark 2>/dev/null || \
iptables -t mangle -A PREROUTING -j CONNMARK --restore-mark

# 3. 回包打标 (路由器 WebUI 等本机服务的回包)：从路由器本机会产生的包经过 OUTPUT 时，提取标记
iptables -t mangle -C OUTPUT -j CONNMARK --restore-mark 2>/dev/null || \
iptables -t mangle -A OUTPUT -j CONNMARK --restore-mark
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
iptables -A INPUT -i tailscale0 -j ACCEPT
iptables -t nat -A POSTROUTING -o tailscale0 -j MASQUERADE
# 允许外部访问 41641 用于 P2P 连接
iptables -A INPUT -p udp --dport 41641 -j ACCEPT
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

# 7. 开启 TCP MSS 钳制
> PPPoE 拨号会使链路 MTU 从 1500 降至 1492。如果没有 MSS 钳制，TCP 握手时协商的 MSS 可能超出实际 MTU，导致大包被 IP 层分片，严重时会出现网页打不开、下载卡住等问题。MSS 钳制是 PPPoE 环境下的基本必备配置。

## 原理简述
MSS 钳制通过 iptables 的 TCPMSS target，在 TCP SYN/SYN-ACK 握手包经过路由器时，将 MSS 值自动压低到当前出口链路的 PMTU 所允许的最大值，从根源上避免 TCP 数据包超过 MTU 而被分片。

## 配置方法
前往 `高级设置 → 自定义设置 → 脚本 → 在防火墙规则启动后执行`，添加以下规则：

```bash
# ======== TCP MSS 钳制（IPv4 + IPv6） ========
# --- IPv4 MSS Clamping ---
# 对所有经过路由器转发的 TCP SYN 包，将 MSS 钳制到 PMTU 允许的最大值
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
# 对路由器自身发出的 TCP SYN 包也做钳制（如路由器自身访问外网）
iptables -t mangle -A OUTPUT  -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

# --- IPv6 MSS Clamping ---
# IPv6 同理，PPPoE 下 IPv6 的 MTU 同样受限
ip6tables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
ip6tables -t mangle -A OUTPUT  -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
# =============================================
```

## 验证是否生效
配置保存并重启防火墙后，通过 SSH 执行：

```bash
# 查看 IPv4 规则
iptables -t mangle -L FORWARD -v -n | grep -i tcpmss
iptables -t mangle -L OUTPUT  -v -n | grep -i tcpmss

# 查看 IPv6 规则
ip6tables -t mangle -L FORWARD -v -n | grep -i tcpmss
ip6tables -t mangle -L OUTPUT  -v -n | grep -i tcpmss
```

### 正常输出示例
```
0     0 TCPMSS     tcp  --  *      *       0.0.0.0/0    0.0.0.0/0    tcp flags:0x06/0x02 TCPMSS clamp to PMTU
```

> **TIP**
> `--clamp-mss-to-pmtu` 会自动根据出口接口的 MTU 计算最优 MSS 值（MTU - 40 对于 IPv4，MTU - 60 对于 IPv6），无需手动指定数字，即使 MTU 发生变化也能自动适配。

# 8. 优化 Android 设备待机后的 IPv6 RA 保活

> 部分 Android 手机在 Wi-Fi 待机较长时间后，可能会出现 IPv6 地址或默认 IPv6 路由丢失的问题。典型表现是：手机 Wi-Fi 页面仍显示连接正常，但 IPv6 测试失败，`ip -6 route` 中默认路由消失，断开并重新连接 Wi-Fi 后又恢复正常。该问题通常与 Android 设备待机省电、APF 过滤、Wi-Fi 多播接收以及 Router Advertisement（RA）刷新不稳定有关。见：https://issuetracker.google.com/issues/241959699

## 原理简述

IPv6 SLAAC 依赖路由器周期性发送 RA（Router Advertisement）来维持客户端的 IPv6 前缀、默认路由和 DNS 信息。
在 Wi-Fi 待机场景下，部分 Android 设备可能会因为省电策略漏收多播 RA，导致默认 IPv6 路由的 lifetime 逐渐倒计时归零，最终出现 IPv6 失效。

Padavan 默认配置中，RA lifetime 可能较短。如果手机在待机期间连续漏收 RA，就容易触发该问题。解决思路主要有两个：

1. **提高 RA 默认路由 lifetime**，让手机即使漏收一部分 RA，也不至于很快丢失默认路由；
2. **调整 DTIM 间隔**，改善 Wi-Fi 待机状态下多播/广播包的接收稳定性。

## 配置方法一：提高 RA 默认路由 Lifetime

前往 `内网LAN → DHCP服务器 → 高级设置 → dnsmasq.conf`，在自定义 `dnsmasq.conf` 中添加：

```conf
# ======== Android Wi-Fi IPv6 RA 保活优化 ========
# br0：LAN 桥接口
# 30：RA 发送间隔，单位秒
# 9000：Router Lifetime，单位秒，避免 Android 待机漏收 RA 后过快丢默认路由
ra-param=br0,30,9000
# ==============================================
```

配置保存并应用后，重启 dnsmasq 或重启路由器使其生效。

## 配置方法二：调整 DTIM 间隔

前往 `无线 2.4GHz/5GHz → 高级设置`，将对应 Wi-Fi 频段的：

```text
DTIM 间隔：1 → 5
```

如果 2.4G 和 5G 都会连接 Android 设备，建议两个频段都改为 `5`。

DTIM 间隔会影响 Wi-Fi 省电设备接收广播/多播包的时机。IPv6 RA 属于多播流量，适当提高 DTIM 间隔有助于改善部分 Android 设备待机后漏收 RA 的问题。

> **TIP**
> DTIM 并不是越大越好。`5` 是一个相对折中的值：既能改善部分 Android 设备待机状态下的多播接收，又不会让广播/多播流量延迟过高。一般不建议一开始就设置为 10 或更大。
