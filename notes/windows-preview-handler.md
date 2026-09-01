# .dspack Windows 预览处理器 · 开发备忘

> 状态：**实现备忘（非 spec）**。对应格式契约见 `../specs/pack-structure/v2.md` 与 `../specs/manifest/v4.md`。
>
> 目标：为 `.dspack` 注册 Windows 资源管理器的**缩略图**（图标视图的小卡片）与**预览窗格**（右侧大卡片），选中文件即展示整合包信息，无需解包、无需打开平台。

---

## 1. 效果目标

| 位置 | 展示 |
| --- | --- |
| 缩略图（中等/大图标视图） | 迷你卡片：名称 + 版本 + 类型徽标 |
| 预览窗格（`Alt+P`） | 完整卡片：`displayName` / `version` / `type` / `description` / `author` / `dshVersion` / `bundles` 数 / `dependencies` 数 / `files` 数 |

---

## 2. 机制概览

Windows 按**扩展名**在注册表里查「shell 扩展」，找到对应 COM 组件后加载：

- **预览处理器（Preview Handler）**：实现 `IPreviewHandler`，加载到独立宿主进程 **`prevhost.exe`**（崩溃不影响 Explorer）。
- **缩略图（Thumbnail Provider）**：实现 `IThumbnailProvider`，**运行在 Explorer 进程内**，崩溃会拖垮 Explorer，所以更要稳健。

---

## 3. 技术栈选型

| 方案 | 结论 |
| --- | --- |
| **C++ / ATL** | ✅ **必选**。原生 COM DLL，零运行时冲突，微软官方样例即 ATL。 |
| **C# / .NET（COM 可见）** | ❌ **实测不可用**。`prevhost.exe`（低完整性原生宿主）拒绝承载托管 COM：完整 HKLM 注册 + 装到 `C:\Program Files` 后依然不加载（见 §3.1）。 |
| **Node.js / TypeScript** | ❌ 不合适。Node 无法当 COM 进程内服务；若强行做要写原生壳 DLL 去拉起 Node，毫秒级响应达不到。 |

> 结论：这个组件**只能走 C++**。C# 版（`dspack-preview` 仓库）已验证解析/COM/注册全链路，但因宿主限制作废。

### 3.1 实测证据（2026-09，C# 托管路线走不通）

| 进程 | 完整性 | 能否加载托管 COM | 说明 |
| --- | --- | --- | --- |
| `powershell.exe`（.NET） | 中 | ✅ | `Activator.CreateInstance` 成功，返回真实类型 |
| `cscript.exe`（原生） | 中 | ✅ | `CreateObject` 成功（原生进程能经 mscoree 加载） |
| `prevhost.exe`（原生宿主） | 低 | ❌ | 从未执行到构造函数（无日志），装到 `Program Files` + HKLM 完整注册依然不行 |

其余已验证：`AssocQueryString(ASSOCSTR_SHELLEXTENSION)` 能正确解析 `.dspack` 的预览处理器 CLSID；注册表两层键（`.dspack\shellex\{cat}` + `CLSID\…\InprocServer32`）完整。唯一失败点即 `prevhost` 加载阶段。

---

## 4. 接口清单

### 4.1 `IPreviewHandler`（必实现，7 个方法）

| 方法 | 作用 |
| --- | --- |
| `SetWindow(HWND, const RECT*)` | 传入父窗口句柄与初始矩形 |
| `SetRect(const RECT*)` | 预览区尺寸变化时被调用 |
| `DoPreview()` | 开始渲染（在这里读文件 + 建子窗口） |
| `Unload()` | 停止渲染、释放资源 |
| `SetFocus()` | 键盘焦点交给预览窗口 |
| `QueryFocus(HWND*)` | 返回当前焦点 HWND |
| `TranslateAccelerator(MSG*)` | 转发快捷键 |

### 4.2 初始化接口（三选一）

| 接口 | 说明 | 推荐 |
| --- | --- | --- |
| `IInitializeWithFile(PCWSTR, DWORD)` | 拿文件路径，最简单 | 本地文件可用 |
| `IInitializeWithStream(IStream*, DWORD)` | 拿流，云端/网络文件也能用 | ✅ 首选 |
| `IInitializeWithItem(IShellItem*, DWORD)` | 拿 Shell 项 | 特定场景 |

### 4.3 可选接口

- `IObjectWithSite`：接入 site 链；
- `IPreviewHandlerVisuals`：`SetTextColor` / `SetFont` / `SetBackgroundColor`，跟随系统深/浅色主题。

---

## 5. 注册表

> ⚠️ 实测：**每用户（`HKCU\Software\Classes`）注册对预览处理器无效**，必须机器级 `HKLM\Software\Classes`（需管理员）。CLSID 换成你自己生成的 GUID；`InprocServer32` 指向原生 C++ DLL。

