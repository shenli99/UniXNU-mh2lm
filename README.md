# UniXNU device repository — LG G8X ThinQ (mh2lm, SM8150)

UniXNU 设备仓库 + 一条 GitHub Actions workflow：在 macOS runner 上用 **Xcode 15** 编出
**xnu-8796** 内核（`mach.development.bcm2837.unstripped`）。

当前阶段：UEFI 固件（Project Aloha `lg-mh2lm`）+ UniXNU 菜单 + FV 里的 payload + DarwinBoot
已经在真机上跑通（`fastboot boot`，不写任何分区）；内核入口（M3）需要这里编出的内核。

## CI：编内核

1. 触发 `.github/workflows/kernel-8796.yml`（Actions → `kernel-8796` → Run workflow）。
   - 默认 pin `UniXNU` @ `40d21cc`，打 `ci/0019-unixnu-8796-ci.patch`（把 xnu 钉到 8796 + 移植补丁）。
   - 输入：`jobs`（默认 3，macos-14 标准 runner 是 3 核）、`unixnu_sha`。
2. 约 **1–2 小时**，产物 artifact `kernel-8796`：
   - `mach.development.bcm2837.unstripped`（4 KiB 页，BCM2837 machine config）
   - `kernel.log` + 各模块日志
3. `prelink` 那一步会**故意失败**（本仓库没配 `[kernel.prelink]`，而且 iOS 16.7.16 的 kernelcache 是
   `xnu-8796.142.1`、与源码 `141.3` 不同修订）。内核二进制在那之前已经生成，workflow 用 `|| true` + `test -f` 处理。

> **仓库必须是公开的**：macOS runner 在公开仓库上免费；私有仓库按 10× 计费，
> 免费额度 2000 min/月 → 实际只有 ~200 macOS 分钟，一次构建就可能超。

## 本地：payload / ADT（Linux，秒级）

内核以外的东西都不需要 macOS：

```sh
# 1) 拿内核（CI artifact）后，生成 ADT
python3 scripts/mkadt.py out.adt --device mh2lm --dram-size 0xc0000000

# 2) 组装 payload（B1：512 KiB FAT12，塞进固件卷；不走 boot.img ramdisk）
#    注意：`unixnu payload` 只会写整数 MiB 的 FAT32，造不出 512 KiB FAT12，
#    这一步目前是手工/脚本（mformat -T 1024 + mcopy），见 NOTES.md 的 B1 记录。
```

### base-d22ap.adt 不在仓库里

`mkadt.py` 需要 Apple 的设备树（`/product` 身份节点）来生成 `/product`。它是从 iOS 16.7.16 IPSW 抽出来的
Apple 产物，按惯例不入库。要恢复：

```sh
# IPSW: iPhone10,3,iPhone10,6_16.7.16_20H392_Restore.ipsw（需自备）
python3 scripts/adt.py unwrap Firmware/all_flash/DeviceTree.d22ap.im4p scripts/base-d22ap.adt
```

没有这个文件时 `mkadt.py` 会打印 `no /product node` 警告，ADT 里缺身份节点（内核构建不受影响，payload 会受影响）。

## 不放进仓库的东西

| 内容 | 原因 |
| --- | --- |
| IPSW / kernelcache / DeviceTree 原文件 | Apple 产物，体积大，从 Apple 官方下载 |
| `scripts/base-d22ap.adt` | 同上，按 IP SE 抽取（见上） |
| 任何设备分区镜像 / 备份 | 体积 + 隐私 |
| `.cache/`、`__pycache__` | 构建缓存 |

## 组件与补丁

| 东西 | 说明 |
| --- | --- |
| `board.toml` | 设备描述：`[ipsw]`（iPhone10,3 / d22ap / 16.7.16 / 20H392）、`[kernel]` defconfig + 归档构建器、`[payload]`、`[bootimg.image]` |
| `configs/mh2lm_defconfig` | 内核开关（corecrypto / pthread / DT 平台 / ARMCPU / 归档构建器） |
| `scripts/` | ADT 工具：`devices.py`（板级值）、`mkadt.py`（生成 ADT）、`adt.py`（解析/解封）、`make-mh2lm-adt.py`（UniXNU 调用入口） |
| `ci/0019-unixnu-8796-ci.patch` | 对 UniXNU 的改动（内核相关部分）：`feeds.toml` 钉 8796、`patches/xnu/*`、`scripts/unixnu/ipsw.py` |
| `NOTES.md` | 开发笔记（沿革、踩坑、B1 payload 方案），链接指向开发工作区，公开仓库里不完整 |
| `AGENTS.md` | 给后续维护者/agent 的边界与硬规则（只 fastboot boot、Apple 产物不入库、版本三角约束） |
| `DEVIATIONS.md` | **为什么本仓库是 format 1**：format 2 的三个前提对 iOS 16 都不成立（附实测内核版本表） |
| `proprietary-files.txt` | 板级固件输入清单（路径/来源/sha256/消费者）；Apple 产物不入库，由工具抽取校验 |
| `local.toml.example` | 模块本地覆盖示例；构建输入在 UniXNU 检出的 `unixnu.local.toml` |

## 已知边界

- 这条 workflow 产出的是 **UniXNU 主线风格 4 KiB 内核**（无内置 HFS/AMFI、无 prelink）：
  够拿 **M3**（内核入口/打印/平台专家匹配 ADT），**到不了 iOS 16 用户态**（那需要 16 KiB 单体内核）。
- iOS 16 用户态需要另一条内核线（单体内核：内置 HFS/AMFI/UFS，iPhone X 身份），目前还在开发工作区里推进。
