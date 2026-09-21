# 启动器集成指南（Launcher Integration Guide）

> 定位：**实现指南（非 spec）**。启动器（`dsh-packforge-app` / `dspack` CLI）承担整合包全生命周期的**三个独立职责**——**导出（`pack`）、市场（`market`）、导入（`install`）**。本文分三部分分别讲清，边界互不混淆。
>
> 权威契约：`../specs/manifest/`、`../specs/pack-structure/`、`../specs/index/`、`../specs/publishing/`。本文只做串联 + 实现要点，**冲突时以 spec 为准**。
>
> 主线：现行标准（manifest v5 + `.dspack` v3）。历史版本兼容统一放到「导入」部分的末尾一节次要处理。

## 职责总览

| 职责 | CLI 命令 | 输入 → 输出 | 对应 spec |
| --- | --- | --- | --- |
| **导出** | `dspack pack` / `pack-home` / `pack --repo` | profile / `$DSH_HOME` → `.dspack` v3（`.sha256` 侧车仅 `--repo` 形态写） | manifest v5（buildManifest）、pack-structure v3（扫描/过滤/打包）、publishing（发布） |
| **市场** | `dspack list / view / market` | 中心索引 → 列表 / 详情 / 下载好的包 | index（索引契约）、publishing（收录流程） |
| **导入** | `dspack install` | 包文件 → 可运行 profile / `$DSH_HOME` | manifest / pack-structure 的「导入行为」章节 |

三者是三个**并列、独立**的环节，由 `.dspack`（导出产物 = 导入输入）与 `index.json`（市场索引 = 下载来源）衔接，但实现上各管各的。

---

# 第一部分 · 导出（Pack）

> 目标：把本机一个 profile（或整个 `$DSH_HOME`）打成可分发、可复现的 `.dspack` v3。
>
> 实现入口（`dsh-packforge-app/packages/core/src/`）：`pack.js`（`packProfile` / `packHome`）、`manifest.js`（`buildManifest` / `buildHomeManifest` / 坐标钉定）、`scan.js` + `security.js`（扫描 + 过滤）、`workspace.js`（`.dshpkcfg` 参数）、`repo.js`（`exportRepo` 源仓库形态）。权威契约仍是 `../specs/manifest/v5.md`、`../specs/pack-structure/v3.md`，本文只做实现串联，冲突以 spec 为准。

## 1.1 三条导出路径（先分清）

导出不是「一条流程」，而是**三个入口、三种产物**：

| 入口 | CLI | 输入 | 产物 | 写 `.sha256` 侧车？ |
| --- | --- | --- | --- | --- |
| `packProfile` | `dspack pack <profile\|dir>` | 单个 profile 目录 | `.dspack` v3（**内存**拼 ZIP，返回 `sha256` / `size`） | 否 |
| `packHome` | `dspack pack-home <home>` | 整个 `$DSH_HOME` | `.dspack` v3（dshhome 形态） | 否 |
| `exportRepo` | `dspack pack --repo` | 单个 profile 目录 | **源仓库**（`release/` 内 `.dspack` + `.sha256`） | 是（见 §1.6） |

- 纯 `pack` / `pack-home` **只返回** `sha256` 与 `size`（供索引/收录校验），**不落 `.sha256` 侧车**；侧车由 `exportRepo` 写，或发布时按 `../specs/publishing/v1.md` §4 手写。
- 三个入口都走「扫描 → manifest → 组装 → 写盘」的主干，区别在形态与产物（§1.4 ~ §1.6）。

## 1.2 参数来源：`.dshpkcfg` 打底 + 显式覆盖

导出参数不是每次手填，而是**工作区快照 `.dshpkcfg` 打底、显式参数覆盖**（`workspace.js` 的 `exportFromWorkspace` / `exportHomeFromWorkspace`）：

- `config`（`.dshpkcfg`，契约 `../specs/workspace-config/v1.md`）打底，CLI flag / GUI 表单 / AI 工具参数 `overrides` 覆盖，`undefined` / `null` 视为「未设置、不覆盖」。
- **dshVersion 注入优先级**（`cli.js`）：`--dsh-version` > `.dshpkcfg.dshVersion` > 本机最新已装版本（`listInstalledDshVersions()[0]`）> 空串（导入端兜底）。`buildManifest` 本身只 `opts.dshVersion || ''`，**注入在 CLI 层**，不在扫描器。
- **dshhome 的 `exportContent` 开关**（`{skill,preset,instruction,data}` 布尔）→ `exclude` 前缀映射：`skill:false` → 排除 `skills/`，`preset:false` → 排除 `.agent-presets/`，`instruction:false` → 排除 `AGENTS.md`，`data:false` → 排除 `data/`。
- `.dshpkcfg` 自身**永不进包**（命中 `security.js` 的 `DENY_EXACT` 精确名 `.dshpkcfg`，与凭据 / 运行时状态一同过滤）。

