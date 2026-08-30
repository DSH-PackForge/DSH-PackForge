# 索引契约（index.json，schemaVersion 1）

> 状态：**现行**。中心索引 = 「指针制」：内容（`.tgz`）放分布式存储，索引只记 `downloadUrl + sha256 + size` 三个指针 + 用于市场展示的元数据平铺。
>
> 事实源：`dsh-pack-market/index/index.json`；部署时由 CI 复制为 `web/index.json` 供市场页读取（**勿手动改后者**）。
>
> 代码定位：`ModPack-CLI/packages/modpack/src/indexfile.js`（`buildEntry` 生成、`validateIndexFile` / `validateEntry` 校验）。
>
> 关联：`../manifest/v3.md`（条目内 manifest 字段的契约）、`../pack-structure/v1.md`（§6 产出 sha256/size，§8 指针制闭环）。

---

## 1. 概述

`index.json` 是整合包平台的**中心索引**：

- **指针制**：条目不存放内容本体，只放 `downloadUrl + sha256 + size` 三个指针；内容托管在作者自己的 GitHub Releases / OSS / npm tarball 等任意源。
- **元数据平铺**：条目把 manifest 的核心字段平铺出来（`name` / `displayName` / `bundles` / `dependencies`…），供市场页直接展示、搜索、筛选，而无需解包。
- **唯一事实源**：`dsh-pack-market/index/index.json`；`web/index.json` 是部署副本（CI 实时复制）。

---

## 2. 顶层结构

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `schemaVersion` | number | ✅ | 固定为 `1`（索引契约版本，**不是** manifest 版本） |
| `generatedAt` | string | 否 | ISO 8601 时间戳；`addEntry` 写入时刷新 |
| `modpacks` | object[] | ✅ | 整合包条目数组（按 `name`、`version` 排序） |

---

## 3. 条目字段

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `manifestVersion` | number | 否 | 声明本条所遵循的 manifest 契约版本（v3 起建议填 `3`，见 `../manifest/v3.md`） |
| `name` | string | ✅ | slug（`^[a-z0-9-]+$`）；与 `version` 组成唯一键 |
| `version` | string | ✅ | semver |
| `displayName` | string \| object | ✅ | 显示名。字符串，或以语言代码为键的 map（manifest v3 起支持，同 v3） |
| `description` | string \| object | 否 | 描述，形式同 `displayName` |
| `author` | string | 否 | 作者 |
| `icon` | string | 否 | 图标（URL 或归档内相对路径） |
| `dshVersion` | string | ✅ | 运行所需 DSH 版本：v2 为 semver 范围，v3 为精确版本号 |
| `profileName` | string | 否 | 安装时创建的 profile 名 |
| `bundles` | string[] | ✅ | profile 层栈（来自 `dsh.profile.bundles`） |
| `dependencies` | object | 否 | 依赖坐标：v2 为 pnpm spec，v3 为「坐标 → 固定版本 / commit sha」 |
| `category` | string | 否 | 分类（市场筛选用） |
| `downloadUrl` | string | ✅ | `http(s)` 下载地址（指针 1） |
| `sha256` | string | ✅ | 覆盖整个 `.tgz` 的 64 位十六进制哈希（指针 2） |
| `size` | number | ✅ | 字节数，正整数（指针 3） |
| `updatedAt` | string | 否 | `YYYY-MM-DD` |

> 校验规则（`validateEntry`）：`name` 必须是小写 slug；`downloadUrl` 必须 `http(s)`；`sha256` 必须 64 位十六进制；`size` 必须正整数；`bundles` 必须是字符串数组；`dependencies` 若存在必须是对象；`name@version` 全局唯一。

---

## 4. 完整示例（manifest v3 条目）

```json
{
  "schemaVersion": 1,
  "generatedAt": "2026-08-15T09:36:47.625Z",
  "modpacks": [
    {
      "manifestVersion": 3,
      "name": "all-about-whales",
      "displayName": { "en-US": "All About Whales", "zh-CN": "大肥鱼套装" },
      "version": "1.0.0",
      "description": {
        "en-US": "Make your DSH smell like big fat whales (beautify webUI with DeepSeek mascot theme)",
        "zh-CN": "让你的DSH充满大肥鱼的味道（用DeepSeek吉祥物主题美化webUI）"
      },
      "author": "hxh230802",
      "icon": "",
      "dshVersion": "0.1.1-rc.2",
      "profileName": "all-about-whales",
      "bundles": ["@deepseek-ai/dsh-base", "@deepseek-ai/dsh-web-app", "dafy-whale-theme", "dsh-whale-widget", "dsh-reasoning-effort", "dsh-pet"],
      "dependencies": {
        "github:DViridescent/dafy-whale-theme": "99e8c57",
        "dsh-pet": "0.2.0",
        "github:HanaAyane/dsh-reasoning-effort": "83bc8c5",
        "github:MeteorNOX/DeepSeek-Balance-Whale-Widget": "4448c61"
      },
      "category": "coding",
      "downloadUrl": "https://raw.githubusercontent.com/hxh230802/dsh-modpacks/main/all-about-whales-1.0.0.tgz",
      "sha256": "c53f18814e8912dc045e9da61ccef0afa92d54f57df9d2ddf08db19476e9b2c2",
      "size": 10916,
      "updatedAt": "2026-08-27"
    }
  ]
}
```

---

## 5. 与 manifest / 包结构的关系

| 层 | 载体 | 说明 |
| --- | --- | --- |
| 整合包 `.tgz` | 归档内 `manifest.json` | manifest 契约（`../manifest/`） |
| 包结构 | 归档内文件布局 | 打包流程产出 `sha256` / `size`（`../pack-structure/v1.md` §6） |
| 索引 `index.json` | 中心仓库 | 从 manifest 平铺元数据 + `downloadUrl` / `sha256` / `size` 指针 |

---

## 6. 已知边界 / 待同步

1. **`displayName` / `description` 多语言**：manifest v3 起为 `string | map`，索引条目跟随；`ModPack-CLI` 的 `index validate`（`indexfile.js#validateEntry`）当前仍按 `typeof === 'string'` 校验，需同步为「string 或 map」。
2. `web/index.json` 是部署副本，勿手动改（事实源是 `index/index.json`）。
3. 指针三件套的 `sha256` / `size` 由打包流程产出（`pack-structure/v1.md` §6），完整性校验「牵一发动全身」。