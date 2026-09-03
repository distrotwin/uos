# 统信 UOS 桌面操作系统 · 构建与测试镜像

与真实统信 UOS 桌面专业版尽可能一致的容器环境，用于**软件构建、打包与兼容性测试**。两个大版本、三个档位、跨三种指令集，全部从统信官方 ISO 的 squashfs 切片而来，公开在 GHCR。

```bash
docker run --rm ghcr.io/distrotwin/uos:v25-devel \
  bash -c 'grep -E "^(PRETTY_NAME|VERSION_CODENAME)=" /etc/os-release; ldd --version | head -1; gcc -dumpfullversion'
```

```
PRETTY_NAME="UOS Desktop 25 Professional"
VERSION_CODENAME=snipe
ldd (Debian GLIBC 2.38-6deepin19) 2.38
12.3.0
```

## 这是什么，不是什么

镜像里没有内核，systemd 只保留容器内有意义的部分。真机上依赖内核特性、安全模块或硬件的行为，在这里不成立。

所以它适合回答「编出来的东西对不对」，不适合回答「跑起来的系统对不对」。

**该用它**：在 CI 里编出能在统信上跑的 C 二进制；检查产物需要的 glibc 符号版本目标系统能否满足；复现只在统信上出现的编译问题；验证已有产物能不能在目标系统上起来。

**别用它**：编 C++（见下）；当生产运行时基础镜像；复现内核相关的行为；当作统信系统的完整替代品做验收测试。

