# FreeDisplay

[English](README.md) · **简体中文**

> **针对 macOS 26 (Tahoe) 适配的 [FreeDisplay](https://github.com/huberdf/FreeDisplay) Fork** — 免费开源的 BetterDisplay 替代品,为 Apple 全新 DCP 显示架构重新打造。

BetterDisplay 是一款优秀的应用,但它最实用的功能都被锁在付费 Pro 版后面。FreeDisplay 把 BetterDisplay 最核心的能力实现为完全免费、开源的 macOS 菜单栏应用。

这个 Fork 把上游项目适配到了 **macOS 26 (Tahoe)**。Apple 在 Tahoe 上对显示子系统做了大量重构,引入了新的 DCP (Display Co-Processor) 驱动模型,上游若干代码路径在 Tahoe 上会无声地失效。本 Fork 包含全部修复。

[下载最新版本](https://github.com/Akuwatoga/FreeDisplay/releases/latest) · [反馈问题](https://github.com/Akuwatoga/FreeDisplay/issues) · [上游项目](https://github.com/huberdf/FreeDisplay)

---

## 为什么需要这个 Fork?

在 macOS 26 上直接运行上游 FreeDisplay 会遇到几个用户可见的故障:

| Tahoe 上的现象 | 根因 | 本 Fork 的修复 |
|---|---|---|
| 菜单栏面板塌缩成只剩底部版本号 — 显示器列表和工具区全不见 | SwiftUI `MenuBarExtra(.window)` + 内嵌 `ScrollView` 的 intrinsic 高度回归为 0 | 在面板容器上显式设置 `minHeight: 400` |
| 拖动亮度滑块时崩溃 (`EXC_BREAKPOINT`) | Swift 6 运行时隔离检查失败 — DDC 写完成回调在 main actor 之外被执行 | `DDCService.writeAsync` 的 completion 显式标注 `@MainActor @Sendable`,通过 `MainActor.assumeIsolated` 跨域执行 |
| 拖 A 屏的滑块却控制了 B 屏(或没反应) | Tahoe 移除了 `IODisplayConnect` 节点;旧的 `IORegistry` 父链 `vendor/product` 匹配始终失败,落到"按 sorted index 配对"瞎对,50/50 |  改用 EDID 直读匹配 — 对每个 `IOAVService` 读 I²C `0x50` 拿 EDID,解析 vendor/product,与 `CGDisplayVendorNumber/ModelNumber` 比对 |
| 部分显示器(如 LG ULTRAGEAR)即使 DDC 可用也被判定为软件路径 | 旧的活性探测用 DDC/CI `0x37/0x51` ping;很多显示器在收到真实 VCP 请求前保持沉默,被误杀 | 改用 EDID 读做探测 — 任何正常工作的显示器都会响应 `0x50` |
| 一次失败的 VCP 读之后亮度永远走软件路径 | 失败的 DDC *读* 被错误地等同于"显示器不支持 DDC",被标记为永远不可用 | 读失败不再剥夺写能力 |

每个 commit 的详细说明见 [`tahoe-compat`](https://github.com/Akuwatoga/FreeDisplay/tree/tahoe-compat) 分支的 git 历史。

---

## 替代了 BetterDisplay 的哪些功能?

| BetterDisplay 功能 | FreeDisplay | 说明 |
|---|:---:|---|
| DDC 亮度与对比度 | ✅ | 通过 IOKit I2C (Intel) / IOAVService (Apple Silicon) 硬件控制 — Tahoe 上基于 EDID 匹配 |
| 软件亮度(Gamma) | ✅ | 每屏独立的 gamma 表控制,带平滑过渡 |
| 外接显示器键盘亮度键 | ✅ | 当鼠标在外接屏上时拦截亮度键,显示 macOS 原生 OSD |
| 自动亮度同步 | ✅ | 跟随内建屏亮度变化按比例调整外接屏 |
| HiDPI 虚拟显示器 | ✅ | 通过 CGVirtualDisplay 私有 API 创建 HiDPI 虚拟显示器 |
| 显示器排列 | ✅ | 调整显示器位置(外接屏置于内建屏上方等) |
| 分辨率与 HiDPI 切换 | ✅ | 浏览并切换所有可用显示模式,包括 HiDPI |
| ICC 色彩描述文件管理 | ✅ | 通过 ColorSync 切换每屏色彩描述文件 |
| 图像调整(Gamma/色温) | ✅ | 软件对比度、色温、RGB 通道、反色 |
| 显示器预设 | ✅ | 一键保存与恢复完整的显示器配置 |
| 虚拟显示器(Dummy) | ✅ | 创建无头虚拟显示器 |
| 刘海管理 | ✅ | 用黑色遮罩隐藏 MacBook 刘海 |
| 开机自启 | ✅ | 通过 SMAppService |

### 主动不实现的功能

- 屏幕串流 / 画中画 — 很少用,会增加复杂度
- EDID 覆写 — 需要关闭 SIP
- XDR/HDR 额外亮度 — 需要特定硬件

### 已知的 Tahoe 系统级限制(非应用问题)

- **高带宽 DCP 通道拒绝 DDC 写指令**。部分显示器在 5K @ 144Hz(或类似 DSC 压缩模式)下,任何 `IOAVServiceWriteI2C` 调用都返回 `kIOReturn 0xe0114000`。应用会自动回退到软件 gamma。这是新 DCP 栈的系统级限制,影响所有用户态工具(MonitorControl、BetterDisplay 同样受影响)。可以把刷新率临时降到 60Hz 验证是否就是这个原因。

---

## 安装

### 方法一:下载 DMG

1. 从 [Releases](https://github.com/Akuwatoga/FreeDisplay/releases/latest) 下载 `FreeDisplay.dmg`
2. 打开 DMG,把 **FreeDisplay.app** 拖到 **Applications**
3. 首次启动:在 Finder 中右键 → **打开**(应用未签名,一次性确认),或直接去掉 quarantine 属性:
   ```bash
   sudo xattr -rd com.apple.quarantine /Applications/FreeDisplay.app
   ```

### 方法二:从源码构建

```bash
brew install xcodegen
git clone https://github.com/Akuwatoga/FreeDisplay.git
cd FreeDisplay
git checkout tahoe-compat
xcodegen generate
xcodebuild -scheme FreeDisplay -configuration Release CODE_SIGN_IDENTITY="-" CODE_SIGNING_REQUIRED=NO build
```

---

## 权限

| 权限 | 用途 |
|---|---|
| **辅助功能 (Accessibility)** | 拦截外接显示器的亮度键事件 |

不需要联网(除非启用了从 GitHub Releases API 的可选更新检查)。

---

## 技术栈

- **Swift 6** + **SwiftUI** (MenuBarExtra)
- **IOKit** — DDC/CI I2C 硬件亮度/对比度控制
- **CoreGraphics** — 显示器枚举、分辨率、排列
- **ColorSync** — ICC 色彩描述文件管理
- **CGVirtualDisplay** — 虚拟显示器创建(私有 API,macOS 14+)
- **CoreDisplay** — 内建屏亮度读取(私有 API,通过 dlopen)
- 零第三方依赖

---

## 项目结构

```
FreeDisplay/
├── App/              # AppDelegate、应用入口
├── Models/           # DisplayInfo、DisplayMode、DisplayPreset
├── Services/         # 系统层服务(DDC、亮度、分辨率、gamma 等)
└── Views/            # 各功能模块的 SwiftUI 视图
```

---

## 工作原理

FreeDisplay 常驻菜单栏,直接与你的显示器通信:

- **外接显示器**:通过 DDC/CI 协议走 I2C(Intel)或 IOAVService(Apple Silicon)硬件控制亮度、对比度等
- **内建显示器**:用 CoreGraphics gamma 表做软件亮度调节
- **亮度键**:安装 CGEventTap 拦截键盘亮度键,路由到鼠标光标所在的显示器
- **自动亮度**:通过 CoreDisplay 私有 API 轮询内建屏亮度,按比例调整外接屏
- **HiDPI**:通过 CGVirtualDisplay 私有 API 创建虚拟显示器,或写入 display override plist 实现持久 HiDPI

---

## 分支与发布

| 分支 | 用途 |
|---|---|
| `main` | 跟随上游 `huberdf/FreeDisplay` |
| `tahoe-compat` | macOS 26 (Tahoe) 适配的活跃开发分支 |

发布采用语义化版本号 + Tahoe 通道后缀(如 `v1.0.1-tahoe.1`)。

---

## 贡献

欢迎 issue 和 PR。本项目使用:
- `xcodegen` 生成工程文件(改 `project.yml`,别直接改 `.xcodeproj`)
- Swift 6 + `SWIFT_STRICT_CONCURRENCY: minimal`
- MVVM 架构(View → ViewModel → Service)
- Conventional Commits 规范(`fix(scope): ...`、`feat(scope): ...`、`chore: ...`)

---

## 许可证

MIT License — 详见 [LICENSE](LICENSE)。

---

## 致谢

- 上游项目:[huberdf/FreeDisplay](https://github.com/huberdf/FreeDisplay) — 本 Fork 基于此项目
- 灵感来源:[BetterDisplay](https://github.com/waydabber/BetterDisplay)、[MonitorControl](https://github.com/MonitorControl/MonitorControl)、[Lunar](https://lunar.fyi/)
- CGVirtualDisplay 桥接头基于 [Chromium 的 virtual_display_mac_util.mm](https://chromium.googlesource.com/chromium/src/+/main/ui/display/mac/test/virtual_display_mac_util.mm)