## 1.3 扫描与安全过滤（两形态共用第一步）

`scanProfile(host, dir)` 递归扫描源目录（`scan.js`）：

- 返回 `{ files: [{rel, abs, size}], excluded: [{rel, abs, reason}] }`，`rel` **一律 `/` 分隔**（与 Windows 反斜杠无关），按 `rel` 排序。
- **符号链接整体跳过**（`reason: 'symlink'`），不做解引用——防逃出源目录与死循环。
- 读不到的目录（无权限等）静默跳过，读不到大小的文件记为 0。

安全过滤 `isExcluded(rel)`（`security.js`）共**五类**，命中即排除：

| 类别 | 命中方式 | 内容（摘） |
| --- | --- | --- |
| 精确名 | 任意路径段 | `node_modules` `dist` `build` `coverage` `.cache` `cordis.yml` `manifest.json` `package-lock.json` `yarn.lock` `.env` `.npmrc` … 及 home 级 `.credentials.yaml` `.anonymous-user-id` `settings.yaml` `.dshpkcfg` |
| 扩展名 | 文件后缀 | `.key` `.pem` `.p12` `.pfx` `.crt` `.der` `.asc` |
| 文件名正则 | basename | `.env*`、`credentials*.ya?ml`、`*.credentials`、SSH 私钥 `id_rsa*` 等、`secrets*.json/yml`、`*token*` / `*api_key*` |
| 相对路径正则 | 整条 rel | `*.tgz` `*.tar.gz` `*.zip` `*.dspack`（**禁止嵌套打包任何压缩包**） |
| 路径前缀 | rel 前缀 | `attachments/` `profiles/web/` `profiles/headless/` `skills/.system/`（home 级运行时 / 基线目录） |

> 注意：`.credentials.yaml` / `settings.yaml` 等是**精确名（任意段命中）**，不是独立的「v3 home 级」类别；`attachments/`、`profiles/web/` 等才是第 5 类**路径前缀**。完整清单以 `security.js` 为准，spec 侧对应 `../specs/pack-structure/v1.md` §5 与 `v3.md` §8。

## 1.4 生成 manifest v5

### 1.4.1 profile 形态（`buildManifest`）

字段来源（`manifest.js`）：

| 字段 | 来源 |
| --- | --- |
| `manifestVersion` / `type` | 固定 `5` / `"profile"` |
| `name` | `sanitizeSlug(opts.name \|\| profile.name)`（小写、非 `[a-z0-9-]` 转 `-`） |
| `version` | `opts.version \|\| package.json.version \|\| '1.0.0'` |
| `displayName` | `opts.displayName \|\| niceName(package.json.name) \|\| name`（去 `dsh-profile-` / `dsh-` 前缀） |
| `description` / `author` | `opts` 优先，回退 `package.json` 同名字段 |
| `icon` | `opts.icon \|\| findIcon(scan.files)`（`icons?/<…>.<png\|jpg\|webp\|ico\|svg>` 或根 `logo.*`） |
| `dshVersion` | `opts.dshVersion`（CLI 注入，见 §1.2） |
| `profileName` | `opts.profileName \|\| profile.name` |
| `bundles` | `extractBundles(pkg)`：`package.json` 的 `dsh.profile.bundles` 原文顺序、去重、只留字符串 |
| `patch` | 读 `cordis.patch.yml` 全文（缺失为空串） |
| `dependencies` | ★ `coordinatesFromProfileDeps`（坐标钉定，见下） |
| `files` | `opts.files ?? []`（重内容指针，见 `../specs/manifest/v5.md` §8） |

### 1.4.2 ★ 依赖坐标钉定（可复现核心）

`coordinatesFromProfileDeps(host, dir, deps)` 把 `package.json.dependencies` 的「包名 → pnpm spec」转成 manifest 的「**坐标 → 固定版本**」：

