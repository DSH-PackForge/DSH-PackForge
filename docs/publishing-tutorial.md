# 整合包仓库上传到 GitHub 教程

> 定位：**上手教程（非 spec）**。把「一个整合包如何从本机走上 GitHub、被市场自动收录」写成分步可执行流程。权威契约见 [`../specs/publishing/v1.md`](../specs/publishing/v1.md)，冲突时以 spec 为准。
>
> 本文既写给人类作者，也**可以直接交给 AI 助手照着执行**：每步都是「一条可复制的命令 + 明确的输入/输出 + 校验点」，不留歧义。AI 只需替换 §0 的占位变量，逐节执行，并在每节末尾回报校验点。

## 0. 变量替换表（动手前先填）

| 占位符 | 含义 | 示例 |
|---|---|---|
| `<owner>` | GitHub 用户 / 组织名 | `DSH-PackForge` |
| `<name>` | 包名 = 仓库名（kebab-case，与 `manifest.name` 一致） | `all-about-whales` |
| `<version>` | 包版本（semver，与 `manifest.version` 一致） | `1.0.0` |
| `<tag>` | Release tag = `v<version>` | `v1.0.0` |

> 硬约束：`<name>` 与 `<version>` 必须和根 `manifest.json` 里的 `name` / `version` **逐字一致**，否则采集器要么扫不到、要么索引键错位。

## 1. 结论先行（三件硬事）

被市场自动收录，只有三件硬事：

1. 一个 **public 仓库**，打了 **主题标签 `dsh-pack`**，默认分支根放了 **`manifest.json`**；
2. 一个 **正式 Release**（非 draft、非 prerelease），tag = `v<version>`；
3. Release 挂 **两个资产**：`<name>-<version>.dspack` 与 `<name>-<version>.dspack.sha256`。

做完这三点，采集器（`dsh-pack-market/scripts/collect.mjs`）就会：扫 `topic:dsh-pack` → 读默认分支根 `manifest.json` → 取 `releases/latest` 的资产 → 自动写进市场索引。无需向任何地方提交 PR。

## 2. 前置条件

