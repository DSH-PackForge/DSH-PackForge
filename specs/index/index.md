# 索引契约（index.json，schemaVersion 2）

> 状态：**现行**。中心索引 = 「精简指针制」：索引只保留列表/搜索/安装命令所需的最少字段，完整 manifest 与 README 拆到 `packs/<owner>.<repo>/` 目录，市场详情页/客户端点开时懒加载。
>
> 事实源：`dsh-pack-market/index/index.json`；部署时由 CI 复制为 `web/index.json`，`index/packs/` 复制为 `web/packs/`（**勿手动改 web 下副本**）。
>
> 代码定位：采集器 `dsh-pack-market/scripts/collect.mjs`（生成）；消费者 `dsh-pack-market/web/market.js`（网页）与 `dsh-packforge-app/packages/core/src/market.js`（`readMarketIndex` / `fetchMarketPackDetail`）。
>
> 关联：`../manifest/v3.md` / `../manifest/v4.md` / `../manifest/v5.md`（条目内 manifest 字段的契约，v3/v4/v5 兼容）、`../publishing/v1.md`（收录流程）。

---

## 1. 概述

`index.json` 是整合包平台的**中心索引**，采用「精简指针制」两段式：

- **索引（`index.json`）**：只平铺列表卡片 / 搜索 / 安装命令必需的字段——`downloadUrl + sha256 + size` 三个指针 + 展示元数据（`name` / `displayName` / `description` / `author` / `category` / `dshVersion` / `profileName` / `updatedAt`）+ 定位字段（`id` / `owner` / `repo`）+ 计数（`bundleCount` / `depCount` / `profileCount`）。**不含** `bundles[]` / `dependencies{}` / `files[]` / `profiles` / `presets` / `skills` 等可能较大的字段。
- **懒加载目录（`packs/<owner>.<repo>/`）**：每包一个目录，存**完整原始 manifest.json**（v3/v4/v5，原样）与源仓库 **README.md**。消费者需要完整字段时按 `id` 懒加载。

> 为什么拆开：v5 `dshhome` 形态的 manifest 可携带多 profile + presets + skills + instructions，体量可能较大；放进索引会让首页 index.json 膨胀。指针字段 + 计数足以支撑列表/搜索，重字段按需拉取。

---

## 2. 顶层结构

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `schemaVersion` | number | ✅ | 固定为 `2`（索引契约版本，**不是** manifest 版本） |
| `generatedAt` | string | 否 | ISO 8601 时间戳；采集器写入时刷新 |
| `modpacks` | object[] | ✅ | 整合包条目数组（采集器按 `updatedAt` 倒序） |

---

## 3. 条目字段

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `manifestVersion` | number | 否 | 声明本条所遵循的 manifest 契约版本（3/4/5），采集器缺省按 4 |
| `type` | string | 否 | `"profile"` \| `"dshhome"`；缺省按 `"profile"`（v5 dshhome 显式声明） |
| `name` | string | ✅ | slug（`^[a-z0-9-]+$`）；与 `version` 组成唯一键 |
| `version` | string | ✅ | semver |
| `displayName` | string \| object | ✅ | 显示名。字符串，或以语言代码为键的 map（manifest v3 起支持） |
| `description` | string \| object | 否 | 描述，形式同 `displayName` |
| `author` | string | 否 | 作者 |
| `category` | string | 否 | 分类（市场筛选用；缺省 `uncategorized`） |
| `dshVersion` | string | 否 | 运行所需 DSH 精确版本号（v3 起） |
| `profileName` | string | 否 | 安装时创建的 profile 名（profile 形态） |
| `downloadUrl` | string | ✅ | `http(s)` 下载地址（指针 1） |
| `sha256` | string | ✅ | 覆盖整个包的 64 位十六进制哈希（指针 2） |
| `size` | number | ✅ | 字节数，正整数（指针 3） |
| `updatedAt` | string | 否 | `YYYY-MM-DD` |
| `id` | string | ✅ | 懒加载目录 key = `<owner>.<repo>` |
| `owner` | string | ✅ | GitHub owner（`repo.full_name` 前半段） |
| `repo` | string | ✅ | GitHub 仓库名（`repo.full_name` 后半段） |
| `bundleCount` | number | 否 | 卡片用：profile 形态 = `bundles.length`；dshhome 形态 = 各 profile 合计 |
| `depCount` | number | 否 | 卡片用：依赖数（dshhome 形态为合计） |
| `profileCount` | number | 否 | 卡片用：仅 dshhome 形态，profile 数量 |

