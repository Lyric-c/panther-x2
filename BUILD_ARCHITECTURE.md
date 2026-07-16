# Panther X2 — Armbian 双仓库构建架构文档

## 仓库定位

```
Lyric-c/panther-x2                  Lyric-c/amlogic-s9xxx-armbian
（内核编译仓库）                       （系统打包仓库，fork 自 ophub）
         │                                      │
         │  Build Kernel action                 │  Build Armbian for Panther X2
         │  → 编译 vendor/current/edge 内核      │  → 从 panther-x2 Release 下载内核
         │  → 打包为 ophub 兼容格式              │  → 从 ophub 上游 Release 下载根文件系统
         │  → 上传到 Release                     │  → rebuild 脚本组装完整镜像
         │                                      │
         ▼                                      ▼
  Release: kernel_rk35xx               Release: Armbian_trixie_arm64_...
  Release: kernel_stable
```

| 仓库 | 定位 | 产物 |
|---|---|---|
| `Lyric-c/panther-x2` | 编译 Panther X2 专用内核，输出 ophub 兼容格式 | `6.1.115.tar.gz` → `kernel_rk35xx` tag |
| `Lyric-c/amlogic-s9xxx-armbian` | Fork 自 [ophub/amlogic-s9xxx-armbian](https://github.com/ophub/amlogic-s9xxx-armbian)，打包 Armbian 镜像 | `Armbian_*.img.gz` |

---

## 原理概述

### 整体流程

```
armbian/build 官方框架
       │
       │  ./compile.sh kernel
       │  + userpatches (panther-x2.conf + DTS)
       ▼
  linux-*.deb (4 个)
       │
       │  dpkg -x → 提取 → 重命名 → 打包
       ▼
  6.1.115.tar.gz (ophub 格式)
  ├── 6.1.115/
  │   ├── boot-6.1.115-rk35xx.tar.gz
  │   ├── dtb-rockchip-6.1.115-rk35xx.tar.gz
  │   ├── modules-6.1.115-rk35xx.tar.gz
  │   ├── header-6.1.115-rk35xx.tar.gz
  │   └── sha256sums
       │
       │  上传到 panther-x2 Release
       ▼
  GitHub Release: kernel_rk35xx/6.1.115.tar.gz
       │
       │  amlogic-s9xxx-armbian workflow:
       │  1. 从 ophub 上游下载 Armbian rootfs 镜像
       │  2. sed 修正 model_database.conf 内核标签
       │  3. sudo ./rebuild → 自动查询/下载内核 → 组装
       ▼
  Armbian_*.img.gz (可烧录镜像)
```

### ophub rebuild 脚本核心逻辑

`rebuild` 脚本做三件事：

1. **内核下载**：从 `kernel_repo` 的 Release 中查询匹配的内核版本，下载 `{ver}.tar.gz`
2. **内核替换**：解压 boot/dt/modules/header 四个 tar.gz，覆盖到 rootfs 的对应路径
3. **引导重配**：根据 `model_database.conf` 写入正确的 DTB 引用、U-Boot、启动脚本

内核版本查询与下载路径示例：

```bash
# query_kernel: 从 kernel_rk35xx tag 的 assets 中匹配 6.1.{X} 最新版
curl https://github.com/Lyric-c/panther-x2/releases/expanded_assets/kernel_rk35xx \
  | grep -oP "6\.1\.[0-9]+(?=\.tar\.gz)" | sort -urV | head -1
# → 6.1.115

# download_kernel: 下载
curl -o kernel.tar.gz \
  https://github.com/Lyric-c/panther-x2/releases/download/kernel_rk35xx/6.1.115.tar.gz

# replace_kernel: 解压到 rootfs
tar -mxzf kernel.tar.gz -C kernel_cache/rk35xx/   # → kernel_cache/rk35xx/6.1.115/
tar -mxzf boot-*.tar.gz -C /boot
tar -mxzf dtb-*.tar.gz -C /boot/dtb/rockchip
tar -mxzf modules-*.tar.gz -C /usr/lib/modules
tar -mxzf header-*.tar.gz -C /usr/src/linux-headers-*
```

---

## panther-x2 仓库改动详情

### 删除

- `.github/workflows/build-armbian.yml` — 不再由该仓库打包系统

### 修改

`.github/workflows/build-kernel.yml` — 所有改动集中在此文件：

| 改动 | 行号 | 说明 |
|---|---|---|
| `PREFER_DOCKER=yes` | L59 | 非 root 环境通过 Docker 编译 |
| `softprops/action-gh-release@v3` | L172 | 消除 Node 20 弃用警告 |
| Prepare Kernel 步骤完全重写 | L63-154 | 从打包 deb 改为打包 ophub 格式 |
| Release tag 映射 | L160-169 | vendor→kernel_rk35xx, current/edge→kernel_stable |

### Prepare Kernel 步骤详解

```bash
# 1. 从 deb 文件名提取 KERNEL_VERSION (如 6.1.115)
KERNEL_VERSION=$(echo "${Kernel_File}" | cut -d '_' -f 5 | cut -d '-' -f 1)

# 2. 确定 KERNEL_NAME (如 6.1.115-rk35xx)
case BRANCH in vendor) SIG="rk35xx" ;; *) SIG="stable" ;; esac
KERNEL_NAME="${KERNEL_VERSION}-${KERNEL_SIG}"

# 3. dpkg -x 全部解压到 /tmp/kernel-extract/
for deb in linux-*.deb; do dpkg -x "$deb" /tmp/kernel-extract/; done

# 4. 逐组件重命名 + 打包

# boot: 文件重命名 + 生成 uInitrd → vmlinuz-6.1.115-rk35xx 等
# 如果 armbian kernel-only build 未生成 uInitrd，自动创建 minimal initramfs
if ! ls boot/uInitrd-* >/dev/null 2>&1; then
  # 用 cpio+gzip+mkimage 生成最小 uInitrd
  mkimage -A arm64 -T ramdisk -C gzip -d /tmp/initrd.img boot/uInitrd-${KERNEL_NAME}
fi
for f in boot/vmlinuz-* boot/config-* boot/System.map-* boot/uInitrd-*; do
  prefix="${f%%-*}"                         # boot/vmlinuz
  newname="${prefix}-${KERNEL_NAME}"        # boot/vmlinuz-6.1.115-rk35xx
  [ "$f" != "${newname}" ] && mv "$f" "${newname}"
done
rm -rf boot/dtb-*   # DTB 由 dtb-rockchip-*.tar.gz 提供，此处冗余
tar -czf boot-${KERNEL_NAME}.tar.gz --owner=0 --group=0 -C boot $(ls boot/)

# dtb: 从 deb 中找到 rockchip/ 子目录
DTB_SRC=$(find boot usr/lib -type d -name "rockchip" | head -1)
tar -czf dtb-rockchip-${KERNEL_NAME}.tar.gz -C "${DTB_SRC}" .

# modules: 目录重命名 → 6.1.115-rk35xx
old_mod_dir=$(ls -d lib/modules/*/ | head -1)
mv "$old_mod_dir" "lib/modules/${KERNEL_NAME}"
tar -czf modules-${KERNEL_NAME}.tar.gz -C lib/modules .

# header: 目录重命名 → linux-headers-6.1.115-rk35xx，从目录内 tar
old_hdr_dir=$(ls -d usr/src/linux-headers-*/ | head -1)
mv "$old_hdr_dir" "usr/src/linux-headers-${KERNEL_NAME}"
tar -czf header-${KERNEL_NAME}.tar.gz -C "usr/src/linux-headers-${KERNEL_NAME}" .

# 5. 生成 sha256sums + 打包为 6.1.115.tar.gz (含 6.1.115/ 子目录)
sha256sum *.tar.gz > sha256sums
mkdir ../${KERNEL_VERSION} && mv * ../${KERNEL_VERSION}/
tar -czf ../${KERNEL_VERSION}.tar.gz ${KERNEL_VERSION}
```

### 保留不变

- `userpatches/config/boards/panther-x2.conf` — 板级配置（U-Boot 源、分区表、启动参数）
- `userpatches/kernel/*/dt/rk3566-panther-x2.dts` — 三套设备树（vendor 6.1 / current 6.18 / edge 7.1）

---

## amlogic-s9xxx-armbian 仓库改动详情

### 修改

| 文件 | 改动 |
|---|---|
| `build-armbian/armbian-files/common-files/etc/model_database.conf` L301 | Panther X2 的 `KERNEL_TAGS` 改为 `stable/6.18.y_rk35xx/6.1.y` |
| `.github/workflows/build-panther-x2-armbian.yml` | 新增（214 行），见下文详解 |

### build-panther-x2-armbian.yml 工作流

| 步骤 | 作用 |
|---|---|
| Checkout | 拉取仓库 |
| Initialize build environment | 安装依赖 (`curl -fsSL https://ophub.org/ubuntu2404-build-armbian-depends`) |
| Create virtual disk | 创建 XFS 虚拟磁盘挂载到 `/builder`，保证足够空间 |
| Map kernel branch → parameters | vendor→rk35xx/6.1.y, current→stable/6.18.y, edge→stable/7.1.y |
| Download base image | 从 ophub 上游 Release 用 GitHub API 下载 Armbian rootfs 镜像 |
| Fix model_database.conf | `sed` 把 Panther X2 的 KERNEL_TAGS 临时替换为单标签 |
| Rebuild Armbian | `sudo ./rebuild -b panther-x2 -r Lyric-c/panther-x2 -k {ver} -a true` |
| Upload | 上传 `.img.gz` 到 Release |

### 为什么需要 sed 修正 KERNEL_TAGS

`model_database.conf` 中 Panther X2 的 KERNEL_TAGS 是 `stable/6.18.y_rk35xx/6.1.y`。rebuild 的 `check_data` 会对所有 tag 都尝试下载内核。如果不修正：

- vendor 构建时，`stable/` 前缀被 `-u rk35xx` 错误替换为 `rk35xx/6.18.y`，导致尝试下载 `kernel_rk35xx/6.18.y.tar.gz`（404）
- 修正：`sed` 把 KERNEL_TAGS 临时改为单标签（vendor → `rk35xx/6.1.y`，current/edge → `stable/6.18.y`）

---

## 自定义内容清单

### 板级适配（来自原 panther-x2 项目）

| 文件 | 说明 |
|---|---|
| `panther-x2.conf` | BOARDFAMILY=rk35xx, U-Boot 使用 Radxa stable-4.19-rock3, 内核分区 GPT+SPL |
| `rk3566-panther-x2.dts` (3 份) | 设备树，vendor 版本含 `&mpp_srv` 硬件解码节点 |

### 内核打包适配（本次新增）

| 内容 | 说明 |
|---|---|
| deb→ophub 格式转换 | `dpkg -x` + 重命名 + 4 组件 tar.gz + sha256sums |
| KERNEL_NAME 规范 | `{major.minor.patch}-{sig}`, sig = rk35xx/stable |
| Release tag 映射 | vendor→kernel_rk35xx, current/edge→kernel_stable |

### 系统打包适配（本次新增）

| 内容 | 说明 |
|---|---|
| model_database.conf KERNEL_TAGS | Panther X2 支持 rk35xx + stable 双标签 |
| sed 临时修正 | 构建前根据 KERNEL_BRANCH 把 KERNEL_TAGS 缩小为单标签 |
| 基础镜像源 | 从 ophub 上游 Release 拉取（不再依赖 armbian.tnahosting.net） |
| Runner | ubuntu-24.04-arm |

---

## 如何增加额外的内核模块

### 场景 A：给现有内核增加模块（不改内核源码）

在 `panther-x2` 仓库的 `userpatches/` 目录放置内核补丁即可。armbian/build 框架会自动应用。

例如，增加 XFS 文件系统支持：

```
userpatches/kernel/rk35xx-vendor-6.1/
└── 999-enable-xfs.patch     ← 内核 .config 补丁
```

补丁内容示例：
```diff
--- a/arch/arm64/configs/rockchip_defconfig
+++ b/arch/arm64/configs/rockchip_defconfig
@@ -1234,6 +1234,7 @@
 CONFIG_EXT4_FS=y
+CONFIG_XFS_FS=m
```

重新触发 Build Kernel 即可。新模块会自动出现在 `modules-*.tar.gz` 中。

### 场景 B：新增 ko 驱动（不重新编译内核）

如果要增加的模块不在内核树中（如第三方 WiFi 驱动），可以：

1. 在 `panther-x2` 仓库创建 `userpatches/kernel/rk35xx-vendor-6.1/` 下的补丁或源码
2. 或者在 `amlogic-s9xxx-armbian` 仓库的 `common-files` 目录放置预编译的 `.ko` 文件

后者不需要重新编译内核。`common-files` 下的文件在 rebuild 的 `refactor_rootfs` 阶段自动覆盖到 rootfs：

```
amlogic-s9xxx-armbian/build-armbian/armbian-files/common-files/
└── lib/modules/
    └── 6.1.115-rk35xx/
        └── extra/
            └── mydriver.ko    ← 会被复制到 rootfs
```

> 注意：`common-files` 需要和 rootfs 目录结构一致，rebuild 回自动覆盖。

### 场景 C：修改内核配置

修改 `userpatches/config/boards/panther-x2.conf`，不直接修改内核 config。内核 config 通过 armbian/build 框架的内核配置扩展机制管理。如需永久改变默认配置，可在 armbian/build 的 `config/kernel/` 目录下创建对应文件。

---

## 故障排查与调试

### Build Kernel 失败

| 症状 | 原因 | 解决 |
|---|---|---|
| `requires root privileges` | 非 root 环境 | 确认 `PREFER_DOCKER=yes` |
| `No kernel deb found` | 编译失败 | 查看 armbian 构建日志 |
| tarball 内文件不对 | 打包脚本问题 | 检查 Prepare Kernel 步骤输出 |

### Build Armbian 失败

| 症状 | 原因 | 解决 |
|---|---|---|
| `armbian original file does not exist` | 基础镜像下载失败 | 确认 ophub Release 中有对应发行版的镜像 |
| `Missing kernel files for [...]` | tarball 缺少顶层目录或文件 | 检查 `6.1.115.tar.gz` 内是否有 `6.1.115/` 子目录 |
| `/boot files is missing` | boot 文件名不匹配 | 确认 boot 文件已重命名为 `{prefix}-{KERNEL_NAME}` |
| `/usr/lib/modules kernel folder is missing` | modules 目录名不匹配 | 确认 modules 目录已重命名为 KERNEL_NAME |
| `kernel_xxx/6.18.y.tar.gz` 404 | KERNEL_TAGS 多标签问题 | 确认 sed 步骤已正确执行 |

### 快速验证内核 tarball 结构

```bash
# 下载并查看结构
curl -LO https://github.com/Lyric-c/panther-x2/releases/download/kernel_rk35xx/6.1.115.tar.gz
tar -tzf 6.1.115.tar.gz | head -20
# 应该输出:
# 6.1.115/
# 6.1.115/boot-6.1.115-rk35xx.tar.gz
# 6.1.115/dtb-rockchip-6.1.115-rk35xx.tar.gz
# 6.1.115/modules-6.1.115-rk35xx.tar.gz
# 6.1.115/header-6.1.115-rk35xx.tar.gz
# 6.1.115/sha256sums

# 检查 boot 文件内容
tar -xzf 6.1.115.tar.gz
tar -tzf 6.1.115/boot-6.1.115-rk35xx.tar.gz
# 应该看到 vmlinuz-6.1.115-rk35xx, config-6.1.115-rk35xx 等
```

---

## 已解决的关键问题

### 启动问题排查历程

| # | 症状 | 根因 | 修复 |
|---|---|---|---|
| 1 | 电源灯不亮，完全无法启动 | boot 分区为 ext4，Panther X2 的 Radxa U-Boot 只支持 FAT32 | `different-files/panther-x2/rootfs/etc/armbian-board-release.conf` 写入 `bootfs_type="fat32"` |
| 2 | 电源灯不亮（修复 FAT32 后仍不行） | 内核 tarball 缺少 uInitrd 且 `ln -sf` 在 FAT32 上失败 | build-kernel.yml 用 `cp` 生成 Image/uInitrd 实文件副本 |
| 3 | 电源灯亮、蓝牙闪一下灭，无法获取 DHCP | minimal initramfs 的 `/init` 只挂 proc/sys/dev 后 `exec /sbin/init`，根文件系统未挂载导致 kernel panic | 改为 `sudo cp` 模块到 `/lib/modules` → `depmod` → `mkinitramfs` 生成含 rootfs mount + switch_root 的完整 initramfs（~21MB） |
| 4 | 系统启动成功但 Docker 无法运行 | vendor 内核默认未编译 netfilter 模块 | 新增 `999-docker-netfilter.config` 内核配置补丁 |

### 镜像构建/打包问题

| # | 症状 | 根因 | 修复 |
|---|---|---|---|
| 5 | rebuild 报告 `Missing kernel files for [6.1.115]` | 内核 tarball 内部文件平铺，缺少 `${KERNEL_VERSION}/` 顶层目录 | tar 前 `mkdir` + `mv` 文件到版本号子目录 |
| 6 | rebuild 报告 `/boot files is missing` | boot 文件名带 armbian 构建哈希 (S1c43-D398d-...)，rebuild 只认 `*{KERNEL_NAME}` 模式 | boot 文件全部 rename 为 `{prefix}-{KERNEL_NAME}` |
| 7 | rebuild 报告 `/usr/lib/modules kernel folder is missing` | modules 目录名带 armbian 哈希，rebuild 期望 `${KERNEL_NAME}` | modules 目录 rename 为 `${KERNEL_NAME}` |
| 8 | rebuild 下载 `kernel_rk35xx/6.18.y.tar.gz` 404 | model_database.conf KERNEL_TAGS 双标签导致 rebuild 对两个 tag 都下载内核 | workflow 中 `sed` 把 Panther X2 的 KERNEL_TAGS 临时替换为单标签 |
| 9 | base image 下载了 allwinner 镜像导致 rebuild `find_armbian` 失败 | ophub Release 中同时存在 `-board_` 和 `-trunk_` 两类镜像 | jq 过滤加 `select(.name \| contains("-trunk_"))` |
| 10 | rebuild 日志大量 `Cannot change ownership to uid 1001` | tar 保留原始 UID，rebuild 以 root 解压时 chown 失败 | 所有 `tar -czf` 加 `--owner=0 --group=0` |

---

## 自定义内核模块

### 网络 / Docker 模块 (`999-docker-netfilter.config`)

通过 `userpatches/kernel/rk35xx-vendor-6.1/999-docker-netfilter.config` 配置，armbian 构建框架自动 merge 到内核 `.config`。

| 分类 | 模块 | 模式 | 用途 |
|---|---|---|---|
| **nftables** | NF_TABLES, NFT_COUNTER/CT/COMPAT/LOG/LIMIT/MASQ/NAT/REJECT/SOCKET/TPROXY/FIB + 子项 | `=y` | nftables 防火墙框架 |
| **iptables** | IP_NF_IPTABLES/FILTER/NAT/MASQ/REDIRECT/MANGLE/RAW + IPv6 | `=y` | iptables 传统框架（与 nftables 可共存） |
| **conntrack** | NF_CONNTRACK + NAT helpers (FTP/SIP/TFTP) | `=y` | 连接跟踪，NAT 转发必需 |
| **bridge netfilter** | BRIDGE_NETFILTER, XT_MATCH_ADDRTYPE/CONNTRACK/IPVS | `=y` | Docker bridge 网络的包过滤 |
| **TPROXY** | XT_TARGET_TPROXY, XT_MATCH_SOCKET | `=y` | 透明代理（如 clash/v2ray） |
| **高级路由** | IP_ADVANCED_ROUTER, IP_MULTIPLE_TABLES, IP_ROUTE_MULTIPATH | `=y` | 策略路由、多路由表 |
| **Docker 基础** | OVERLAY_FS, VETH, BRIDGE | `=y` | Docker 镜像分层存储 + 虚拟网卡 + 网桥 |
| **单臂路由** | BRIDGE_VLAN_FILTERING | `=y` | 桥接 VLAN 过滤（`bridge vlan add`） |
| **通用网络** | MACVLAN, IPVLAN, VLAN_8021Q, IP_TUNNEL, VXLAN | `=m` | 按需加载：`modprobe macvlan` 等 |
| **BBR** | TCP_CONG_BBR | `=m` | TCP BBR 拥塞控制：`modprobe tcp_bbr` |
| **WireGuard** | WIREGUARD | `=m` | VPN：`modprobe wireguard` |
| **eBPF** | BPF, BPF_SYSCALL, BPF_JIT, CGROUP_BPF | `=y` | eBPF 程序运行环境（Cilium/可观测性等） |

> **`=y` vs `=m`**：`=y` 内建于内核镜像，始终可用无需手动加载；`=m` 编译为独立 .ko 模块，内核按需自动加载或 `modprobe` 手动加载。Docker 依赖的 netfilter 模块建议 `=y` 避免容器启动时加载失败。

---

## 日常维护

### 触发内核更新

1. 访问 [panther-x2 Actions](https://github.com/Lyric-c/panther-x2/actions)
2. 手动触发 **Build Kernel**，选择所需 BRANCH
3. 等待完成后，Release 中会出现新的 `{ver}.tar.gz`

### 触发系统打包

1. 确保 `kernel_rk35xx` Release 中有最新内核
2. 访问 [amlogic-s9xxx-armbian Actions](https://github.com/Lyric-c/amlogic-s9xxx-armbian/actions)
3. 手动触发 **Build Armbian for Panther X2**，选择 RELEASE 和 KERNEL_BRANCH

### 追踪上游内核更新

vendor 内核版本号来自 armbian 的 linux-rockchip 仓库：

```bash
curl -s https://raw.githubusercontent.com/armbian/linux-rockchip/rk-6.1-rkr5.1/Makefile \
  | grep '^SUBLEVEL =' | awk '{print $3}'
```

SUBLEVEL 变化即代表上游已更新，重新触发 Build Kernel 即可获得新内核。

---

## 相关仓库

| 仓库 | 用途 |
|---|---|
| [Lyric-c/panther-x2](https://github.com/Lyric-c/panther-x2) | Panther X2 内核编译（本项目） |
| [Lyric-c/amlogic-s9xxx-armbian](https://github.com/Lyric-c/amlogic-s9xxx-armbian) | Armbian 系统打包（fork） |
| [armbian/build](https://github.com/armbian/build) | Armbian 官方构建框架 |
| [ophub/amlogic-s9xxx-armbian](https://github.com/ophub/amlogic-s9xxx-armbian) | ophub 上游项目，rebuild 脚本来源 |
| [ophub/kernel](https://github.com/ophub/kernel) | ophub 内核格式参考 |
| [Zane-E/Armbian-Actions](https://github.com/Zane-E/Armbian-Actions) | 原始参考项目 |