- **git 依赖**（`github:owner/repo[#sha][&path:pkg]` 或 `git+https://…`）：sha 优先取 spec 的 `#sha`，缺则从 `pnpm-lock.yaml` 的 `packages:` 区块 `resolution.commit` 补（`gitCommitFromLock`），再缺标 `latest`（跟随默认分支最新，不强制钉 sha）。坐标形如 `github:owner/repo` 或 `github:owner/repo#path:/pkg`。
- **npm 依赖**：若是 **semver 范围**（`^` / `~` / `>` 等，非精确 `x.y.z`），读 `node_modules/<name>/package.json` 的实测 `version` 钉精确；**已是精确版本则原样保留**。
- **协议型 spec 原样保留**：带 `:` 的（`file:` / `link:` / `workspace:` / URL / 别名）**不做钉定**——否则导入侧会当 npm registry 包去 404。

> 正向（坐标 → `package.json` 依赖，导入侧）与反向（`package.json` 依赖 → 坐标，导出侧）的转换表见 §3.3；导出侧就是它的**逆**。

### 1.4.3 dshhome 形态（`buildHomeManifest` + 四类单元识别）

`packHome` 先 `summarizeHome(files)` 从扫描结果识别四类单元，再生成 manifest：

| 单元 | 判定规则 | manifest 落点 |
| --- | --- | --- |
| profile | `profiles/<name>/package.json` | `profiles[name]` = `buildProfileUnit`（`bundles` + `dependencies` + `patch`，即 v4 契约去掉 `profileName`） |
| preset | `.agent-presets/<id>/agent.cordis.yml` | `presets[id] = { path: ".agent-presets/<id>" }` |
| skill | `skills/<name>.md`（平铺）或 `skills/<name>/SKILL.md`（目录 bundle） | `skills[] = { path: "skills/<name>" }`（去重） |
| 指令 | 根 `AGENTS.md` | `instructions = "AGENTS.md"` |

- **`web` / `headless` 不进包**：识别后过滤掉这两个安装基线 profile 模板（由 `dshVersion` 决定，见 `../specs/manifest/v5.md` §4）。
- `defaultProfile` = `opts.defaultProfile || 字母序第一个 profile`。
- `skills` 是**轻索引**（只记 `path`）；重技能才带 `sha256` / `size` / `urls`（`../specs/manifest/v5.md` §6）。

## 1.5 组装归档（内存拼 ZIP，非暂存区）

> 实现是**在内存里拼 ZIP 一次写盘**，全程只读源目录、**不落任何临时清单 / 暂存区**，自然不改动源 profile / `$DSH_HOME`。

1. **条目映射** `dspackEntryPath(rel)`：机器文件（`package.json` / `pnpm-workspace.yaml` / `pnpm-lock.yaml`，`pack.js` 的 `ROOT_MACHINE`）→ 归档根；**其余用户文件 → `overrides/<rel>`**。
2. **profile 形态的 `home/`**：另扫上一级目录 `opts.home`（缺省 `profile.dir/../..` = `$DSH_HOME`），过滤掉 `profiles/` 前缀后由 `opts.homeInclude`（rel 白名单 Set）勾选，勾选项进 `home/<rel>`（→ `$DSH_HOME` 根）。**默认一个都不带**，要带全局 skill / preset 得显式勾选。
3. **dshhome 形态**：用户文件按 `$DSH_HOME` 相对路径**平铺**进 `overrides/`（`overrides/profiles/<name>/`、`overrides/.agent-presets/`、`overrides/skills/`、`overrides/AGENTS.md`、`overrides/data/`）。
4. **`manifest.json` + `dspack.json` 最后写入**归档根，覆盖任何扫描残留；`dspack.json` = `{ "format": "dspack", "version": 3 }`。
5. **纯 ZIP**（`fflate` 的 `zipSync`，标准 `PK` 头），文件名 `<manifest.name>-<manifest.version>.dspack`。
6. 输出已存在且未 `--force` → 抛「输出文件已存在」；空扫描 / 无选中文件 → 抛错（不产出空包）。

> 机器文件写根、用户文件写 `overrides/`，与 `../specs/pack-structure/v3.md` §3 布局一致；机器文件用 UTF-8 **无 BOM**（Windows 误写 BOM 会让安装端 `JSON.parse` 崩）。

## 1.6 源仓库形态（`exportRepo` / `dspack pack --repo`）

