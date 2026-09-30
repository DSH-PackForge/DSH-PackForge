# vendoring 与离线包 · 规范解读（v5 r2）

> 面向：**整合包作者**（怎么标、怎么写）与**启动器实现者**（怎么装）。
> 规范本体：`../specs/manifest/v5.md` §12、`../specs/pack-structure/v3.md` §8、`../specs/publishing/v1.md` §8。
> 本文是解读——与规范冲突时，以规范为准。

---

## 1. 一句话心智模型

**依赖的来源由键唯一确定，没有「看情况」。**

```
裸坐标        "zustand": "4.5.0"                   → 只从网络
vendor: 前缀  "vendor:dsh-pet": "0.3.0"            → 只从包内（vendored{} 必有条目）
闭包条目      vendored{} 里的 "name@version"        → 在 vendored{} 里 = 包内；否则网络
```

一个包可以两类并存。**离线包**只是「全部直接依赖都带前缀」的特例，不是必须声明的模式。

## 2. 三类条目

### 2.1 裸坐标（只从网络）

```jsonc
"dependencies": {
  "zustand": "4.5.0",                                  // npm
  "github:DViridescent/dafy-whale-theme": "99e8c57"     // git（值是 commit sha）
}
```

### 2.2 `vendor:<包名>`（只从包内，不记来源）

```jsonc
"dependencies": {
  "vendor:dsh-pet": "0.3.0",
  "vendor:my-private-plugin": "1.0.0"
},
"vendored": {
  "vendor:dsh-pet": { "version": "0.3.0", "sha256": "…", "size": 123,
                      "path": "vendor/npm__dsh-pet/dsh-pet-0.3.0.tgz", "reason": "upstream-missing" },
  "vendor:my-private-plugin": { "version": "1.0.0", "sha256": "…", "size": 123,
                      "path": "vendor/npm__my-private-plugin/my-private-plugin-1.0.0.tgz", "reason": "unpublished" }
}
```

- 键上**只有包名**：不写 owner/repo、不写 commit sha（内嵌后来源不参与安装）；
- 值 = **包版本**，与 `vendored[].version`、tarball 内 `package.json` 的 `version` **三处一致**；
- **老启动器**遇到这个键名 → pnpm 判非法包名 → **响亮失败**（它兑现不了「只从包内」；要服务老启动器就另发不含前缀的版本）；
- `reason`：`upstream-missing`（上游已消失）/ `unpublished`（从未发布）/ `local-modified`（本地魔改）/ `explicit`（为离线的显式内嵌）。

### 2.3 闭包条目（传递依赖）

```jsonc
"vendored": {
  "esbuild@0.25.12": { "kind": "closure", "name": "esbuild", "version": "0.25.12",
                       "sha256": "…", "size": 456, "path": "vendor/npm__esbuild/esbuild-0.25.12.tgz" }
}
```

- 键 = `name@version`（同名多版本会并存，必须带版本），`kind` / `name` 必填；
- 必须能在随包 `pnpm-lock.yaml` 的 `packages:` / `snapshots:` 里找到——这是**防夹带**校验（闭包条目不是直接依赖，最容易被塞私货）；
- 要「整包离线」就必须收齐闭包（`full` 档沿 lockfile 收）。

## 3. 怎么选（决策树）

```
依赖全部健在、也不要求固定字节?
├─ 是 → 什么都不写（纯在线，现状）
└─ 否 ↓
   只有个别依赖会消失 / 要固定魔改版?
   ├─ 是 → 给那几个坐标加 vendor: 前缀（其余裸键）
   └─ 目标机器无网 / 要做死上游存档?
       └─ 是 → 全部直接依赖加前缀 + 收齐闭包（离线包）
```

## 4. 安装器视角（本地化，两支）

```
阶段 0 预检：vendored 键 = dependencies 键（含前缀）逐字对账；裸键出现在 vendored{} → 拒装；
             逐 tarball sha256 + size 校验；读 tarball 内 package.json 核对 name / version
阶段 1 落盘：解 vendor/ 到 profile 的 vendor-blobs/；写 package.json 与 pnpm-lock.yaml（下表两支）
阶段 2 安装：pnpm install --frozen-lockfile --trust-lockfile
             （全部直接依赖带前缀时可加 --offline；包内 pnpm-workspace.yaml 写 minimumReleaseAge: 0）
```