```reg
; 预览 handler：固定类别 GUID {8895b1c6-b41f-4c1c-a562-0d564250836f}
[HKEY_LOCAL_MACHINE\Software\Classes\.dspack\shellex\{8895b1c6-b41f-4c1c-a562-0d564250836f}]
@="{YOUR-CLSID}"

; 缩略图 provider：固定类别 GUID {e357fccd-a995-4576-b01f-234630154e96}
[HKEY_LOCAL_MACHINE\Software\Classes\.dspack\shellex\{e357fccd-a995-4576-b01f-234630154e96}]
@="{YOUR-CLSID}"

; ★ 易漏：文件类型 ProgId（让 shell 识别 .dspack 为已注册类型）
[HKEY_LOCAL_MACHINE\Software\Classes\.dspack]
@="DspackPreview.PreviewHandler"

; ★ 易漏：ProgId 下也挂 shellex（部分 shell 从 ProgId 而非扩展名查找）
[HKEY_LOCAL_MACHINE\Software\Classes\DspackPreview.PreviewHandler\shellex\{8895b1c6-b41f-4c1c-a562-0d564250836f}]
@="{YOUR-CLSID}"

; COM 类注册（原生 DLL，装到低权限进程可读的 Program Files）
[HKEY_LOCAL_MACHINE\Software\Classes\CLSID\{YOUR-CLSID}]
@="DspackPreview.PreviewHandler"
; ★★★ 最关键：AppID 决定处理器被托管进 prevhost.exe 代理进程。
;   这是系统共享的 "Preview Handler Surrogate Host" AppID，其 DllSurrogate=%SystemRoot%\system32\prevhost.exe。
;   缺了它，AssocQueryString 能解析到 CLSID，但 COM 不把它注入 prevhost → 预览报「无法预览此文件」且 prevhost 里没有 DLL 装载记录。
"AppID"="{6d2b5079-2f0b-48dd-ab7f-97cec514d30b}"
"AutomaticallyPreviewUntrustedFiles"=dword:00000001

[HKEY_LOCAL_MACHINE\Software\Classes\CLSID\{YOUR-CLSID}\InprocServer32]
@="C:\\Program Files\\DSH-PackForge\\dspack-preview\\DspackPreview.dll"
"ThreadingModel"="Apartment"

; ★ 易漏：登记到系统预览处理器清单(缺它 prevhost 会 LoadLibrary 该 DLL 却不实例化、无 ctor)。
[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\PreviewHandlers]
"{YOUR-CLSID}"="DSH-PackForge .dspack Preview Handler"
```

> 全部键都由 `DllRegisterServer` 一次性写入（`regsvr32` 即生效）。实测「三层」缺一不可，按调用链理解：**`AppID`（进 prevhost 代理）→ `PreviewHandlers`（被实例化）→ `ProgId`/`.dspack\shellex`（被找到）**，缺任何一层都是「无法预览此文件」。

---

## 6. `.dspack` 读取流程

在 `DoPreview` / `GetThumbnail` 里：

1. 读流（`IInitializeWithStream`）或路径 → 直接按**标准 ZIP** 打开（`.dspack` 即纯 ZIP，文件头 `PK`，压缩软件可直开）。
2. 读根 `manifest.json`（v4），解析出卡片字段：

```jsonc
{
  "manifestVersion": 4,
  "type": "profile",
  "name": "all-about-whales",
  "version": "1.0.0",
  "displayName": { "zh-CN": "大肥鱼套装" },   // pickLang: en-US → zh-CN → 首个值
  "description": { "…": "…" },
  "author": "…",
  "dshVersion": "…",
  "bundles": ["…"],
  "dependencies": { "…": "…" },
  "files": [ { "path": "…", "sha256": "…", "size": 0, "urls": ["…"] } ]
}
```

3. 渲染：名称/版本/类型/描述/作者/运行版本/三层计数。

> 容器识别由根 `dspack.json`（`format===dspack`、`version===2`）承载，见 `pack-structure/v2.md` §2.2；预览处理器只需 `manifest.json` 即可出卡。早期带 8 字节 `DSPK` 前导头的旧文件已废弃——处理器保留剥头向后兼容，但新文件一律为纯 ZIP。

---

## 7. 渲染方案

| 方案 | 说明 | 评价 |
| --- | --- | --- |
| 裸 HWND + GDI / Direct2D | 自绘卡片 | 轻量但排版代码量大 |
| **内嵌 WebView2 + 一段 HTML** | 由 manifest 拼 HTML 卡片交给 WebView2 渲染 | ✅ 推荐：图文排版好看、主题易适配 |
| WPF / WinForms（C#） | `HwndSource` 挂载 | 仅 C# 路线用 |

---

## 8. 开发路线

1. 新建 C++/ATL 项目（或 C# Class Library + `[ComVisible]`）；
2. **先做缩略图** `IThumbnailProvider`（读 manifest 画一张小卡片位图）→ 注册 → 图标视图见效；
3. **再做预览** `IPreviewHandler`（`DoPreview` 建子窗口 + 渲染）→ 注册 → 预览窗格见效；
4. `regsvr32` 或 .reg 注册；反复调试时 `taskkill /f /im prevhost.exe` 清宿主缓存。

---

## 9. 调试与注意

- **必须快**：预览窗格有约 2~3 秒超时，超时显示「没有可用预览」。按需解包，别把整个 zip 解完。
- **别长期占文件句柄**：读完即放，避免用户删文件时被占用。
- **缩略图线程**：`IThumbnailProvider` 在 Explorer 进程，务必异常安全。
- **防栈溢出依赖**：C++ 里用 `try/catch` 兜住 zip 解析错误。

---

## 10. 边界：纯 ZIP 容器（已定案）

`.dspack` **就是标准 ZIP + 根 `dspack.json` 标记**（无前导魔数字节），与 `pack-structure/v2.md` §2 一致：

- **压缩软件可直接打开**：文件头即 `PK`，WinRAR / 7-Zip / Explorer 自带 zipfolder 都能直接浏览；
- **识别靠根标记文件** `dspack.json`（`format` / `version`），而非文件头字节；
- 早期「前导 8 字节 magic」方案因「压缩软件打不开」已废弃；处理器保留剥头逻辑仅向后兼容旧文件，新文件一律纯 ZIP。

---

## 11. 参考资料

- learn.microsoft.com → 搜 **Preview Handlers** / `IPreviewHandler` / `IThumbnailProvider` / `IInitializeWithStream`
- GitHub → `microsoft/Windows-classic-samples`，内含官方 **RecipePreviewHandler** 完整样例