把 profile 物化为**可二次开发 / 重打包的 git 仓库**，`content` 三档（默认 `readme`）：

| 档 | 内容 |
| --- | --- |
| `manifest` | 仅 `manifest.json` |
| `readme` | `manifest.json` + `README.md`（由 manifest 渲染） |
| `full` | 全套：机器文件进根、其余进 `overrides/` + `.dspackignore` |

- 仓库目录 = `<out>/<name>`（**不带版本号**）；`release/` 始终产出 `.dspack` + `.sha256` 侧车（`.gitignore` 不入库）。
- **先产 release 拿 sha256**，再把顶层 `manifest.sha256` 写进仓库根的 `manifest.json`——该字段只存在于**源仓库形态**，`.dspack` 内部的 manifest 不带（避免循环）。
- **版本冲突**：`release/` 已有同版本产物且未「覆盖 / 跳过」→ 抛 `ReleaseConflictError`（`replaceRelease: true` 覆盖 / `'skip'` 跳过）。
- git：首次 `init + add + commit`（commit 消息 `export:<name>@<version>`），之后增量提交；无 git / 无变更降级不算失败。

## 1.7 打包前预览（`inspect`）

`inspectProfile` / `inspectHome`（CLI `dspack inspect`）是**干跑检查**：返回扫描结果 + manifest v5 预览 + 特殊目录摘要（skill / agent-preset / icon，`summarizeSpecial`）+ home 级候选（供 GUI 勾选），**不写任何文件**。用于打包前「将包含 / 排除哪些文件、manifest 长什么样」的人工确认。

## 1.8 导出自检清单

- [ ] **三入口分清**：`pack`（单 profile）/ `pack-home`（dshhome）/ `pack --repo`（源仓库）；纯 pack 只返回 `sha256` / `size`，侧车由 repo 形态或发布环节写。
- [ ] **字段完整**：`manifest.json` 必需字段齐全；`displayName` / `description`（多语言）、`author`、`icon`、`dshVersion`（精确）、`patch` 尽量填。
- [ ] **坐标钉定**：npm 范围依赖已钉精确、git 依赖 sha 已从 spec / pnpm-lock 补齐；`file:` / `link:` / `workspace:` 协议 spec 原样保留未被压平。
- [ ] **机器文件**：`package.json`（快照）、`pnpm-workspace.yaml`、`pnpm-lock.yaml` 一并进包根；`package.json` 只是快照，`dependencies` / `bundles` 由 manifest 权威重建。
- [ ] **机器 / 用户分离**：用户文件只进 `overrides/`（+ 勾选的 `home/`）；`manifest.json` / `dspack.json` 最后写入覆盖残留。
- [ ] **五类安全过滤生效**：含 home 级路径前缀（`profiles/web/` `profiles/headless/` `attachments/` `skills/.system/`）与精确名（`.dshpkcfg` / `.credentials.yaml` / `settings.yaml`）；重内容走 `files[]` / `skills[]` 不进包体。
- [ ] **dshhome 专属**：`profiles` 非空且不含 `web` / `headless`，`defaultProfile` 指向存在的 key；四类单元（profile / preset / skill / 指令）识别正确。

---

# 第二部分 · 市场（Market）

> 目标：发现、浏览、下载整合包。输入是中心索引，输出是「下载好的包 + 三指针」，交给导入环节。

## 2.1 消费 `index.json`

事实源 `dsh-pack-market/index/index.json`（部署副本 `web/index.json`），契约 `../specs/index/index.md`：

- `schemaVersion` 固定 `2`（**索引契约版本**，与 manifest 版本无关）；
- `modpacks[]` 按 `updatedAt` 倒序，每条是「精简指针」——够列表 / 搜索 / 安装命令所需，**不含** `bundles[]` / `dependencies{}` / `files[]` / `profiles` / `presets` / `skills` 等较大字段。

列表 / 搜索（`dspack list`）只用 `index.json`，别一次拉全量详情。

## 2.2 懒加载详情（`dspack view`）

详情页需完整字段时，按条目 `id`（= `${owner}.${repo}`）拉 `packs/<id>/manifest.json`（完整原始 manifest v5）+ `README.md`。**`id` 从 `owner` + `repo` 直接拼，不反向解析**（owner 无点、repo 可含点）。

## 2.3 下载与整包校验（`dspack market`）

