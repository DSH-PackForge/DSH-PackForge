# DSH-PackForge

DSH 整合包平台 · **规范仓库**。

像玩 Minecraft 整合包一样，**一键导出、分享、安装** DSH AI 智能体配置包。本仓库定义「整合包（modpack）」的全部格式标准：

- `specs/manifest/` —— 包内 `manifest.json` 的契约（版本演进 v1 → v2 → v3 → v4）；
- `specs/pack-structure/` —— 包结构：`.tgz`（v1，现行）→ `.dspack`（v2，未来）；
- `specs/index/` —— `index.json` 索引契约（指针制 + 元数据平铺）。

## 生态

| 仓库 | 职责 |
|---|---|
| **[DSH-PackForge](https://github.com/DSH-PackForge/DSH-PackForge)**（本仓库）| 格式规范：manifest / pack-structure / index |
| **[ModPack-CLI](https://github.com/DSH-PackForge/ModPack-CLI)** | CLI：`modpack create / pack / publish / install / index` |
| **[dsh-pack-market](https://github.com/DSH-PackForge/dsh-pack-market)** | 市场仓库：`index/index.json` 索引 + GitHub Pages 市场页（由 ModPack-Index + ModPack-Web 合并而来） |

内容流水线：**ModPack-CLI 打包** → **dsh-pack-market 分发** → **modpack install 安装**（dsh 启动器兼容导入）。

## 目录结构

```
DSH-PackForge/
├── specs/
│   ├── manifest/                  # manifest.json 契约
│   │   ├── v1.md                  # 已废弃（压平的 plugins[]）
│   │   ├── v2.md                  # 历史（层栈契约；启动器仍兼容导入）
│   │   ├── v3.md                  # ★ 现行（可复现的层栈契约）
│   │   └── v4.md                  # 未来（v3 + type + files[]）
│   ├── pack-structure/            # 包结构
│   │   ├── v1.md                  # ★ 现行（L1：单 .tgz 布局 / 扫描 / 安全过滤 / 打包 / 安装）
│   │   └── v2.md                  # 未来（.dspack：DSPK 头 + ZIP + overrides）
│   └── index/
│       └── index.md               # ★ 现行（index.json 索引契约，schemaVersion 1）
├── notes/                         # 实现备忘（非 spec）
│   └── windows-preview-handler.md # .dspack 预览/缩略图处理器开发备忘
├── examples/                      # 示例包（预留）
├── LICENSE                        # MIT
└── README.md
```

## 规范一览

| 规范 | 状态 | 一句话 |
|---|---|---|
| `specs/manifest/v4.md` | **未来** | v3 + `type`（profile/collection 预留）+ 可选 `files[]` 下载清单 |
| `specs/manifest/v3.md` | **现行** | 依赖「坐标 → 固定版本 / commit sha」、`dshVersion` 精确、displayName 多语言，可复现 |
| `specs/manifest/v2.md` | 历史 | `bundles` / `dependencies` / `patch` 三分离层栈契约；启动器兼容导入 |
| `specs/manifest/v1.md` | 已废弃 | 压平的 `plugins[]`，无法表达加载语义；安装时拒绝 |
| `specs/pack-structure/v1.md` | **现行** | L1 单 `.tgz`：扁平 Profile 快照 + 根 `manifest.json`，四类安全过滤 |
| `specs/pack-structure/v2.md` | **未来** | `.dspack`：ZIP 内核 + `DSPK` 头 + `overrides/` + 可选 `files[]` 按需拉取 |
| `specs/index/index.md` | **现行** | index.json 索引契约：指针制（`downloadUrl` + `sha256` + `size`）+ 元数据平铺 |

## 怎么选版本

- **写新包 → v3**（ModPack-CLI 产出；启动器同时兼容导入 v2）。
- **装旧包 → 启动器按 v2 导入**；v1 仅做拒绝与降级展示。
- **v4 与 `.dspack` 尚处「未来」阶段**，等实现落地后再从 v3 / `.tgz` 切换。
- **改打包/安装结构 → `pack-structure`**，并与 `ModPack-CLI/docs/architecture.md` 的 L1 → L3 演进对齐。

## 参与修订

1. 在 `specs/` 下新增或修改对应版本文档；
2. manifest 变更需与 **ModPack-CLI（生成侧）** 和 **dsh 启动器（导入侧）** 实现同步对齐；
3. 每篇文档开头标注**状态**（现行 / 历史 / 已废弃）；历史版本**只标记、不删除**；
4. 重大变更走 PR + 评审。

## License

[MIT](LICENSE) © 2026 DSH-PackForge
