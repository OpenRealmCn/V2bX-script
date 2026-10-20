# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

本仓库是 V2bX 一键安装 / 运维脚本的 fork，origin 为 `OpenRealmCn/V2bX-script`，默认分支 `master`。纯 Bash，无构建步骤、无测试。

> 本目录常与另外两个仓库并排放在工作区 `D:\poject\OpenRealmWork\v2bx` 下（`v2board/` 面板、`v2bx/` 节点服务端），三者**各自独立带 `.git`**。本文件自包含，单独 clone 本仓库时也能用。

## 文件构成

| 文件 | 作用 |
|---|---|
| `install.sh` | 一次性安装脚本。下载 release zip、装 systemd unit、部署配置、安装 `V2bX` 命令 |
| `V2bX.sh` | 交互式管理菜单，安装后落到 `/usr/bin/V2bX`（并软链 `/usr/bin/v2bx`） |
| `V2bX.service` | systemd unit，`ExecStart=/usr/local/V2bX/V2bX server` |
| `config/` | XrayR 时代的遗留模板与地理数据（`config.yml`、`dns.json`、`route.json`、`custom_inbound.json`、`custom_outbound.json`、`geoip.dat`、`geosite.dat`、`rulelist`）。**`config.yml` 与 `rulelist` 已不再是任何代码路径的来源**（见下文「具体修复清单」第 2 条），保留仅为兼容；`dns.json`/`route.json`/`custom_*.json`/`geoip.dat`/`geosite.dat` 仍由 install.sh 从 release zip 部署 |

改完务必做语法检查（这两个文件含中文与 ANSI 转义，sed 容易出错）：

```bash
bash -n install.sh && bash -n V2bX.sh
```

## 安装流程（install.sh）

按顺序：检测发行版 → `install_base` 装依赖（wget/curl/unzip/tar/socat）→ 架构映射 → 删旧目录 → 从 GitHub Releases 拉 zip → 解压到 `/usr/local/V2bX/` → 装 systemd unit → 复制配置到 `/etc/V2bX/` → 装 `V2bX` 命令。

架构映射只认三种，其余回退 `64`：

```
x86_64|x64|amd64  → 64
aarch64|arm64     → arm64-v8a
s390x             → s390x
```

对应下载的资产名是 `V2bX-linux-<arch>.zip`。

## 与 v2bx 主仓库的配置契约（已对齐）

这套脚本原先停留在 XrayR 时代，而 v2bx 早已换了配置格式，导致**一键安装装完节点起不来**。**现已全部修复**，脚本与 v2bx 的配置契约一致。

### 契约要点（改脚本时必须维持）

- **配置文件路径**：`/etc/V2bX/config.json`。`V2bX.service` 的 `ExecStart=/usr/local/V2bX/V2bX server` **不带 `--config`**，v2bx 走默认路径 `/etc/V2bX/config.json`（见 v2bx 仓库 `cmd/server.go:32`）。**service 文件因此不需要改**。
- **格式是 JSON5（允许注释），不是 YAML。** v2bx 的 `conf` 包无任何 yaml 依赖，`LoadFromPath` 只走 `json5.NewTrimNodeReader`。脚本生成的是纯 JSON——比 JSON5 更严格，是它的子集，安全。
- **顶层结构 `{Log, Cores[], Nodes[]}`，节点字段扁平**（`Core` / `ApiHost` / `ApiKey` / `NodeID` / `NodeType` / `Timeout` / `ListenIP` / `SendIP` / `CertConfig` 等），**不再是 XrayR 时代的 `CoreConfig` → `XrayConfig` 三层嵌套**。权威样例在 v2bx 仓库 `example/config.json`，字段定义在 `conf/*.go`。

> 生成模板里**每一个 JSON 键都必须能在 v2bx `conf/*.go` 的 `json:"..."` tag 中找到**，否则 v2bx 会静默忽略（`json.Unmarshal` 不报未知字段错）。改动模板后可用：`grep -ho 'json:"[A-Za-z0-9_]*"' v2bx/conf/*.go` 导出全部合法键再逐一对账。

### 具体修复清单

**1. `install.sh` 的 `check_status()` 查错服务名（已修）**

`install.sh` 第 86、89 行原先查的是 `XrayR.service` / `systemctl status XrayR`，而 `V2bX.sh` 里的同名函数（`V2bX.sh:271` 起）是正确的。后果是 `check_status` 恒返回 2（未安装），安装完成后的「V2bX 重启成功 / 可能启动失败」判断永远走失败分支、误报。现两处已改为 `V2bX`。

**2. `install.sh` 复制了 zip 里不存在的文件（已修）**

- 判断并复制 `config.yml`（`install.sh:143-144`）→ 改为 `config.json`。zip 里**只有 `config.json`**，没有 `config.yml`（v2bx 的 `release.yml` 打包时 `cp example/*.json ./build_assets/`）。
- `cp rulelist`（原 `install.sh:171-172`）→ **整块删除**。`rulelist` 不在 zip 里。v2bx 侧 `conf/node.go:33` 虽仍定义 `RuleListPath` 字段，但**它是可选字段、新配置模板不引用它**，属死代码。删掉比从 raw URL 拉取更简单，且不引入网络依赖与失败路径。