1. 下载条目 `downloadUrl` 到临时目录（别直接下到 profile 目录）；
2. 校验 `size`（下载字节数 = 条目 `size`，早期预警）；
3. 校验 `sha256`（整包 SHA-256 与条目逐字节比对）；
4. 任一不匹配 → 丢弃临时文件、报「包不完整 / 已被篡改」。

三个指针 `downloadUrl` / `sha256` / `size` 即市场环节交付给导入环节的输入。`downloadUrl` 的来源（Release 资产 / 直链）由采集器统一（`../specs/publishing/v1.md` §5），市场环节只消费。

---

# 第三部分 · 导入（Install）

> 目标：把到手的包文件变成可运行的 profile / `$DSH_HOME`。

## 3.1 解包与容器识别

`.dspack` 是标准 ZIP（首两字节 `PK`），「是不是 dspack、哪个版本」由根 `dspack.json` 承载：

| 情况 | 结论 |
| --- | --- |
| 缺失 / 非对象 | 不是 dspack（外来 ZIP / 损坏）→ 拒载 |
| `format` ≠ `"dspack"` | 拒载 |
| `version` ≠ 3 | 报「不支持该版本」（历史 `.dspack` v2 见 §3.5） |
| `version` = 3 | **现行**：`.dspack` v3，承载 manifest v5 → §3.2 |

## 3.2 主线：manifest v5 + `.dspack` v3

v5 用 `type` 统一两种形态：

| `type` | 语义 | 容器布局 |
| --- | --- | --- |
| `"profile"` | 单 profile 整合包 | `overrides/` → profile 根 + 可选 `home/` → `$DSH_HOME` |
| `"dshhome"` | 整个 `$DSH_HOME` 快照（多 profile + 全局 preset / skill / 指令） | `overrides/` → `$DSH_HOME` 根平铺 |

`"collection"` 预留，校验器拒绝。

### 3.2.1 前置：安装 `dshVersion` + 建立实例

1. 读 `dshVersion`（精确版本；缺省用本机最新已装版本），未装则自动下载（GitHub-only 标签走源码构建），装不上报错；
2. 为整合包**新建独立实例**（专属 DSH_HOME / profile 命名空间，互不污染），`web` / `headless` 系统模板保持纯净。

### 3.2.2 profile 形态

1. 依赖重建（§3.3，`pnpm install`）；
2. `overrides/` 覆盖到 profile 根（`cordis.patch.yml` 等用户文件**最后落盘，压过依赖默认项**）；
3. 对账（§3.4，`reconcileProfile`）；
4. `home/`（可选）覆盖到 `$DSH_HOME` 根（全局 skill / `.agent-presets` / `AGENTS.md`）；
5. `files[]` 重内容下载 + 校验（§3.5），失败回滚；
6. 新建 profile：名称取 manifest 的 `profileName`（缺省 `pack`），设为该实例默认 profile。

### 3.2.3 dshhome 形态

1. 校验 `profiles` 非空、不含 `web` / `headless`，`defaultProfile` 指向存在的 key；
2. 建实例 `$DSH_HOME`；
3. **逐 profile**：`overrides/profiles/<name>/` 落盘 → apply + `pnpm install` + `reconcileProfile`（各自独立）；
4. home 级 `overrides/` 其余内容落盘（`.agent-presets/` / `skills/` / `AGENTS.md` / `data/`）；
5. `files[]` / `skills[]` 下载 + 校验（§3.5），失败回滚；
6. `defaultProfile` 设为默认。

> dshhome 四类单元的落盘与依赖坐标层见 `../specs/pack-structure/v3.md` §6：profile 走 `dependencies`（权威重建），preset / skill / 全局指令都是纯文件。

## 3.3 依赖重建：`dependencies` → `package.json` → `pnpm install`

manifest 是**唯一事实源**：覆盖重建 profile 的 `package.json`，再 `pnpm install`（优先 `--frozen-lockfile`，失配回退普通安装）。

v5 的 `dependencies` 是「坐标 → 固定版本」，转换规则（`../specs/manifest/v3.md` §5）：

| manifest 形式 | 转换后 `package.json` |
| --- | --- |
| `"dsh-pet": "0.2.0"` | `"dsh-pet": "0.2.0"` |
| `"github:owner/repo": "<sha>"` | `"repo": "github:owner/repo#<sha>"` |
| `"github:owner/repo#path:/pkg": "<sha>"` | `"pkg": "github:owner/repo#<sha>&path:pkg"` |