> **最要紧的一条：这两个版本都编不了 C++。** 统信桌面的安装介质里**没有 `g++`、没有 `libstdc++-dev`**，而它的在线源只提供应用商店的 GUI 应用、不含任何 OS 包，所以补也补不上。这不是切片切漏了，是介质里就没有。要编 C++ 请改用[银河麒麟镜像](https://github.com/distrotwin/kylin)或 manylinux 交叉验证。

## 先跑一遍

进容器，写个 A+B，编了跑。

```bash
docker run -it --rm ghcr.io/distrotwin/uos:v25-devel /bin/bash
```

进去以后：

```bash
echo '#include <stdio.h>
int main(void){ int a, b; if (scanf("%d %d", &a, &b) != 2) return 1; printf("%d\n", a + b); return 0; }' > ab.c

gcc -O2 -o ab ab.c
echo "3 4" | ./ab
objdump -T ab | grep -oE 'GLIBC_[0-9.]+' | sort -uV | tail -1
```

```
7
GLIBC_2.34
```

同一份代码在 `v20-devel` 里也输出 `7`，但符号底线是 `GLIBC_2.2.5`——**在 V20 上编的产物能送到 V25，反过来不行。**

## 选哪一个

```
ghcr.io/distrotwin/uos:<版本>-<档位>
```

| 版本 | 系统 | 代号 | glibc | GCC | 上游血脉 | 架构 |
|---|---|---|---|---|---|---|
| `v25` | UOS 桌面专业版 V25（2500u1） | `snipe` | 2.38 | 12.3 | OSTree 不可变 | amd64 · arm64 · **loong64** |
| `v20` | UOS 桌面专业版 V20（1070） | `eagle` | 2.28 | 8.3 | Debian 10 | amd64 · arm64 |

**V20 线内的 1010→1070 是点版本关系，不是换代。** 实测依据：1043（2022-01）与 1070（2024-04）相隔两年零四个月，卷标同为 `UOS 20`、代号同为 `Eagle`，装机清单里 `libc6` 都是 2.28.x、`gcc` 都是 8.3.0、`dpkg` 都是 1.19.7，只有厂商修订号在动。所以只做末版 1070。V20 → V25 才是换代：代号 `eagle`→`snipe`、glibc 2.28→2.38，形态也从传统 deb 变成 OSTree 不可变。

| 档位 | 装了什么 | 什么时候用 |
|---|---|---|
| `micro` | libc、libstdc++、CA 证书、时区、locale | 跑已经编好的二进制，验证它在目标系统上能不能起来 |
| `base` | 加常用命令行工具、python3、systemd、apt | 平台脚本、集成测试 |
| `devel` | 加 `gcc`、`make`、`binutils`、`libc6-dev`、`dpkg-dev` | **编译与打包（仅 C）** |

不带档位后缀的 tag 指向 `devel`；`latest` 是最高版本的 `devel`，也就是 `v25-devel`。

```bash
docker pull ghcr.io/distrotwin/uos:v25-devel
docker pull ghcr.io/distrotwin/uos:v25          # 同上
docker pull ghcr.io/distrotwin/uos:v20-base
docker pull ghcr.io/distrotwin/uos:v20-micro
docker pull ghcr.io/distrotwin/uos:latest       # = v25-devel
```

## 怎么用

**编一个程序，并确认它的 ABI 需求**

```bash
docker run --rm -v "$PWD:/w" -w /w ghcr.io/distrotwin/uos:v20-devel bash -c '
  gcc -O2 -o app app.c
  objdump -T app | grep -oE "GLIBC_[0-9.]+" | sort -uV | tail -1
'
```

最后那行是产物需要的最高 glibc 符号版本。它不高于目标机器的 glibc，产物才跑得起来。

**两代同时验**

```bash
for v in v20 v25; do
  printf '%-4s ' "$v"
  docker run --rm -v "$PWD:/w" -w /w "ghcr.io/distrotwin/uos:$v-devel" \
    bash -c 'gcc -O2 -o /tmp/app app.c 2>/dev/null && echo 通过' || echo 失败
done
```

**在 GitHub Actions 里当构建环境**

```yaml
jobs:
  build-for-uos:
    runs-on: ubuntu-latest
    container: ghcr.io/distrotwin/uos:v20-devel-20260903   # 钉住带日期的 tag
    steps:
      - uses: actions/checkout@v4
      - run: make
```

**验证一个已有产物能不能在目标系统上起来**（用 `micro`，最接近装完系统什么都没多装的状态）

```bash
docker run --rm -v "$PWD:/w" -w /w ghcr.io/distrotwin/uos:v20-micro \
  bash -c 'ldd ./app; ./app --version'
```

**跑 loong64**（需要宿主注册 binfmt，见下方「本地构建」）

```bash
docker run --rm --platform linux/loong64 ghcr.io/distrotwin/uos:v25-devel \
  bash -c 'echo $MACHTYPE; gcc -dumpmachine'
```

## 认出自己在哪个系统上

```bash
docker run --rm ghcr.io/distrotwin/uos:v25-base cat /etc/os-release
```

```
PRETTY_NAME="UOS Desktop 25 Professional"
NAME="Uos"
VERSION_ID="25"
VERSION="25"
ID=uos
HOME_URL="https://www.chinauos.com/"
BUG_REPORT_URL="http://bbs.chinauos.com"
VERSION_CODENAME=snipe
```

V20：

```
PRETTY_NAME="UOS Desktop 20 Professional"
NAME="uos"
VERSION_ID="20"
VERSION="20"
ID=uos
HOME_URL="https://www.chinauos.com/"
BUG_REPORT_URL="http://bbs.chinauos.com"
VERSION_CODENAME=eagle
```

脚本里判断平台时有三处要留意。

**`NAME` 的大小写两版不一致**：V20 是 `"uos"`、V25 是 `"Uos"`。按 `NAME` 匹配会踩，用 `ID=uos`——两版一致且规范要求小写。

**没有 `ID_LIKE`**，两版都不声明，`case "$ID_LIKE" in *debian*)` 一律掉进 default 分支。判族系用 `command -v dpkg`。

**`VERSION_CODENAME` 才是底座的身份证**：`eagle` 是 Debian 10 那一代，`snipe` 是 V25 的新底座。V20 线内不管 1010 还是 1070，代号都是 `eagle`——这正是它们同属一代的证据。

另外两版都**没有中文系统名**，这一点与银河麒麟不同；容器里 `uname` 报的是宿主内核，跟统信无关。

## tag 与钉版

| tag | 性质 |
|---|---|
| `v25-devel` | 滚动，跟随最新构建 |
| `v25` | 滚动，= `v25-devel` |
| `latest` | 滚动，= 最高版本的 `devel` |
| `v25-devel-20260903` | 按 commit 日期，约定不动 |
| `v25-devel-20260903-amd64` | 单架构，manifest list 的成员 |

**CI 里请用带日期的那个。** 滚动 tag 会随重建变化，`latest` 还会随大版本推进跨越 ABI 边界，把构建环境从 glibc 2.28 换成 2.38。

日期取的是发布时仓库 HEAD 的 commit 日期（UTC），不是构建时刻。**要钉死请用 digest**——GHCR 的 tag 本来就可变。

```bash
docker buildx imagetools inspect ghcr.io/distrotwin/uos:v25-devel --format '{{.Manifest.Digest}}'
docker pull ghcr.io/distrotwin/uos@sha256:<digest>
```

## 镜像自带的溯源信息

```bash
docker inspect --format '{{json .Config.Labels}}' \
  ghcr.io/distrotwin/uos:v25-devel | python3 -m json.tool
```

| label | 内容 |
|---|---|
| `cn.internal.glibc` / `libstdcpp` / `glibcxx` | **实测**的 ABI 值，来自测试阶段在干净机器上跑出来的结果 |
| `cn.internal.arch` / `tier` / `build-method` | 架构、档位、走的哪条构建路径（这里一律是 `slice`） |
| `cn.internal.repo-commit` / `buildkit-commit` | 哪份配置与哪份脚本建的 |
| `cn.internal.iso-url` / `squashfs-sha256` | 切自哪张盘的哪个 squashfs——ISO 路径的溯源锚点 |
| `cn.internal.build-run` | 跳回当时的 CI 日志与测试报告 |

## 架构与 ABI 分叉

amd64 与 arm64 由原生 runner 构建；V25 另有 loong64（QEMU 模拟构建与测试）。

**只有 V25 有 LoongArch 镜像，而且是新世界。** 统信为 V25 出的是 `loong64`（新世界 ABI），上游 QEMU 支持，实测能在模拟下完整跑通检查集。V20 出的是 `loongarch64`（旧世界），上游 QEMU 没有实现它需要的系统调用；V20 另有 `sw_64`（申威），QEMU 没有这个后端、GitHub 也没有这种 runner。这两个**建得出但测不了**，按「测过才发」的口径不列入，而不是留一个永远红着的 job。

`loong64` 与 `loongarch64` 是两个不通用的 ABI 世界，命名一字之差，别混。

## 已知的怪癖

**编不了 C++，两个版本都是。** 安装介质里没有 `g++` 与 `libstdc++-dev`，在线源里也没有——它只提供应用商店的 GUI 应用。`cmake`、`git`、`gdb`、`strace`、`python3` 头文件同样是硬缺口。这是介质与分发方式决定的，不是切片漏了。

**装不上 OS 包，这是产品设计不是缺陷。** `apt` 二进制在、源可达、`apt check` 干净，但源索引里只有应用商店条目，连 `nano` 都查不到候选。统信的 OS 分发走 OSTree 加玲珑，不走 apt。镜像忠实反映了这一点，验收里把它声明成期望而非失败。

**V25 的 `dpkg`/`apt` 被指回了 `.real` 真二进制。** 它出厂时把真二进制改名成 `*.real`、用 `deepin-immutable-ctl` 适配器顶替；容器里没有 OSTree 部署，适配器必然失败。切片时指回真二进制，这是与真机的**有意偏差**，好处是 `dpkg` 查询与本地装包可用。V20 不是不可变系统，没有这一层。

## 镜像是怎么造的

统信桌面**没有可 bootstrap 的 apt 源**，所以两个版本都走切片：HTTP Range 从官方 ISO 里只抽 `live/filesystem.squashfs`（不下整盘）→ 按 sha256 校验 → `unsquashfs` → 按包依赖闭包切片 → 补回 postinst 生成物（`ld.so.cache`、locale 归档、CA 信任库、`update-alternatives` 链接）。

盘内虽有 `dists/eagle`，但那是安装器的 overlay 仓库（`Origin: isobuild`，`Packages` 只有 123 KB），不足以自举；系统本体在 squashfs 里。厂商的容器镜像仓库 `registry.uniontech.com` 只有服务器版，桌面版一个都没有。

每个镜像发布前都在**干净机器上**装载、真正启动、跑完整检查集。最近一轮 15 个镜像、619 项检查、**零异常**。测试报告与完整日志按系统打包在每次 run 的 artifact 里。

## 本地构建

```bash
git clone --recurse-submodules https://github.com/distrotwin/uos.git
cd uos
sudo apt-get install -y squashfs-tools zstd curl python3

# 抽 squashfs（约 3 GiB，按 conf 里的 sha256 校验）并解包
sudo ARCH=amd64 ROOT=$PWD BK=$PWD/buildkit buildkit/tools/prepare-slice-src.sh v25
# 切出三个档位并导入
sudo ARCH=amd64 ROOT=$PWD BK=$PWD/buildkit buildkit/build/build.sh  v25 micro base devel
sudo ARCH=amd64 ROOT=$PWD BK=$PWD/buildkit buildkit/build/import.sh v25 micro base devel
sudo ARCH=amd64 ROOT=$PWD BK=$PWD/buildkit buildkit/test/verify.sh  v25
```

`unsquashfs` 需要 root 才能保住属主。跨架构切片还要宿主装 `qemu-user-static` 与 `binfmt-support`——切片要 `chroot` 进目标 rootfs 跑 `ldconfig` 与 `localedef`。LoongArch 的用户态模拟是 QEMU 7.1 才加的，Ubuntu 22.04 只带 6.2。不要引入 `tonistiigi/binfmt` 容器，实测它会破坏本来可用的 binfmt 注册。

抽盘是最慢的一段：这个 CDN 实测约 1 MB/s，3 GiB 要一到两小时。抽取器按字节续传，中断后重跑会从断点接上。

## CI

构建只允许手动触发（`workflow_dispatch`）。三个开关：

- `include-loongarch`：是否一并构建 V25 的 loong64（默认关，需 QEMU 模拟）
- `publish`：测试全绿后是否发布到 GHCR（默认关）
- `harvest`：只采 squashfs 指纹不构建（`SQUASHFS_SHA256` 为空时用一次）

构建按「版本 × 架构」切分，一个 job 出三个档位——切片的抽盘加解包是一次昂贵的共享前置。**测试仍是一镜像一 job、干净机器装载。**

## 仓库结构

构建机器码在 [`distrotwin/buildkit`](https://github.com/distrotwin/buildkit)，本仓库以 submodule 引用并钉住 commit。

```
distros/v20.conf  v25.conf        # 源 ISO、切片种子、ABI 基线、期望声明
.github/workflows/build.yml       # 调用 buildkit 的可复用 workflow
buildkit/                         # submodule
```
