# chiron rootfs（Server 路线）

Xiaomi MIX 2（chiron / MSM8998）跑主线内核时，用户空间需要的那一层**定制**。
系统本体是 Ubuntu Server arm64；这里放的是"在这个系统上改了什么"，不是整份 rootfs 镜像。

## 仓库结构

| 路径 | 内容 |
|---|---|
| `overlays/` | **设备上真正在跑的那份定制层**：58 个普通文件 + `MANIFEST.sha256`（逐文件 sha256）+ **12 个符号链接**（`multi-user.target.wants/chiron-*.service` 这类启用链接、以及键盘布局别名）。全部保留设备上的真实绝对路径（去掉前导 `/`）：`overlays/etc/chiron/greeter.conf` → `/etc/chiron/greeter.conf` |
| `package-selections.txt` | 设备上 `dpkg --get-selections` 快照（684 个包） |
| `firmware-manifest.txt` | `/lib/firmware/qcom/msm8998` 下 49 个文件的 sha256 与大小（固件二进制本身在 kernel 仓库的 `firmware` 分支） |

`overlays/` 覆盖的内容：

| 目录 | 内容 |
|---|---|
| `etc/chiron/` | greeter 配置、键盘布局、GTK 主题 |
| `etc/systemd/system/` | `chiron-*` 服务/drop-in：亮度持久化、zram 重试、RTC、SSH 主机密钥、journald、logind、内核模块加载 |
| `usr/libexec/`、`usr/local/{bin,sbin,libexec}/` | `chiron-*` 脚本：屏幕、传感器、Wayland、RTC、充电限流、USB 角色、Wi-Fi |
| `etc/udev/rules.d/`、`etc/sysctl.d/`、`etc/sudoers.d/`、`etc/modules-load.d/` | 设备与内核参数的落地配置 |
| `etc/netplan/`、`etc/NetworkManager/conf.d/` | 网络（不含任何 Wi-Fi 口令：连接配置在 `/etc/NetworkManager/system-connections/`，不在本仓库） |
| `etc/ssh/sshd_config.d/` | SSH 登录策略（见文末"已知安全缺口"） |

## 怎么套回去

```bash
# 只读比对（先看会改什么）
rsync -avin overlays/ / | head -60

# 真正落盘
rsync -a overlays/ /
```

**三条纪律**（2026-11-06 用 `tar -C /` 覆盖过 `/lib` 符号链接，把设备搞成起不来，
见 [docs/INCIDENT-2026-11-06-usrmerge.md](https://github.com/zzh-zzh-zzh/chiron-kernel-mainline)）：

1. 任何归档都不要 `tar -C /` 解 —— `lib/` 是 `usr/lib` 的符号链接，`tar` 会把符号链接换成真目录；
2. 归档内一律用 `usr/lib/...` 而不是 `lib/...`；
3. 先 `rsync -n`（dry-run）再落盘，落盘前后各跑一次 `ls -ld /lib /bin /sbin` 确认还是符号链接。

`overlays/` 里没有 `lib/systemd/system/chiron-rtc-*.service`：usrmerge 下它与
`usr/lib/systemd/system/` 是同一份文件，按 `usr/` 路径归档，避免解包时重复写同一 inode。

那 12 个符号链接不是可有可无的装饰：`multi-user.target.wants/` 下的 11 条就是"这些服务被 enable 了"的唯一记录
（`chiron-rtc-restore` 在 `sysinit.target.wants/`），两个键盘 yaml 是别名。**用会丢符号链接的方式复制
（例如先 `tar` 再解到别处、或用不支持链接的同步工具），服务会变成未启用状态。**

## 不在这个仓库里的东西

| 东西 | 在哪 |
|---|---|
| 内核源码与配置 | kernel 仓库 `chiron-7.2.10` 分支（`arch/arm64/configs/chiron_defconfig` + `CHIRON.md`） |
| 内核模块 | 由上面那棵树编出，装到 `/usr/lib/modules/<kernel.release>/`（166 个 `.ko`） |
| 固件二进制（208 个文件 / 70 MiB） | kernel 仓库 `firmware` 分支，带 README 说明每个文件谁加载、干什么 |
| 旧构建脚本 2,529 个（`workspace/`） | **故意不公开**：其中 89 个文件写着设备的默认口令，大量脚本写死迁移前的 Windows 路径；它们不是 rootfs 内容 |
| 镜像本体（`userdata.img` / `boot.img`） | 不进 git（`.gitignore` 已排除） |

## 已知安全缺口（未修）

`overlays/etc/ssh/sshd_config.d/10-chiron.conf` 就是：

```
PermitRootLogin yes
PasswordAuthentication yes
```

配合 root 的默认弱口令 —— 这是 `docs/ISSUES.md` 里的 I-14，**尚未修复**。本仓库如实保留设备现状；
把这套东西部署到不可信网络之前，请先写 `authorized_keys`、改成
`PermitRootLogin prohibit-password` + `PasswordAuthentication no`，再考虑防火墙规则。