## 3.4 对账：`reconcileProfile` 三态

安装后校验层栈（Bundle 的唯一判据是 `dsh.bundle.patch` 字段存在）：

| 结果 | 条件 | 处理 |
| --- | --- | --- |
| **missing** | 层栈中「同时是依赖」的包，装完无 `dsh.bundle.patch` 声明 | 报错 + **删除整个 profile 回滚** |
| **added** | 依赖声明了 `dsh.bundle.patch` 却没进层栈 | 自动补进 `bundles` 并写回 |
| 模板型 bundle | 仅在 `bundles`、不在 `dependencies` | 跳过验证，DSH 安装目录 fallback 兜底 |

## 3.5 `files[]` / `skills[]` 下载 + 回滚

- 逐个下载到 `path`（profile 形态落 profile 根内，dshhome 形态落 `$DSH_HOME` 内）；
- 每项校验 `sha256` + `size`，`urls[]` 多镜像依次尝试；
- **任一失败 → 删除已下文件并整体回滚**；
- 常规插件走 npm / git（`dependencies`），不进 `files[]` / `skills[]`。

## 3.6 向后兼容（次要）

新启动器可**只实现 §3.2 主线**，旧包按下表分派：

| 归档 | 容器标记 | `manifestVersion` | `type` | 动作 |
| --- | --- | --- | --- | --- |
| `.tgz` | 无 `dspack.json`（扁平 tar.gz） | 1 | — | **拒绝** + 降级提示（已废弃） |
| `.tgz` | 同 | 2 | — | 兼容导入（层栈契约） |
| `.tgz` | 同 | 3 | — | 兼容导入（可复现层栈契约） |
| `.dspack` | `version:2` | 4 | `profile` | 兼容导入（`overrides/`→profile 根 + `files[]`） |
| `.dspack` | `version:3` | 5 | `profile` / `dshhome` | **现行**（§3.2） |

各历史版本一句话：

- **v1**：压平 `plugins[]`，无法表达加载语义 → **拒绝** + 降级展示。
- **v2**（`.tgz`）：`bundles` / `dependencies` / `patch` 三分离；`dependencies` 是「键=包名、值=pnpm spec **原样透传**」，`dshVersion` 是 semver 范围取下限。
- **v3**（`.tgz`）：v2 + 可复现化（`dependencies` 改「坐标 → 固定版本」、`dshVersion` 改精确版本、`displayName`/`description` 多语言）。
- **v4**（`.dspack` v2）：v3 + `type`（`profile`）+ 可选 `files[]`；用户文件进 `overrides/` → profile 根，**无 `home/`**。

历史版本安装执行与 §3.2.2 profile 形态**同构**，差异只在上面列的语义点。

---

## 4. 安全过滤（安装侧）

`overrides/` / `home/` 内容打包时已经过过滤（`../specs/pack-structure/v1.md` §5 + v3 §8），但**导入侧仍应做防御式校验**：拒绝落盘 `.env` / 密钥（`.key` `.pem` `.p12`…）/ `credentials.yaml` / SSH 私钥 / 嵌套压缩包等，防止恶意包写敏感文件。

---

## 5. 实现坑与已知边界

1. **编码 BOM 坑**：机器文件误写 UTF-8 BOM 会让安装端 `JSON.parse` 崩；导出侧用 utf8NoBOM。
2. **路径分隔符统一 `/`**：归档内 `rel` 一律 `/`，对照逻辑依赖这一点。
3. **符号链接整体丢弃**：不做解引用。
4. **隔离不污染源目录**：打包在内存拼 ZIP（不落临时目录）、全程只读源目录；导入用系统临时目录解包，结束清理。
5. **v2 判定靠 `plugins` 字段**：早期 `validateManifest` 靠 `plugins` 存在性判 v1/v2、不读 `manifestVersion`（`../specs/manifest/v2.md` §8.1）。现代实现应统一以 `manifestVersion` 为准。
6. **`home/` 与 `overrides/` 语义一致**：都是文件级复制替换，不做字段级合并；合并式覆盖由 `cordis.patch.yml`（Cordis 补丁层）承担。

---

## 6. 伪代码骨架

