# 开发指引

统信 UOS 桌面专业版 V20 / V25 的**构建与测试镜像**。定位是「与真实桌面系统尽可能一致的环境，用于软件构建与测试」，不是服务器镜像，不面向生产部署。动手前先读 README 开头两节。

建仓规范见 [`buildkit/docs/downstream-repo.md`](buildkit/docs/downstream-repo.md)，本文件只讲这个仓库特有的部分。UOS 与麒麟差别很大，照搬 `distrotwin/kylin` 的做法会踩坑。

## 硬性约定

- commit **不允许带 co-author**
- 文档一律中文；Markdown **自然段内不换行**，一段写成一行长句
- 不在仓库里讨论许可与法务
- 写进文档的版本号一律来自跑镜像实测，不取源索引或包清单里的元包版本

## 这个仓库放什么

```
distros/{v20,v25}.conf         # 源、ISO、切片种子、ABI 基线
.github/workflows/build.yml    # 只负责调用 buildkit 的可复用 workflow
buildkit/                      # submodule → distrotwin/buildkit
```

只跟某个版本自己的事实有关就进 conf，跟怎么构建、怎么测、怎么发有关就进 buildkit。

## 与麒麟最大的不同：只能切片

麒麟四个版本都从 apt 归档 bootstrap。UOS **没有这条路**：

- V25 是 OSTree 不可变系统，`apt` 与 `dpkg` 被 `deepin-immutable-ctl` 接管
- 在线源不是打不开，是**通了也没用**——只有应用商店的 GUI 应用，一个 OS 包都没有
- 盘内虽有 `dists/eagle`，但那是安装器的 overlay 仓库（`Origin: isobuild`，V20 1070 的 `Packages` 只有 123 KB），不足以 bootstrap

唯一路径是 `METHOD=slice`：HTTP Range 抽 `live/filesystem.squashfs` → 校验 sha256 → `unsquashfs`（须 root）→ 按包依赖闭包切片。

## 版本怎么选的

**V20 线内的 1010→1070 是点版本关系，不是换代。** 实测依据：1043（2022-01）与 1070（2024-04）相隔两年零四个月，卷标同为 `UOS 20`、`.disk/info` 里 codename 同为 `Eagle`，装机清单里 `libc6` 都是 2.28.x、`gcc` 都是 8.3.0、`dpkg` 都是 1.19.7、`systemd` 都是 241，只有厂商修订号在动（`-1+dde` → `-deepin1`）。所以只做末版 1070，多做一版只是多花一份几个 GiB 的抽盘成本，换不来一个新的 ABI 档位。

**V20 → V25 才是换代**：codename `Eagle`→`Snipe`、卷标 `UOS 20`→`uos`、glibc 2.28→2.38，而且形态从传统 deb 变成 OSTree 不可变。

要重新核这类判断，用 `buildkit/tools/iso9660.py` 直读远程 ISO，几十 KB 就够，不必下整盘：

```bash
python3 - <<'PY'
import sys; sys.path.insert(0,'buildkit/tools')
from iso9660 import ISO
iso = ISO("<ISO 直链>")
print(iso.volid)
e = iso.find(".disk/info");            print(iso.cat(e).decode())
e = iso.find("live/filesystem.manifest")  # 全量装机包清单，几十 KB
PY
```

## 架构为什么是这几个

| 架构 | V20 | V25 | 做不做 |
|---|---|---|---|
| amd64 / arm64 | ✔ | ✔ | 做，原生 runner |
| `loong64`（新世界） | ✘ 无盘 | ✔ | 做（默认关，需 QEMU） |
| `loongarch64`（旧世界） | ✔ 有盘 | ✘ 无盘 | **不做** |
| `sw_64`（申威） | ✔ 有盘 | ✘ 无盘 | **不做** |

后两个建得出但**测不了**：上游 QEMU 没有 sw_64 后端，旧世界 loong 也跑不了（麒麟 V10 SP1 那边实测是读出 ELF 头后报 `Unknown syscall 80` 退 127）。按「测过才发」的口径，宁可明确不列入，也不留一个永远红着的 job。

`loong64` 与 `loongarch64` 是两个不通用的 ABI 世界，命名一字之差，别混。

## 必知事实