脚本没有 `set -e`，这类 `cp` 失败原先只打印、不中断，装完看起来「成功」。

**3. `V2bX.sh` 的 `generate` 模板整体改写为 JSON（已修）**

`generate_config_file()` 原先用 heredoc 生成 YAML 到 `/etc/V2bX/config.yml`，模板注释里还写着 `/etc/XrayR/...` 路径。现改为生成纯 JSON 到 `/etc/V2bX/config.json`，结构对齐 v2bx `example/config.json`。配套改动：

- `V2bX.sh:112`（`config` 函数）`vi /etc/V2bX/config.yml` → `config.json`
- 生成向导的提示文字、`mv config.json config.json.bak` 备份行为保留
- 内核类型从单选变为**可交互选择 xray / sing**，节点类型映射改为 v2bx 认识的小写值（`shadowsocks` / `vmess` / `trojan`；v2bx `api/panel/panel.go:58` 会做 `ToLower` 并把 `v2ray` 归一为 `vmess`）

### 生成模板的易踩点

- **sing 内核的 `OriginalPath` 必须留空。** `core/sing/sing.go:50` 在 `OriginalPath` 非空时会 `os.ReadFile`，文件不存在就 `return nil, err`——**sing 内核会直接构造失败**。v2bx 的 `example/config.json` 里写的是 `/etc/V2bX/sing_origin.json`，但该文件并不随 release zip 分发，照抄样例会让 sing 起不来。脚本生成的是 `"OriginalPath": ""`。
- **用户输入必须做 JSON 转义。** 脚本用 `json_escape()`（`V2bX.sh:357` 起）对 `ApiHost` / `ApiKey` 处理反斜杠与双引号，否则含 `"` 的输入会生成非法 JSON。注意 bash 的 `${s//\\/\\\\}` 参数展开在**经 heredoc 传输时反斜杠会被吞**，写这类替换时要用文件编辑、别用 heredoc。
- `NodeID` 做了非数字兜底（默认 `1`），`ApiHost` 空则默认 `http://127.0.0.1`。

### 验证方式

改完两个脚本必须过语法检查（含中文与 ANSI 转义，sed 容易出错）：

```bash
bash -n install.sh && bash -n V2bX.sh
```

生成的 JSON 模板要验证**合法 JSON**。可把 `generate_config_file` 里的 `if [[ x"${CoreType}" == x"sing" ]] ... fi` 内核选择块 + heredoc 抽成一个独立 harness，用不同参数跑一遍再喂给 `python -c "json.loads(...)"`。已实测四种组合（xray/vmess、sing/trojan、空输入、含 `"` 与 `\` 的 ApiKey）全部产出合法 JSON。

## 已迁移到 OpenRealmCn

上游引用（原为 `Yuzuki616/`）已全量切换，并修正了一个无效域名。

**域名陷阱**：原脚本里多处写 `raw.githubusercontents.com`（多了个 "s"）——**这是无效域名**，curl 返回 000、DNS 不解析。正确写法是 `raw.githubusercontent.com`。这些自更新命令在迁移前就是坏的，现已修好。`V2bX.sh:248` 的 `chiakge/Linux-NetSpeed` 只修了域名、保留原 owner（第三方）。

刻意**未动**的第三方引用：`install.sh:155` 的 `V2bX-project/V2bX/wiki`（错误提示里的 wiki 链接）、`V2bX.sh:554` 的 `cubemaze/V2bX`（菜单页脚署名）。这两处与本项目 fork 无关，改它们没有意义。

**`config/geosite.dat` 里的 `githubusercontents.com` 不要碰** —— 那是二进制 protobuf 地理路由数据库中的一条**域名记录**（属于 GitHub 镜像域名列表），不是 URL。该格式是长度前缀编码（`0x16` = 22 字节），用 sed 替换会留下错误的长度前缀，损坏该记录及其后所有分类。校验时排除它：

```bash
grep -rn "githubusercontents" --include="*.sh" .                     # 应为 0
grep -rn "Yuzuki616" --exclude-dir=.git --exclude=CLAUDE.md .         # 应为空
```

同步上游时这些会被改回，合并后需重新检查。

## 发布链路依赖

`install.sh` 靠 `api.github.com/repos/OpenRealmCn/V2bX/releases/latest` 取版本号再拼下载 URL，所以 **`OpenRealmCn/V2bX` 必须有带资产的 Release**，否则脚本在「检测 V2bX 版本失败」处直接 `exit 1`。

该仓库是 fork，**GitHub 不会为 fork 自动派发 push / release 事件**，Release 要用 `workflow_dispatch` 手动触发（详见 v2bx 仓库的 CLAUDE.md）。发布命令：

```bash
gh workflow run release.yml --repo OpenRealmCn/V2bX --ref dev_new -f release_tag=v0.4.0-or1
```

脚本需要的三个资产名（与 `friendly-filenames.json` 产出一致）：
`V2bX-linux-64.zip` / `V2bX-linux-arm64-v8a.zip` / `V2bX-linux-s390x.zip`。

`install.sh` 只 `unzip`、**不校验** `.dgst`（虽然 workflow 会生成），解压后自行 `chmod +x V2bX`。