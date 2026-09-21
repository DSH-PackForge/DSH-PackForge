# 启动器集成指南（Launcher Integration Guide）

> 定位：**实现指南（非 spec）**。启动器（`dsh-packforge-app` / `dspack` CLI）承担整合包全生命周期的**三个独立职责**——**导出（`pack`）、市场（`market`）、导入（`install`）**。本文分三部分分别讲清，边界互不混淆。
>
> 权威契约：`../specs/manifest/`、`../specs/pack-structure/`、`../specs/index/`、`../specs/publishing/`。本文只做串联 + 实现要点，**冲突时以 spec 为准**。
>
> 主线：现行标准（manifest v5 + `.dspack` v3）。历史版本兼容统一放到「导入」部分的末尾一节次要处理。

## 职责总览

| 职责 | CLI 命令 | 输入 → 输出 | 对应 spec |
| --- | --- | --- | --- |
| **导出** | `dspack pack` | profile / `$DSH_HOME` → `.dspack` v3 + `sha256` 侧车 | manifest v5（buildManifest）、pack-structure v3（扫描/过滤/打包）、publishing（发布） |
| **市场** | `dspack list / view / market` | 中心索引 → 列表 / 详情 / 下载好的包 | index（索引契约）、publishing（收录流程） |
| **导入** | `dspack install` | 包文件 → 可运行 profile / `$DSH_HOME` | manifest / pack-structure 的「导入行为」章节 |

三者是三个**并列、独立**的环节，由 `.dspack`（导出产物 = 导入输入）与 `index.json`（市场索引 = 下载来源）衔接，但实现上各管各的。

---

# 第一部分 · 导出（Pack）

> 目标：把本机一个 profile（或整个 `$DSH_HOME`）打成可分发、可复现的 `.dspack` v3。

## 1.1 产物

```
<manifest.name>-<manifest.version>.dspack      # 包本体（纯 ZIP）
<manifest.name>-<manifest.version>.dspack.sha256   # sha256 侧车（发布/收录用）
```

打包流程还产出两个供索引校验的字段：整包 `sha256` 与 `size`。

## 1.2 流程

1. **扫描源**：profile 形态扫单个 profile 目录；dshhome 形态扫整个 `$DSH_HOME`（`~/.dsh`，**明确排除** `~/.agents` 与项目级 `.dsh/`、`.agents/`、`AGENTS.md` / `CLAUDE.md`）。
2. **安全过滤**：四类规则（精确名 / 扩展名 / 文件名正则 / 路径正则，见 `../specs/pack-structure/v1.md` §5）+ v3 新增 home 级排除（`.credentials.yaml` / `.anonymous-user-id` / `attachments/` / `settings.yaml` / `skills/.system/` / `profiles/web` / `profiles/headless`，见 `../specs/pack-structure/v3.md` §8）。
3. **buildManifest**：生成 v5 `manifest.json`（`type: "profile"` 或 `"dshhome"`，字段见 `../specs/manifest/v5.md` §2）。**字段必须完整**：必需字段（`manifestVersion`=5 / `type` / `name` / `version`；profile 形态的 `bundles` `dependencies`；dshhome 形态的 `defaultProfile` `profiles`）一个不能少；`displayName` / `description`（多语言 map）、`author`、`icon`、`dshVersion`（精确版本）、`patch` 等可选字段也应尽量填——它们决定市场页展示与可复现性。
4. **布局组装**（`../specs/pack-structure/v3.md` §3）：
   - 根：`dspack.json`（`{ "format": "dspack", "version": 3 }`）+ `manifest.json` + 机器文件 `package.json`（profile 清单快照）/ `pnpm-workspace.yaml`（pnpm 工作区配置）/ `pnpm-lock.yaml`（锁文件）（务必一并进包，见 §1.4）；
   - profile 形态：用户文件进 `overrides/`（→ profile 根）+ 可选 `home/`（→ `$DSH_HOME` 根）；
   - dshhome 形态：用户文件按 `$DSH_HOME` 相对路径平铺进 `overrides/`。
5. **打 ZIP**：纯 ZIP（`PK` 头），根必含 `dspack.json` 与 `manifest.json`。
6. **产出三指针**：整包 `sha256` + `size`，写 `.sha256` 侧车（64 位小写 hex）。
7. **发布**（把包送上市场，见第三部分前的衔接）：按 `../specs/publishing/v1.md` 建仓 / 打 tag / 发 Release，或写 `downloadUrl` 直链。

