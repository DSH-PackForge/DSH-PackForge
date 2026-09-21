# DSH-PackForge

DSH 整合包平台 · **规范仓库**。

像玩 Minecraft 整合包一样，**一键导出、分享、安装** DSH AI 智能体配置包。本仓库定义「整合包（modpack）」的全部格式标准：

- `specs/manifest/` —— 包内 `manifest.json` 的契约（版本演进 v1 → v2 → v3 → v4 → v5）；
- `specs/pack-structure/` —— 包结构：`.tgz`（v1，历史）→ `.dspack`（v2，历史）→ `.dspack` v3（现行）；
- `specs/index/` —— `index.json` 索引契约（schemaVersion 2：精简指针制 + packs/ 懒加载）；
- `specs/publishing/` —— 发布契约：仓库创建 + Release 发布 + `dsh-pack` 收录。
- `specs/workspace-config/` —— `.dshpkcfg` 导出工作区快照（导出参数持久化，不打包）。

## 生态

| 仓库 | 职责 |
|---|---|
| **[DSH-PackForge](https://github.com/DSH-PackForge/DSH-PackForge)**（本仓库）| 格式规范：manifest / pack-structure / index / publishing |
| **[dsh-packforge-app](https://github.com/DSH-PackForge/dsh-packforge-app)** | 图形化管理工具（Electron GUI + DSH 插件）+ CLI：`dspack list / pack / view / install / market` |
| **[dsh-pack-market](https://github.com/DSH-PackForge/dsh-pack-market)** | 市场仓库：`index/index.json` 索引 + `index/packs/` 懒加载源 + GitHub Pages 市场页 |
| **[all-about-whales](https://github.com/DSH-PackForge/all-about-whales)** | 端到端参考实现：manifest v5 + `.dspack` v3 示例整合包 |

内容流水线：**dsh-packforge-app（dspack CLI）打包** → **dsh-pack-market 分发** → **dspack install / dsh 启动器导入**。

## 目录结构

```
DSH-PackForge/
├── specs/
│   ├── manifest/                  # manifest.json 契约
│   │   ├── v1.md                  # 已废弃（压平的 plugins[]）
│   │   ├── v2.md                  # 历史（层栈契约；启动器仍兼容导入）
│   │   ├── v3.md                  # 历史（可复现的层栈契约；仍兼容导入）
│   │   ├── v4.md                  # 历史（v3 + type + files[]）
│   │   └── v5.md                  # ★ 现行（统一：type profile / dshhome）
│   ├── pack-structure/            # 包结构
│   │   ├── v1.md                  # 历史（L1：单 .tgz 布局 / 扫描 / 安全过滤 / 打包 / 安装）
│   │   ├── v2.md                  # 历史（.dspack：纯 ZIP + dspack.json + overrides）
│   │   └── v3.md                  # ★ 现行（统一：profile + home/ 与 dshhome 两形态）
│   ├── index/
│   │   └── index.md               # ★ 现行（index.json 索引契约，schemaVersion 2）
│   ├── publishing/
│   │   └── v1.md                  # ★ 现行（仓库创建 + Release 发布 + 收录契约）
│   └── workspace-config/
│       └── v1.md                  # ★ 现行（.dshpkcfg 导出工作区快照）
├── notes/                         # 实现备忘（非 spec）
│   └── windows-preview-handler.md # .dspack 预览/缩略图处理器开发备忘
├── docs/                          # 实现指南（非 spec）
│   └── launcher-integration.md    # 启动器集成指南（导出 pack / 市场 market / 导入 install 三职责）
├── examples/                      # 示例包（预留）
├── LICENSE                        # MIT
└── README.md
```

## 规范一览

| 规范 | 状态 | 一句话 |
|---|---|---|
| `specs/manifest/v5.md` | **现行** | 统一版本：`type:"profile"`（单 profile）或 `type:"dshhome"`（多 profile + preset / skill / 指令） |
| `specs/manifest/v4.md` | **历史** | v3 + `type`（profile/collection 预留）+ 可选 `files[]` 下载清单 |
| `specs/manifest/v3.md` | 历史 | 依赖「坐标 → 固定版本 / commit sha」、`dshVersion` 精确、displayName 多语言，可复现；仍兼容导入 |
| `specs/manifest/v2.md` | 历史 | `bundles` / `dependencies` / `patch` 三分离层栈契约；启动器兼容导入 |
| `specs/manifest/v1.md` | 已废弃 | 压平的 `plugins[]`，无法表达加载语义；安装时拒绝 |
| `specs/pack-structure/v1.md` | 历史 | L1 单 `.tgz`：扁平 Profile 快照 + 根 `manifest.json`，四类安全过滤 |
| `specs/pack-structure/v2.md` | **历史** | `.dspack`：纯 ZIP + 根 `dspack.json` 标记 + `overrides/` + 可选 `files[]` 按需拉取 |
| `specs/pack-structure/v3.md` | **现行** | `.dspack` v3：统一 profile（`overrides/` + 可选 `home/`）与 dshhome（`overrides/` 按 `$DSH_HOME` 平铺）两形态 |
| `specs/index/index.md` | **现行** | index.json 索引契约（schemaVersion 2）：精简指针制 + `packs/<owner>.<repo>/` 懒加载完整 manifest/README |
| `specs/publishing/v1.md` | **现行** | 仓库创建 + Release 发布 + sha256 侧车 + `dsh-pack` 话题收录 |
| `specs/workspace-config/v1.md` | **现行** | `.dshpkcfg` 导出工作区快照（导出参数持久化，不打包） |

> 给启动器作者：见 [`docs/launcher-integration.md`](docs/launcher-integration.md) —— 导出（pack）、市场（market）、导入（install）三段式实现指南（现行 v5 / `.dspack` v3 为主线，历史版本向后兼容）。

## 怎么选版本

- **写新包 → v5**（`dsh-packforge-app` 产出 `.dspack` v3，manifest v5 + pack-structure v3）。
- **装旧包 → 启动器按 v4 / v3 / v2 导入**；v1 仅做拒绝与降级展示。
- **v5 `dshhome` 形态（多 profile + preset / skill / 指令）现由 `.dspack` v3 统一承载**。
- **改打包/安装结构 → `pack-structure`**，与 `dsh-packforge-app` 的格式演进对齐（L1 `.tgz` → L2 `.dspack` → L3 重内容 `files[]` 按需拉取）。

## 启动器徽章

启动器可在 README 挂徽章，声明对 DSH-PackForge 整合包的支持。徽章由 `dsh-pack-market` 生成并部署在 GitHub Pages（把 `zh` 换成 `en` 即英文版）。

**支持徽章**（声明「这是一个支持 DSH-PackForge 整合包的启动器」）：

```markdown
[![支持 DSH-PackForge 整合包](https://dsh-packforge.github.io/dsh-pack-market/badges/launchers/dsh-packforge-support-zh.svg)](https://dsh-packforge.github.io/dsh-pack-market/)
```

**版本徽章**（声明支持的具体规范版本，文件名统一为）：

```
https://dsh-packforge.github.io/dsh-pack-market/badges/versions/manifest-v{1..5}-{zh,en}.svg   # 清单（manifest）
https://dsh-packforge.github.io/dsh-pack-market/badges/versions/pack-v{1..3}-{zh,en}.svg       # 结构（pack-structure）
```

| 版本 | 状态 |
|---|---|
| `manifest v5` / `pack v3` | **现行** |
| `manifest v4 / v3 / v2`、`pack v2 / v1` | 历史（仍兼容导入） |
| `manifest v1` | 已废弃（安装时拒绝，一般不挂） |

示例（支持最新规范的启动器 README 顶部）：

```markdown
[![支持 DSH-PackForge 整合包](https://dsh-packforge.github.io/dsh-pack-market/badges/launchers/dsh-packforge-support-zh.svg)](https://dsh-packforge.github.io/dsh-pack-market/)
![清单 v5](https://dsh-packforge.github.io/dsh-pack-market/badges/versions/manifest-v5-zh.svg)
![结构 v3](https://dsh-packforge.github.io/dsh-pack-market/badges/versions/pack-v3-zh.svg)
```

## 参与修订

1. 在 `specs/` 下新增或修改对应版本文档；
2. manifest 变更需与 **dsh-packforge-app（生成/导入侧）** 和 **dsh-pack-market（索引/收录侧）** 实现同步对齐；
3. 每篇文档开头标注**状态**（现行 / 历史 / 已废弃）；历史版本**只标记、不删除**；
4. 重大变更走 PR + 评审。

## License

[MIT](LICENSE) © 2026 DSH-PackForge
