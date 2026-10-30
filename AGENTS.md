# Device agent guide — LG G8X ThinQ (mh2lm)

本仓库是 **format 1** 的 UniXNU 设备仓库（没有 `schema_version = 2`，没有 `porting.toml`）。
选择 format 1 的原因与代价写在 [DEVIATIONS.md](DEVIATIONS.md)；改路线前先读它。

## 本仓库拥有什么

| 东西 | 位置 |
| --- | --- |
| 设备描述（IPSW 身份、内核 defconfig/归档、payload、bootimg 偏移） | `board.toml` |
| 内核开关 | `configs/mh2lm_defconfig` |
| ADT 生成（板级值 + 设备树构建） | `scripts/devices.py`、`scripts/mkadt.py`、`scripts/make-mh2lm-adt.py`、`scripts/adt.py` |
| 内核构建 CI（macOS + Xcode 15） | `.github/workflows/kernel-8796.yml` |
| 对 UniXNU 的改动（补丁） | `ci/0019-unixnu-8796-ci.patch` |
| 固件输入清单 | `proprietary-files.txt` |

## 先读这些

| 任务 | 必读 |
| --- | --- |
| 任何改动 | [DEVIATIONS.md](DEVIATIONS.md)（为什么是 format 1）、[README](README.md)、`ci/README.md` |
| 内核/补丁 | [UniXNU: source and patches](https://unixnu.com/xnu/source-and-patches)、`ci/README.md` |
| 启动链/镜像 | [Payload](https://unixnu.com/concepts/payload)、[UEFI firmware](https://unixnu.com/porting/uefi-firmware)、[bring-up](https://unixnu.com/porting/bring-up) |
| 硬件事实 | 开发工作区的 `docs/notes/10-device-mh2lm.md`（分区、面板、触控、Wi-Fi） |

## 硬规则

1. **只 `fastboot boot`，不写任何分区**（不 `flash`/`erase`/`format`；写 userdata 属于需要单独批准的动作）。
2. **Apple 产物不入库**：IPSW、kernelcache、DeviceTree、`scripts/base-d22ap.adt`。它们由工具在本地抽取/校验。
3. **不要改 `mkadt.py` 产出的节点语义去"绕过"内核检查**：`/chosen` 必须带 CHRP NVRAM proxy，`/options` 必须存在
   （`ArmDTPlatform::configure()` 缺任一都拒绝启动）。
4. 改内核补丁后，**同一提交里**更新 `ci/0019-unixnu-8796-ci.patch`，否则 CI 与本地树不一致。
5. 版本三角是硬约束：iOS 16 用户态需要 **16 KiB 单体内核**；不要试图用"源码内核 + 原版 kernelcache prelink"
   去配 iOS 16（`DEVIATIONS.md` 有实测表）。

## 常用命令

```sh
# 本地：选配置、检查解析
python3 unixnu defconfig mh2lm_defconfig -D ../UniXNU-mh2lm
python3 unixnu status -D ../UniXNU-mh2lm

# 本地：生成 ADT（需要 scripts/base-d22ap.adt 才有 /product 节点）
python3 scripts/mkadt.py out.adt --device mh2lm --dram-size 0xc0000000

# 内核：不在本地编，跑 CI（Actions → kernel-8796）；产物是 artifact kernel-8796
```

## 记录方式

format 1 没有 `porting.toml` 和记录检查。证据仍要留：

- 构建/刷写结果写进开发工作区的 `evidence/<任务>/`；
- 本仓库里只保留**可复现的输入**（board/defconfig/scripts/CI patch）和结论性说明（README/DEVIATIONS）。
