---
name: publish-to-github
description: 把 DSH 整合包发布到 GitHub 供 dsh-pack-market 自动收录。当用户要「发布/上传整合包到 GitHub」「发 Release」「被市场收录」时使用。覆盖：准备产物（dspack pack --repo 或手工）、建 public 仓库、打 dsh-pack 话题、发正式 Release（.dspack + .sha256 侧车）、逐项校验。
---

# 发布整合包到 GitHub

把「一个整合包从本机走上 GitHub、被 `dsh-pack-market` 自动收录」执行到底。权威契约见 `specs/publishing/v1.md`，冲突时以 spec 为准；分步教程见 `docs/publishing-tutorial.md`。

## 执行前：收集变量

先向用户确认（或从 manifest / 产物读取）以下四个变量，缺一不可：

| 变量 | 含义 | 来源 |
|---|---|---|
| `<owner>` | GitHub 用户 / 组织名 | 问用户 |
| `<name>` | 包名 = 仓库名（kebab-case） | `manifest.name` |
| `<version>` | 包版本（semver） | `manifest.version` |
| `<tag>` | Release tag = `v<version>` | 由 `<version>` 派生 |

> 硬约束：`<name>` 与 `<version>` 必须和根 `manifest.json` 的 `name` / `version` **逐字一致**，否则采集器扫不到或索引键错位。

## 第 1 步：确认产物路径（二选一）

- **路径 A（推荐）**：用户装了 `dsh-packforge-app`、有 `dspack` CLI 和本机 profile → 用 `dspack pack --repo`。
- **路径 B（手工）**：用户手里已有 `manifest.json` + `overrides/` + 已打包的 `.dspack`。

## 第 2 步：准备产物（按所选路径）

**路径 A**：

```bash
dspack pack --repo <profile目录> --out <输出目录>
```

产出 `<输出目录>/<name>/`：源仓库（已 git init + commit）+ `release/` 成品区（`.dspack` 与 `.dspack.sha256` 侧车已写好）。校验点：`release/` 里两个文件都在；`git log` 有提交 `export:<name>@<version>`。

**路径 B**：在 `<name>/` 下摆好 `manifest.json`（根，v5）、`overrides/`、`README.md`，然后：

```bash
git init && git add . && git commit -m "export:<name>@<version>"
```

再生成 sha256 侧车（文件名 = 打包资产名 + `.sha256`，内容 = 64 位小写 hex）：

```powershell
# Windows PowerShell
(Get-FileHash .\<name>-<version>.dspack -Algorithm SHA256).Hash.ToLower() `
  | Out-File -Encoding ascii .\<name>-<version>.dspack.sha256
```

```bash
# macOS / Linux
shasum -a 256 <name>-<version>.dspack | cut -d' ' -f1 > <name>-<version>.dspack.sha256
```

校验点：`.dspack` 与 `.dspack.sha256` 都在；侧车为 64 位小写 hex，无大写、无多余内容。

## 第 3 步：建 public 仓库 + 打话题

在源仓库目录内执行：

```bash
gh repo create <owner>/<name> --public --source . --push
gh repo edit <owner>/<name> --add-topic dsh-pack
```

校验点：

```bash
gh repo view <owner>/<name> --json visibility,repositoryTopics \
  --jq '{visibility, topics: [.repositoryTopics[].name]}'
# 期望：{"visibility":"PUBLIC", "topics":["dsh-pack", ...]}
```

> 三件硬事缺一不可：**public**、**`dsh-pack` 话题**、**根 `manifest.json`**。`archived` 仓库会被跳过，别归档。

## 第 4 步：发正式 Release

```bash
# 路径 A：资产在 release/ 目录
gh release create <tag> \
  release/<name>-<version>.dspack \
  release/<name>-<version>.dspack.sha256 \
  --title "<name> v<version>"

# 路径 B：资产在当前目录（去掉 release/ 前缀）
gh release create <tag> \
  <name>-<version>.dspack \
  <name>-<version>.dspack.sha256 \
  --title "<name> v<version>"
```

硬约束：

- tag 必须是 `v<version>`（对应 `manifest.version`）；
- 必须是**正式发布**——**不加** `--draft` / `--prerelease`，否则 `releases/latest` 取不到；
- **同一 Release 只放一个打包资产**（采集器按 `.dspack > .tgz > .zip` 只取一个，混放可能选中旧包）。

校验点：`gh release view <tag>` → 非 draft、非 prerelease；资产恰好两个（`.dspack` + `.dspack.sha256`）。

## 第 5 步：逐项校验并回报用户

- [ ] 仓库 `public`；已打 `dsh-pack`；未 `archived`
- [ ] 默认分支根 `manifest.json` 存在，`name` / `version` 与仓库名 / tag 一致
- [ ] Release tag = `v<version>`
- [ ] Release 为正式发布（非 draft / 非 prerelease）
- [ ] 只有一个打包资产 `<name>-<version>.dspack`
- [ ] 侧车 `<name>-<version>.dspack.sha256` 为 64 位小写 hex

## 第 6 步：告知收录方式

收录是**异步**的（采集器按周期跑，非发版即时）。告诉用户等一轮扫描后查市场：

```bash
curl -s https://raw.githubusercontent.com/dsh-pack-market/dsh-pack-market/main/index/index.json \
  | jq '.modpacks[] | select(.name == "<name>")'
```

期望出现条目，且 `downloadUrl` / `sha256` / `size` 三指针齐全。

## 常见坑速查（出现即按此修）

| 现象 | 原因 | 修法 |
|---|---|---|
| 市场扫不到 | 没打 `dsh-pack` 话题 / 仓库私有 / 已 `archived` | 三件套补齐 |
| 有条目但下载失败 | Release 是 draft / prerelease | 删掉重发正式 Release |
| 装的是旧包 | 同一 Release 混放多个打包资产 | 只留一个 `.dspack` |
| sha256 对不上 | 侧车写了大写 hex 或带多余内容 | 重生成，只写小写 hex |
| tag 与版本不一致 | `v<tag>` 没对齐 `manifest.version` | 改 tag 或改 manifest |
