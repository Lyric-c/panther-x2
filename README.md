# Panther X2 — Armbian 纯上游构建

基于 [armbian/build](https://github.com/armbian/build) 官方框架，通过 `userpatches` 机制适配 Panther X2 (RK3566)。

---

## 快速开始

1. **Fork** 本仓库
2. 在 Actions 页面触发 **Build Armbian** 或 **Build Kernel** 工作流
3. 等待构建完成，在 Releases 页面下载产物

---

## 工作流

### Build Armbian — 打包完整固件

| 参数 | 说明 | 可选值 |
|---|---|---|
| `BRANCH` | 内核分支 | `vendor`（硬件解码） / `current` / `edge` |
| `RELEASE` | 系统发行版 | `trixie` / `bookworm` / `noble` / `jammy` |
| `BUILD_MINIMAL` | 最小化系统 | `yes` / `no` |
| `BUILD_DESKTOP` | 包含桌面 | `yes` / `no` |

产物在 Release `Armbian-{release}-{YYYY.MM}` 中。

### Build Kernel — 单独编译内核

| 参数 | 说明 | 可选值 |
|---|---|---|
| `BRANCH` | 内核分支 | `vendor` / `current` / `edge` |

产物在 Release `Kernel-{linux_family}` 中，包含 `linux-image` `linux-headers` `linux-dtb` `linux-libc-dev` 四个 deb 包。

---

## 内核分支说明

| 分支 | 内核版本 | 内核源码 | 硬件解码 |
|---|---|---|---|
| `vendor` | 6.1.y (Rockchip BSP) | `armbian/linux-rockchip` `rk-6.1-rkr5.1` | ✅ 完整 MPP |
| `current` | 6.18.y (mainline LTS) | armbian 默认 linux-rockchip | ❌ 仅 V4L2 |
| `edge` | 7.1.y+ (mainline) | armbian 默认 linux-rockchip | ❌ 仅 V4L2 |

> 需要 Jellyfin 硬件转码请选 `vendor`。

---

## 目录结构

```
fortest/
├── .github/workflows/
│   ├── build-armbian.yml    # 打包完整固件
│   └── build-kernel.yml      # 单独编译内核
├── userpatches/
│   ├── config/boards/
│   │   └── panther-x2.conf   # 板级配置
│   └── kernel/
│       ├── rockchip64-6.18/dt/
│       │   └── rk3566-panther-x2.dts
│       ├── rockchip64-7.1/dt/
│       │   └── rk3566-panther-x2.dts
│       └── rk35xx-vendor-6.1/dt/
│           └── rk3566-panther-x2.dts
└── README.md
```

只需维护 **4 个文件**即可完成适配，无需修改 armbian/build 的任何源码。

---

## 追踪内核更新

上游 vendor 内核版本号来自 Makefile：

```bash
curl -s https://raw.githubusercontent.com/armbian/linux-rockchip/rk-6.1-rkr5.1/Makefile \
  | grep '^SUBLEVEL =' | awk '{print $3}'
```

SUBLEVEL 变化即代表上游已更新，重新触发 Build Armbian / Build Kernel 即可获得新内核。
