# 启动器注册表（launcher-registry）

> 状态：**现行**。本表是 manifest v5 r2 `launchers` 字段（`../manifest/v5.md` §14）引用的**启动器 canonical ID 认领表**。安装端判定规则见 `../pack-structure/v3.md` §8.4。
>
> **机器可读版本**：`https://dsh-packforge.github.io/dsh-pack-market/launchers.json`（schemaVersion 1，与本表同步发布；第三方如 DSHL 的 README 可直接引用）。消费纪律：拉取失败 / 结构非法时必须回落内置清单，不得阻断安装或编辑流程。

---

## 1. ID 一览

| ID | 启动器 / 桌面端 | 版本自报 |
| --- | --- | --- |
| `dshl` | PCL-Deepseek-Harness-Launcher（PCL2 魔改） | ✅ 四段式（如 `0.1.1.2`） |
| `hdsl` | [HDSL · Hello DeepSeek Launcher](https://github.com/MCXCC303/HDSL)（HMCL 内核 JavaFX；`.dspack` 市场采纳者，另维护自有 `.hdslp` 格式） | ✅ |
| `dsh-packforge-app` | DSH PackForge GUI / `dspack` CLI | ✅ |
| `official-desktop` | DeepSeek Harness 官方桌面端 | ✅ |
| `dsh-cli` | 裸 `dsh` 命令行（无启动器） | ✅（`dsh --version`） |

> ⚠️ **防混淆**：`dshl`（PCL 系）与 `hdsl`（Hello 系）一个字母之差、完全无关，引用与文档书写时务必核对。

> 待认领：第三方启动器按 §2 流程 PR 追加。

## 2. 命名与认领规则

1. ID 为 **kebab-case 全小写**，认领后**稳定不变**；废弃则标记 deprecated，ID 不复用；
2. **认领**：向本仓库提 PR，在本表追加一行——`ID + 显示名 + 仓库链接 + 版本自报方式`；
3. manifest `launchers` 的 key 未注册 → 校验器**警告不拒绝**（防拦截新生态，但提示作者去认领）；
4. 安装器对不认识的自身 ID（注册表查无）按「未列出」三态处理（`../pack-structure/v3.md` §8.4）；
5. 启动器须**自报版本号**（用于 `minVersion` 判定）；无法自报时，安装器按「版本未知」轻提示放行。

## 3. 版本比较规则

`minVersion` 判定统一使用：按 `.` 分段、逐段**数值**比较、缺段视为 `0`；**不做** rc 等预发布语义（启动器场景用不到，避免过度设计）。

示例：`0.1.1.2 ≥ 0.1.1`（后者缺段补 `0`，即 `0.1.1.0`）→ 满足。

## 4. 与 `launchers` 字段的关系速查

| manifest 写法 | 含义 |
| --- | --- |
| `{ "dshl": true }` | 声明支持 DSHL（白名单模式：其他启动器装时轻提示） |
| `{ "dshl": "0.1.1.2" }` | 支持 DSHL 且要求 ≥ 0.1.1.2 |
| `{ "official-desktop": false }` | 与官方桌面端冲突（纯黑名单模式：其余启动器静默安装） |
| `{ "official-desktop": { "supported": false, "reason": "…" } }` | 全式：冲突 + 警告文案 |