**ISO 站给存在性信号，apt 站不给。** `cdimage-download.chinauos.com` 对真文件回 200、对随机构造路径回 404，所以枚举探测是可信的。而 `professional-packages.chinauos.com` 对真 suite `eagle` 和编造的 `nosuchsuite` **一律 401**，`professional-security.chinauos.com` 对 `1070` 和 `9999` **一律 404**——两个站都不能用来判断某个东西存不存在。**探任何新站点前先做阴性对照**，否则很容易把 401 读成「存在但要授权」。

**每张盘的 squashfs 指纹都不同**，`SQUASHFS_SHA256` 必须按架构条件覆写。缺锚点时构建必须失败——那等于不知道手上这份是不是原货。要采新指纹用 harvest 模式：`gh workflow run build.yml -f harvest=true`，它只抽盘算 sha 然后停，不解包不构建。

**Range 下载必须自己续传。** 不能靠 `curl --retry`：那是个抽区间的下载，重试会按原区间从零再来。3 GiB 的盘在 1 MB/s 下将近一小时，一次卡顿就白费一小时（实测踩过，进度从 40% 掉回 8%）。`fetch-squashfs.py` 现在按已落地字节数重算区间、追加写入。

**瓶颈是带宽不是磁盘。** runner 有 145 GB 盘、可用 86~112 GB，塞得下 squashfs 加解开的 rootfs；而抽 3 GiB 要 57~59 分钟。所以单次抽取窗口开到 240 分钟（loong 300），retry 只留给真正的传输中断。

**devel 档只有 C 工具链。** V20 与 V25 的装机清单里**都没有 `g++`**，源里也补不上（14 个常用工具源内可装性实测 0/14，麒麟三家都是 14/14）。`cmake`/`git`/`gdb`/`strace`/`python3` 头文件同样是硬缺口。这不是切漏了，是补不了。

**V25 的 `dpkg`/`apt` 要指回 `.real`。** 它把真二进制改名成 `*.real`，再把 `dpkg`/`apt`/`apt-get` 换成 `deepin-immutable-ctl` 适配器；容器里没有 OSTree 部署，适配器必然失败。切片时指回真二进制，这是与真机的**有意偏差**，README 已记录。V20 不是不可变系统，没有这一层。

**两版的 dpkg admindir 不同。** V25 搬到了 `usr/lib/dpkg/var`（配合 OSTree 的 /var 分离），V20 还在 `var/lib/dpkg`。conf 里的 `ADMINDIR` 别照抄。这一处还牵连 SBOM：扫描器从镜像层 tar 里找 `/var/lib/dpkg/status` 且**不跨归档跟随符号链接**，放符号链接会让扫描结果静默变成空的。

**`info/format` 不能漏拷。** 它内容只有一个 `1`，却决定了 `Multi-Arch: same` 的包用 `pkg:arch.list` 命名；漏掉会让 dpkg 对一大片包报 `missing the list control file`。

## 跑 CI

```bash
gh workflow run build.yml --repo distrotwin/uos -f include-loongarch=true
gh workflow run build.yml --repo distrotwin/uos -f publish=true -f include-loongarch=true
gh workflow run build.yml --repo distrotwin/uos -f harvest=true          # 只采指纹
```

构建按「版本 × 架构」切分，一个 job 出三个档位——切片的抽盘加解包是一次昂贵的共享前置，三档各做一遍等于白花两遍。**测试仍是一镜像一 job、干净机器装载**，这条不许放宽。

改了 buildkit 之后**不能用 `gh run rerun --failed`**：`uses:` 的 pin 钉在调用方 commit 上，重跑会照旧用旧版脚本。

## 门禁

清单见规范第六节。这个仓库额外靠两道：`SQUASHFS_SHA256` 逐架构核对（抽错盘或抽残都会当场失败），以及解包后断言 rootfs 里真有 `sh`（`unsquashfs` 半途而废时目录也在，只看退出码看不出来）。

## 排错

- 先在本地复现，别拿 CI 当实验台；读远程 ISO 只要几十 KB，很多问题不必等一小时的抽盘
- 看到「拿不到」「不支持」这类结论，先分清是观察还是推断。本项目已经两次因为**只按一种命名抽样**而漏判：一次是麒麟 V10 的 libc6 在 `main` 而非 `universe`，一次是 UOS 的 `loong64` 盘因为只试了 `loongarch64` 而被判为不存在
- 日志里的 warning 要读