```ts
/* 导出（三入口，实现于 packages/core/src/pack.js / repo.js） */
async function packProfile(host, profile, opts) {                      // dspack pack
  const scan = await scanProfile(host, profile.dir);                  // 扫描 + 五类安全过滤
  const manifest = await buildManifest(host, profile, opts, scan);    // v5 type:"profile"（含坐标钉定）
  const entries = {};
  for (const f of selectFiles(scan.files, opts.include)) entries[dspackEntryPath(f.rel)] = await host.readFile(f.abs);
  for (const f of selectedHome(opts)) entries[`home/${f.rel}`] = await host.readFile(f.abs);  // home/ 上一级内容
  entries['manifest.json'] = entries['dspack.json'] = /* 最后写入，覆盖残留 */;
  const dspack = buildDspack(entries);                                // 内存 zipSync，无暂存区
  return { manifest, output, sha256: sha256hex(dspack), size: dspack.length };
}

async function packHome(host, home, opts) {                           // dspack pack-home
  const summary = summarizeHome(files);                               // 四类单元识别（去 web/headless）
  const manifest = await buildHomeManifest(host, home, { ...opts, ...summary });
  /* 组装同上，overrides/ 按 $DSH_HOME 相对路径平铺 → 写 <name>-<version>.dspack */
}

async function exportRepo(host, profile, opts) {                      // dspack pack --repo
  const pack = await packProfile(host, profile, { ...opts, out: releaseDir });
  await host.writeTextFile(`${releaseDspack}.sha256`, `${pack.sha256}  ${releaseDspack}\n`);  // 侧车在这里写
  manifest.sha256 = pack.sha256;                                      // 顶层 sha256，仅源仓库形态
  /* 写 manifest.json / README / .dspackignore / .gitignore → git init + add + commit */
}

/* 市场 */
async function market(entry: IndexEntry): Promise<Buffer> {
  const buf = await download(entry.downloadUrl);
  if (buf.length !== entry.size) throw new Error('size mismatch');
  if (sha256hex(buf) !== entry.sha256) throw new Error('sha256 mismatch');
  return buf;                                       // 交给导入
}

/* 导入 */
async function install(buf: Buffer, m?: Manifest) {
  if (buf[0] === 0x50 && buf[1] === 0x4b) {        // 'PK'
    const marker = readRootJson(buf, 'dspack.json');
    if (!marker || marker.format !== 'dspack') throw new Error('not a dspack');
    if (marker.version !== 3) return installLegacy(buf);
  }
  m ??= readRootJson(buf, 'manifest.json');
  if (m.manifestVersion === 1) return { ok: false, reason: 'deprecated v1' };
  if (m.manifestVersion !== 5) return installLegacy(buf);
  return m.type === 'dshhome' ? installDshhome(buf, m) : installProfile(buf, m);
}

async function installDshhome(buf, m) {
  assert(m.profiles && !('web' in m.profiles) && !('headless' in m.profiles));
  assert(m.profiles[m.defaultProfile]);
  await ensureDshInstalled(m.dshVersion);
  const home = createInstanceHome();
  for (const [name, unit] of Object.entries(m.profiles)) {
    await extractTo(home, `overrides/profiles/${name}/`);
    await rebuildAndInstall(unit);                  // dependencies → package.json → pnpm install
    await reconcile(unit);                          // missing → 回滚删 profile
  }
  await extractHomeOverrides(home, m);              // .agent-presets/ skills/ AGENTS.md data/
  await downloadFiles(home, m.files, m.skills);     // 任一失败 → 整体回滚
  setDefaultProfile(m.defaultProfile);
}
```

---

## 7. 速查表

| 维度 | 判断依据 | 取值 |
| --- | --- | --- |
| 三职责 | CLI 命令 | `pack` `pack-home` `pack --repo`（导出）/ `list` `view` `market`（市场）/ `install`（导入） |
| 容器 | 文件头 / 根 `dspack.json` | `.dspack` v3（现行）/ `.dspack` v2 / `.tgz` |
| manifest | 根 `manifest.json` 的 `manifestVersion` | 5（现行）/ 4 / 3 / 2 / 1（拒绝） |
| 形态 | v5 的 `type` | `profile` / `dshhome`（`collection` 拒绝） |
| 覆盖落点 | 容器 + 形态 | profile 根 / `$DSH_HOME` 根 |
| 依赖语义 | manifest 版本 | v3+ 坐标→固定版本；v2 透传 spec |
| 重内容 | `files[]` / `skills[]` | 指针 + sha256 + size + urls[] |
