# 启动器注册表（launcher-registry）

> 状态：**现行**。本表是 manifest v5 `launchers` 字段（`manifest/v5.md` §13）引用的**启动器 canonical ID 认领表**；安装端判定规则见 `pack-structure/v3.md` §9.1。
>
> **机器可读版本**：`https://dsh-packforge.github.io/dsh-pack-market/launchers.json`（schemaVersion 1，与本表同步发布；第三方 README 可直接引用）。消费纪律：拉取失败 / 结构非法时必须回落内置清单，不得阻断安装或编辑流程。

---

## 1. ID 表

| ID | 启动器 / 桌面端 | 版本自报 |
| --- | --- | --- |
| `dshl` | PCL-Deepseek-Harness-Launcher（PCL2 魔改） | ✅ 四段式（如 `0.1.1.2`） |
| `hdsl` | [HDSL · Hello DeepSeek Launcher](https://github.com/MCXCC303/HDSL)（HMCL 内核 JavaFX） | ✅ |
| `dsh-packforge-app` | DSH PackForge GUI / `dspack` CLI | ✅ |
| `official-desktop` | DeepSeek Harness 官方桌面端 | ✅ |
| `dsh-cli` | 裸 `dsh` 命令行（无启动器） | ✅（`dsh --version`） |

> ⚠️ **防混淆**：`dshl`（PCL 系）与 `hdsl`（Hello 系）一个字母之差、完全无关，引用与书写时务必核对。

## 2. 认领规则

1. **命名**：kebab-case，尽量短、可辨识（优先项目自身的缩写）。
2. **认领方式**：向本仓库提 PR，在本表加一行（ID + 启动器名 + 版本自报方式），并附仓库链接。
3. **唯一性**：ID 全局唯一、不得与既有 ID 混淆性相似（差异仅一个字母的须在名称里写清品牌）。
4. **版本自报**：启动器**必须**能报告自身版本号（写进安装日志、用于 `minVersion` 比较）；不能自报的，`minVersion` 一律按"版本未知"处理（轻提示后放行）。

## 3. 版本比较规则

- 按 `.` 分段、**逐段数值比较**、缺段视为 `0`（`1.2` ≡ `1.2.0`）；
- **不做** `rc` / `beta` 等预发布语义（`0.1.1-rc.2` 之类只在 DSH 版本里出现，启动器版本不依赖它）；
- 比较结果只用于**提示与警告**：不满足 `minVersion` 时提示后**放行**，绝不硬拒（`pack-structure/v3.md` §9.1）。

## 4. 速查（给包作者）

```jsonc
// 简式：只声明支持哪些
"launchers": ["dshl", "hdsl", "dsh-packforge-app"]

// 全式：需要最低版本 / 负面声明时
"launchers": {
  "dshl": "0.2.0",
  "official-desktop": false,
  "dsh-cli": { "supported": false, "reason": "需要桌面端的设置界面" }
}
```

缺省（不写）= 通用包，任何启动器都按现状安装。