| 支 | 触发 | 改什么 |
| --- | --- | --- |
| **A · npm 来源** | lockfile 节点是 registry 版本（`pkg@1.2.3`） | `package.json` 写 `file:./vendor-blobs/<file>.tgz`；lockfile **四处同步**：importer 的 `specifier` + `version`、`packages` 键、`resolution`、`snapshots` 键 |
| **B · git 来源** | 节点含 `gitHosted` / codeload URL | 只改 `resolution.tarball` → 本地文件；`gitHosted` 与 `integrity` 不动；`package.json` 的 spec 从 lockfile importer 复制 |
| 闭包条目 | 在 `vendored{}` 里 | 同支 B（只改 `resolution.tarball`） |

- **不用 `--prefer-offline`**：它表达「本地优先、否则联网」，与「来源由键确定」矛盾；
- 只改一处会报 `ERR_PNPM_OUTDATED_LOCKFILE`（importer 与 `package.json` 的 specifier 必须一致）；
- **不要试图「预填充 pnpm store」**：`file:` 预热出的条目不带 integrity，registry 风格解析仍报 `ERR_PNPM_NO_OFFLINE_TARBALL`；
- 安装后 `vendor-blobs/` 属已安装状态，**不可清理**（profile 的 `package.json` / `pnpm-lock.yaml` 引用它）。

完整导入流程（阶段划分、回滚）见 `../specs/manifest/v5.md` §11。

## 5. 常见误区

| 误区 | 事实 |
| --- | --- |
| 「裸键的依赖也会用包内副本」 | 不会。裸键 = **只从网络**；要包内就得加 `vendor:` 前缀 |
| 「`vendor:` 键要写来源（`vendor:github:owner/repo`）」 | 不需要也不允许——内嵌后来源不参与安装与定位（定位按 tarball 内 `name@version` 认 lockfile 节点） |
| 「预填充 store 就能离线」 | 不行（`ERR_PNPM_NO_OFFLINE_TARBALL`）；必须改 lockfile 的取件地址 |
| 「`package.json` 写 `file:` 就够了」 | 不够——lockfile 的 importer / packages / snapshots 必须同步，否则 `ERR_PNPM_OUTDATED_LOCKFILE` |
| 「魔改必须改版本号」 | 不必（早期的版本后缀约定已撤销为**可选**）；`vendor:` 前缀已保证老启动器响亮失败 |
| 「老启动器也能装离线包」 | 不能。全部带前缀 = 全部非法包名 → 整包装不上；须声明 `launchers` 白名单并在详情页写明 |
| 「`vendored` 可以写任意 `dependencies` 没有的坐标」 | 只有**闭包条目**可以（键 `name@version`，且须 ∈ 随包 lockfile）；**裸键**出现在 `vendored{}` 一律拒装 |
| 「重内容也该走 `vendored`」 | 不。插件依赖走 `vendored`；非依赖重内容走 `files[]`，两通道不混用 |
| 「`dependencies` 值直接用 `file:` 代替 `vendored{}`」 | 不可——闭包条目无资格进 `dependencies`（离线需求落空），且丢 `sha256` 装前预验与 `reason` 元数据 |

## 6. 与 `files[]` 的分工

| | `vendored` + `vendor/` | `files[]` |
| --- | --- | --- |
| 装什么 | **插件依赖**（`dependencies` 里的坐标） | **任意重内容**（模型、数据、大资源） |
| 落点 | `node_modules`（经本地化两支） | manifest 声明的 `path` |
| 云端形态 | registry / git | `urls[]` 指针下载 |
| 本地形态 | 包内 `vendor/*.tgz` | （无——通常走指针） |

## 7. 发布注意（`../specs/publishing/v1.md` §8）

- 体积：> 500 MB 建议标注；GitHub Release 单资产上限 2 GiB；
- license：内嵌即再分发，逐一确认；
- 含 `vendor:` 键的包**必须**声明 `launchers` 白名单，并在 README / 详情页注明「N 个依赖为包内副本」；
- 离线包发版前以离线模式 dry-run 试装一次（确认闭包无遗漏）。
