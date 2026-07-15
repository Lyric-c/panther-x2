# Panther X2 — Armbian 构建

基于 [armbian/build](https://github.com/armbian/build) 官方框架 + [ophub/amlogic-s9xxx-armbian](https://github.com/ophub/amlogic-s9xxx-armbian) 打包工具链，为 Panther X2 (RK3566) 编译内核并构建 Armbian 系统。

## 两个仓库分工

| 仓库 | 定位 | Actions |
|---|---|---|
| **[panther-x2](https://github.com/Lyric-c/panther-x2)** ← 你在这里 | 内核编译 → 上传 Release | **Build Kernel** |
| **[amlogic-s9xxx-armbian](https://github.com/Lyric-c/amlogic-s9xxx-armbian)** | 系统打包 → 上传 Release | **Build Armbian for Panther X2** |

> 📖 完整架构说明见 [BUILD_ARCHITECTURE.md](./BUILD_ARCHITECTURE.md)

## 快速开始

### 编译内核

1. 在 Actions 页面触发 **Build Kernel**
2. 选择内核分支：`vendor`（硬件解码） / `current` / `edge`
3. 内核将上传到 Release（tag: `kernel_rk35xx` 或 `kernel_stable`）

### 打包 Armbian 系统

1. 确保 `kernel_rk35xx` 或 `kernel_stable` Release 中有可用的内核
2. 去 [amlogic-s9xxx-armbian Actions](https://github.com/Lyric-c/amlogic-s9xxx-armbian/actions) 触发 **Build Armbian for Panther X2**
3. 选择发行版（trixie/bookworm/noble/jammy）和内核分支

## 内核分支说明

| 分支 | 内核版本 | 内核源码 | 硬件解码 |
|---|---|---|---|
| `vendor` | 6.1.y (Rockchip BSP) | `armbian/linux-rockchip` `rk-6.1-rkr5.1` | ✅ MPP |
| `current` | 6.18.y (mainline LTS) | armbian 默认 linux-rockchip | ❌ |
| `edge` | 7.1.y+ (mainline) | armbian 默认 linux-rockchip | ❌ |

## 目录结构

```
panther-x2/
├── BUILD_ARCHITECTURE.md         ← 架构文档
├── .github/workflows/
│   └── build-kernel.yml          # 编译内核 + 打包为 ophub 格式
├── userpatches/
│   ├── config/boards/
│   │   └── panther-x2.conf       # 板级配置
│   └── kernel/
│       ├── rockchip64-6.18/dt/
│       ├── rockchip64-7.1/dt/
│       └── rk35xx-vendor-6.1/dt/
│           └── rk3566-panther-x2.dts
└── README.md
```

## 追踪内核更新

```bash
curl -s https://raw.githubusercontent.com/armbian/linux-rockchip/rk-6.1-rkr5.1/Makefile \
  | grep '^SUBLEVEL =' | awk '{print $3}'
```
