# v6 格式示例

`profile/` 与 `dshhome/` 分别对应一个完整的 v4 容器归档根目录。将任一目录内的全部文件直接打包为标准 ZIP 并命名为 `.dspack`，即可在支持 v6 的读取端中预览。以 SHA-256 命名的内容文件按原始字节校验，手动修改时须同步更新对应摘要。示例中的 `registry.example`、`api.example` 与 `mcp.example` 仅用于演示格式，实际联网安装需由作者提供真实可用的服务地址与发布坐标。

| 示例 | 涵盖内容 |
| --- | --- |
| [单配置档（profile）](profile/manifest.json) | 插件多语言名称、内嵌归档图标、精确 npm 包源（registry）、插件数据路径及模型服务商（Provider）凭据引用 |
| [全局目录（dshhome）](dshhome/manifest.json) | 两个配置档、一个 MCP 服务实例及内嵌技能（Skill）目录 |
| [插件路径声明](plugin/dsh_pack/paths.json) | 演示如何从插件发布目录的 `dsh_pack/paths.json` 生成单配置档示例中的 `path.json` |

两个示例归档均不包含真实凭据；导入预览时可跳过需要凭据的项目。[清单规范 v6](../../specs/manifest/v6.md) · [容器规范 v4](../../specs/pack-structure/v4.md) · [插件数据与可选资源规范 v1](../../specs/dsh_pack/v1.md)
