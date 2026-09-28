# DSH-PackForge 协议

> 当前发布标签:**`protocol-2026-09`**(本文件永远描述当前状态;历史快照见 git tag)。

DSH 整合包的开放协议:manifest 格式、`.dspack` 容器布局、市场索引、发布礼仪与启动器互操作。由 DSH-PackForge 组织治理;采纳者包括 `dsh-packforge-app`(GUI/CLI)、HDSL(Hello DeepSeek Launcher)、DSHL(PCL-Deepseek-Harness-Launcher)、`dsh-pack-market`、官方桌面端等。

**本文件是伞文档**:只做导航与治理,不复述内容——规范本体永远在 `specs/` 各篇中,此处一律引用路径。

---

## 1. 组件地图与门禁速查

全协议唯一的版本速查表。**你的实现要检查的门禁值**:

| 组件 | 规范文件 | 门禁号(现行) | 管什么 |
|---|---|---|---|
| Manifest | [`specs/manifest/v5.md`](specs/manifest/v5.md) | `manifestVersion: 5`(r2) | 包元数据、依赖、`dshVersions`、`vendored{}`、`launchers`、导入行为 |
| 容器 / 打包与安装 | [`specs/pack-structure/v3.md`](specs/pack-structure/v3.md) | dspack `version: 3`(r2) | `.dspack` 布局、安全过滤、`vendor/` 目录、打包端义务(§8.6)与统一安装算法(§8.3)、方言互操作(§8.5) |
| 市场索引 | [`specs/index/index.md`](specs/index/index.md) | `schemaVersion: 2` | 市场条目格式与采集行为 |
| 发布礼仪 | [`specs/publishing/v1.md`](specs/publishing/v1.md) | —(编辑性) | 体积/license/存档审计/离线包须知(§8) |
| 工作区配置 | [`specs/workspace-config/v1.md`](specs/workspace-config/v1.md) | —(隐式 v1) | `.dshpkcfg` 导出参数快照(工具本地,不进包) |
| 启动器注册表 | [`specs/launcher-registry.md`](specs/launcher-registry.md) | —(PR 认领) | 启动器 canonical ID(`dshl`/`hdsl`/`dsh-packforge-app`/`official-desktop`/`dsh-cli`);机器可读版 [launchers.json](https://dsh-packforge.github.io/dsh-pack-market/launchers.json)(schemaVersion 1,拉取失败回落内置清单) |
| Vendoring 解读 | [`docs/vendoring-guide.md`](docs/vendoring-guide.md) | —(非规范) | 两种覆盖形态、决策树、常见误区 |

> 门禁号是**部署事实**:各消费者代码硬检查这些值(如 dsh-packforge-app 拒收 `manifestVersion` ∉ {4,5})。协议演进绝不移动已部署的门禁号——见 §2。

## 2. 版本治理(r 修订体系)

- **增量变更 → 文档修订 r+1**(如「v5 r2」):只做加法——新增可选字段、放宽约束、补充行为规范。协议号(`manifestVersion` 等)**永不移动**。
- **破坏性变更 → 协议号 +1,消费侧先行**:先推动消费者同时支持新旧两个号,再产出新号的包;绝不允许只改规范不改消费者。
- **文件树 = 门禁值映射**:`specs/manifest/v1–v5` 一个版本号一个文件,查证时直接翻到对应文件——这就是规范按版本分文件的原因,也是不要合并成单文档的原因(合并时机:下一次破坏性变更的 v6 时刻)。
- **展示型字段可拓宽,行为型字段只能加新**:枚举展示值(如 `reason` 的取值)后续可加;决定安装行为的字段语义冻结。
- **派生属性不声明**:能从数据算出来的(如「离线可装」= vendored 覆盖传递闭包)就不设声明字段——派生属性不可能说谎,声明字段需要校验器防说谎。案例:`offline`/`vendorMode` 字段在 r2 定稿前被删除。
- **修订记录**:

| 批次 | 日期 | 内容 |
|---|---|---|
| v5 r2 / v3 r2 | 2026-09 | vendoring 机制、`dshVersions`、`launchers`、四阶段导入、§8.6 打包端义务、方言互操作、launcher-registry(需求来源 [issue #3](https://github.com/DSH-PackForge/DSH-PackForge/issues/3)) |
| HDSL 调研吸收 | 2026-09 | 探测三态(UNKNOWN → 保守内嵌)、`hdsl` ID 注册 |
| workspace-config 增补 | 2026-09 | `.dshpkcfg` 新增 `dshVersions` / `vendor` / `launchers` 旋钮 |

## 3. 角色路由

| 你是谁 | 读这些 |
|---|---|
| 启动器 / 安装器作者 | manifest §11–§14(导入行为、dshVersions、launchers)+ pack-structure §8(vendoring/统一算法/方言)+ §10(安装端行为)+ launcher-registry(认领 ID) |
| 包作者 | manifest §2–§6(字段与约束)+ [vendoring-guide](docs/vendoring-guide.md)(选型)+ publishing(发布礼仪) |
| 市场维护者 | index(条目格式)+ publishing §8(离线包须知)+ 采集器行为 |
| 新来的 | 本文件 §1–§2 + [README](README.md) 特性徽章,然后按角色进 §3 |

## 4. 互操作边界(生态事实)

- **正式语法**:`vendored{}` + `vendor/` 目录(manifest v5 r2 / pack-structure v3 r2)。
- **消费宽容、生产禁止**:PCL-DSHL 的 `vendor:<file>.tgz` 值方言 + `vendor/vendor.json`——安装器**应当**能消费(按隐式 vendored 条目处理),生成器**禁止**产出。
- **独立格式**:HDSL 的 `.hdslp` 自有格式(与 `.dspack` 平行,非方言);其导出侧三态探测已反哺本协议(§8.6)。
- ⚠️ `dshl`(PCL 系)与 `hdsl`(Hello 系)一个字母之差,完全无关。
- **老启动器物理极限**(诚实声明):不识 r2 字段的启动器遇死上游依赖会失败——协议的义务是「不让任何人变得更差」,不是「给旧设备新能力」;缓解靠 `launchers` 白名单声明 + 市场详情页 + README 带外告知。

## 5. TL;DR

一个合法 `.dspack` = ZIP 容器(dspack v3)+ `manifest.json`(v5,未知字段必须忽略;除 `name`/`version`/`manifestVersion`/`dshVersion` 等少数必填外大量可选)+ 依赖即 npm/git 坐标(`dependencies` 值为精确版本或 commit sha)+ 可选 `vendor/` tarball 目录(`vendored{}` 哈希清单逐一校验)。

安装器最低要求 = 校验 `manifestVersion` ∈ 支持集 → 按 [manifest §11](specs/manifest/v5.md) 四阶段导入(预检 → 落盘 → 运行时与依赖 → 收尾)→ 任何阶段失败全量回滚。统一安装算法:`vendor/` 存在 → 预填充 pnpm store → `pnpm install --prefer-offline`,一条路径无模式开关。