> 校验规则：`name` 必须小写 slug；`downloadUrl` 必须 `http(s)`；`sha256` 必须 64 位十六进制；`size` 必须正整数；`name@version` 全局唯一。`id` = `${owner}.${repo}`，owner 无点、repo 可含点，故从 `owner`+`repo` 字段重建、不做反向解析。

---

## 4. 完整示例（manifest v4 profile 条目）

```json
{
  "schemaVersion": 2,
  "generatedAt": "2026-09-02T15:44:00.000Z",
  "modpacks": [
    {
      "manifestVersion": 4,
      "type": "profile",
      "name": "all-about-whales",
      "version": "1.0.0",
      "displayName": { "en-US": "All About Whales", "zh-CN": "大肥鱼套装" },
      "description": {
        "en-US": "Make your DSH smell like big fat whales (beautify webUI with DeepSeek mascot theme)",
        "zh-CN": "让你的DSH充满大肥鱼的味道（用DeepSeek吉祥物主题美化webUI）"
      },
      "author": "hxh230802",
      "category": "coding",
      "dshVersion": "0.1.0-rc.8",
      "profileName": "all-about-whales",
      "downloadUrl": "https://github.com/DSH-PackForge/all-about-whales/releases/download/v1.0.0/all-about-whales-1.0.0.dspack",
      "sha256": "6dfaf5cc50f45c751e4bdff6a35eaedccd90cb67f6f888238a27a6d75409cf45",
      "size": 6670,
      "updatedAt": "2026-09-01",
      "id": "DSH-PackForge.all-about-whales",
      "owner": "DSH-PackForge",
      "repo": "all-about-whales",
      "bundleCount": 6,
      "depCount": 4
    }
  ]
}
```

懒加载源 `packs/DSH-PackForge.all-about-whales/manifest.json` 为该包完整原始 manifest（含 `bundles[]` / `dependencies{}` / `icon` / `patch` 等索引未平铺的字段），`README.md` 为源仓库 README 原文。

---

## 5. 与 manifest / 包结构的关系

| 层 | 载体 | 说明 |
| --- | --- | --- |
| 整合包 `.dspack` | 归档内 `manifest.json` | manifest 契约（`../manifest/`，v3/v4/v5） |
| 包结构 | 归档内文件布局 | 打包流程产出 `sha256` / `size`（`../pack-structure/`） |
| 索引 `index.json` | 中心仓库 | 精简指针：`downloadUrl` / `sha256` / `size` + 展示元数据 + `id` |
| 懒加载 `packs/<id>/` | 中心仓库 | 完整 manifest + README，详情页/客户端按需拉取 |

---

## 6. 已知边界 / 待同步

1. **`displayName` / `description` 多语言**：manifest v3 起为 `string | map`，索引条目跟随；校验器需按「string 或 map」处理。
2. `web/index.json`、`web/packs/` 是部署副本，勿手动改（事实源是 `index/`）。
3. 指针三件套的 `sha256` / `size` 由打包流程产出，完整性校验「牵一发动全身」。
4. v5 `dshhome` 形态的完整字段（`profiles` / `presets` / `skills` / `instructions`）**不在索引平铺**，仅存于懒加载 manifest；索引只带 `type` + 计数。