## 1.3 打包实现约定（继承 v1 §6，通用）

- **暂存区隔离**：在系统临时目录组装，**绝不改动源 profile / `$DSH_HOME`**，结束清理；
- **`/` 分隔符**：归档内 `rel` 一律 `/`，与 Windows 反斜杠无关；
- **符号链接跳过**：不做解引用，防逃出源目录与死循环；
- **机器文件 utf8NoBOM**：Windows 下误写 UTF-8 BOM 会导致安装端 `JSON.parse` 崩；
- **重内容不进包体**：模型 / 数据 / 非 npm/git 二进制写进 manifest 的 `files[]`（dshhome 形态另可 `skills[]`），只记指针。

## 1.4 导出自检清单

- [ ] **字段完整**：`manifest.json` 必需字段齐全；`displayName` / `description`（多语言）、`author`、`icon`、`dshVersion`（精确版本）、`patch` 尽量填，缺失会让市场页展示空、`dshVersion` 回退本机最新（可复现性打折）。
- [ ] **工作区配置 + 机器文件**：`pnpm-workspace.yaml`（pnpm 工作区配置，承载 hoist / allowBuilds 等 profile 的 pnpm 设置）、`pnpm-lock.yaml`（锁文件，让导入侧 `--frozen-lockfile` 复现传递依赖）、`package.json`（profile 清单快照）一并进包。
- [ ] **机器/用户文件分离**：`package.json` 只是快照，其 `dependencies` / `dsh.profile.bundles` 由 manifest 权威重建，**不依赖快照内容**；用户文件只进 `overrides/`（+ `home/`）。
- [ ] 四类安全过滤 + v3 扩展已生效，重内容走 `files[]` 不进包体。

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

1. `overrides/` 覆盖到 profile 根（`cordis.patch.yml` 等用户文件落地）；
2. 依赖重建（§3.3，`pnpm install`）→ 对账（§3.4，`reconcileProfile`）；
3. `home/`（可选）覆盖到 `$DSH_HOME` 根（全局 skill / `.agent-presets` / `AGENTS.md`）；
4. `files[]` 重内容下载 + 校验（§3.5），失败回滚；
5. profile 名默认 `profileName`（缺省 `pack`），设为默认 profile。

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
4. **暂存区隔离**：打包 / 导入都用系统临时目录，结束清理，不污染用户目录。
5. **v2 判定靠 `plugins` 字段**：早期 `validateManifest` 靠 `plugins` 存在性判 v1/v2、不读 `manifestVersion`（`../specs/manifest/v2.md` §8.1）。现代实现应统一以 `manifestVersion` 为准。
6. **`home/` 与 `overrides/` 语义一致**：都是文件级复制替换，不做字段级合并；合并式覆盖由 `cordis.patch.yml`（Cordis 补丁层）承担。

---

## 6. 伪代码骨架

```ts
/* 导出 */
async function pack(source: Profile | DshHome): Promise<{ dspack, sha256, size }> {
  const scan = scanSource(source);                 // 扫描 + 四类安全过滤 + v3 扩展
  const manifest = buildManifest(source, scan);    // v5：type profile / dshhome
  const staging = assemble(manifest, scan);        // overrides/ (+ home/) + dspack.json v3 + manifest.json
  const dspack = zip(staging);                     // 纯 ZIP
  return { dspack, sha256: sha256hex(dspack), size: dspack.length };
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
| 三职责 | CLI 命令 | `pack`（导出）/ `list` `view` `market`（市场）/ `install`（导入） |
| 容器 | 文件头 / 根 `dspack.json` | `.dspack` v3（现行）/ `.dspack` v2 / `.tgz` |
| manifest | 根 `manifest.json` 的 `manifestVersion` | 5（现行）/ 4 / 3 / 2 / 1（拒绝） |
| 形态 | v5 的 `type` | `profile` / `dshhome`（`collection` 拒绝） |
| 覆盖落点 | 容器 + 形态 | profile 根 / `$DSH_HOME` 根 |
| 依赖语义 | manifest 版本 | v3+ 坐标→固定版本；v2 透传 spec |
| 重内容 | `files[]` / `skills[]` | 指针 + sha256 + size + urls[] |
