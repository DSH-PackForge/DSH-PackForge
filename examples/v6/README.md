# v6 格式示例

`profile/` 和 `dshhome/` 各自是一个完整的 v4 归档根。将目录内文件直接加入标准 ZIP，并将后缀设为 `.dspack`，即可在读取端预览。文件名为 SHA-256 的内容按原始字节校验，编辑时须同步更新摘要。域名 `registry.example`、`api.example` 和 `mcp.example` 用于说明格式，实际联网安装需要作者提供可用的服务与真实发布坐标。

| 示例 | 可检查的内容 |
| --- | --- |
| [单 profile](profile/manifest.json) | 插件多语言名称、归档图标、精确 registry、插件布局文件和模型服务商凭据引用 |
| [整个 DSH_HOME](dshhome/manifest.json) | 两个 profile、一个 MCP 实例和内嵌技能目录 |
| [插件作者的路径声明](plugin/dsh_pack/paths.json) | 从插件发布目录的声明生成单 profile 示例的 `path.json` |

两个归档均不含凭据；导入预览可跳过依赖凭据的项目。[清单规范](../../specs/manifest/v6.md) · [容器规范](../../specs/pack-structure/v4.md) · [四种附件格式](../../specs/dsh_pack/v1.md)