- `git`、[`gh` CLI](https://cli.github.com/) 已安装；
- `gh auth login` 已登录到对 `<owner>` 有写权限的账号；
- 整合包产物已就绪（二选一，见 §3）：
  - **路径 A（推荐）**：装了 `dsh-packforge-app`，有 `dspack` CLI，本机有一个 profile；
  - **路径 B（手工）**：手里已有 `manifest.json` + `overrides/` + 一个已打包的 `.dspack`。

## 3. 准备产物（二选一）

两条路径只在这一步分叉，产出「源文件 + `.dspack` + `.sha256`」三样东西后，在 §4 完全合流。

### 3.1 路径 A：`dspack pack --repo`（推荐，一步到位）

```bash
dspack pack --repo <profile目录> --out <输出目录>
```

产出（`<name>` 不带版本号）：

```
<输出目录>/<name>/                     # 源仓库（已 git init + commit）
├── manifest.json                      # v5（根，含顶层 sha256 字段）
├── overrides/                         # 用户文件
├── README.md                          # 由 manifest 渲染
└── release/                           # ★ 成品区（不进 git，.gitignore 已排除）
    ├── <name>-<version>.dspack
    └── <name>-<version>.dspack.sha256   # 侧车已写好，跳过 §3.3
```

校验点：`release/` 里两个文件都在；`git log` 有提交 `export:<name>@<version>`。

### 3.2 路径 B：手工组装

若你已有现成文件，按 `specs/publishing/v1.md` §2.3 的布局摆好并建 git 仓库：

```bash
mkdir <name> && cd <name>
# 放置：manifest.json（根，v5）、overrides/、README.md、可选的 package.json / pnpm-lock.yaml / .github/
git init
git add .
git commit -m "export:<name>@<version>"
```

### 3.3 生成 sha256 侧车（仅路径 B；路径 A 已由 `--repo` 写好）

侧车文件名 = 打包资产名 + `.sha256`，内容是 **64 位小写十六进制 sha256**（可带换行，采集器取第一个空白分隔 token）。

Windows PowerShell：

```powershell
(Get-FileHash .\<name>-<version>.dspack -Algorithm SHA256).Hash.ToLower() `
  | Out-File -Encoding ascii .\<name>-<version>.dspack.sha256
```

macOS / Linux：

```bash
shasum -a 256 <name>-<version>.dspack | cut -d' ' -f1 > <name>-<version>.dspack.sha256
```

校验点：`cat <name>-<version>.dspack.sha256` 得到 64 位小写 hex，无大写、无多余内容。

## 4. 建仓库 + 打主题标签

```bash
# 在源仓库目录（路径 A：<输出目录>/<name>；路径 B：刚建的 <name>/）内执行：

# 1) 建 public 仓库并推送
gh repo create <owner>/<name> --public --source . --push

# 2) 打主题标签 dsh-pack（全网唯一发现入口，漏了就永远扫不到）
gh repo edit <owner>/<name> --add-topic dsh-pack
```

校验点（跑完回报这三项）：

```bash
gh repo view <owner>/<name> --json visibility,repositoryTopics \
  --jq '{visibility, topics: [.repositoryTopics[].name]}'
# 期望：{"visibility":"PUBLIC", "topics":["dsh-pack", ...]}
```

> 三个「必须」缺一不可：**public**（私有仓库 `raw.githubusercontent.com` / Release 读不到，会被跳过）、**`dsh-pack` 话题**、**根 `manifest.json`**。另外 `archived` 的仓库会被跳过，别归档。

## 5. 发正式 Release

```bash
# 路径 A：资产在 release/ 目录
gh release create <tag> \
  release/<name>-<version>.dspack \
  release/<name>-<version>.dspack.sha256 \
  --title "<name> v<version>"

# 路径 B：资产在当前目录
gh release create <tag> \
  <name>-<version>.dspack \
  <name>-<version>.dspack.sha256 \
  --title "<name> v<version>"
```

硬约束（来自 `specs/publishing/v1.md` §3）：

- tag 必须是 **`v<version>`**（与 `manifest.version` 对应）；
- Release 必须是**正式发布**——`gh release create` 默认即正式，**不要**加 `--draft` / `--prerelease`，否则 `releases/latest` 取不到；
- **同一 Release 只放一个打包资产**（采集器按 `.dspack > .tgz > .zip` 只取一个，混放可能选中旧包）。

校验点：

```bash
gh release view <tag>
# 期望：Draft=否、Pre-release=否；Assets 恰好两个：.dspack + .dspack.sha256
```

## 6. 发布校验清单（人和 AI 共用）

- [ ] 仓库 `public`；About 已打 `dsh-pack`；未 `archived`
- [ ] 默认分支根 `manifest.json` 存在，`manifestVersion` / `type` / `name` / `version` 齐全，且与仓库名 / tag 一致
- [ ] Release tag = `v<version>`
- [ ] Release 为正式发布（非 draft / 非 prerelease）
- [ ] 只有一个打包资产 `<name>-<version>.dspack`（或过渡期 `.tgz` / `.dspack` v2）
- [ ] 侧车 `<name>-<version>.dspack.sha256` 为 64 位小写 hex

## 7. 收录与验证

收录是**异步**的——采集器按周期跑，不是发版即时。等一轮扫描后查市场：

```bash
# 市场索引里搜你的包（下载市场仓库后 grep，或在市场页搜索 <name>）
curl -s https://raw.githubusercontent.com/dsh-pack-market/dsh-pack-market/main/index/index.json \
  | jq '.modpacks[] | select(.name == "<name>")'
```

期望出现你的条目，且 `downloadUrl` / `sha256` / `size` 三指针齐全、`downloadUrl` 指向你 Release 的 `.dspack` 资产。

## 8. 常见坑速查

| 现象 | 原因 | 修法 |
|---|---|---|
| 市场扫不到 | 没打 `dsh-pack` 话题 / 仓库私有 / 已 `archived` | §4 三件套补齐 |
| 有条目但下载失败 | Release 是 draft / prerelease，`releases/latest` 取不到 | 删掉重发正式 Release |
| 装的是旧包 | 同一 Release 混放了多个打包资产 | 只留一个 `.dspack` |
| sha256 对不上 | 侧车写了大写 hex 或带了多余内容 | 重生成，只写小写 hex |
| tag 与版本不一致 | `v<tag>` 没跟 `manifest.version` 对齐 | 改 tag 或改 manifest，两者一致 |

## 9. 变体：`downloadUrl` 直链（可选，替代 Release）

若你走裸链分发（raw / 对象存储 / CDN），可在 `manifest.json` 写 `downloadUrl` 直接指向打包资产，采集器会**跳过 Release**、改读 `<downloadUrl>.sha256` 侧车与 `HEAD` 的 `content-length`。否则**省略 `downloadUrl`**，交给 Release 默认流程（§5）。
