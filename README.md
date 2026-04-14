# android-reverse-engineering-skill

这个仓库同时包含一套 `Claude Code` 插件和一套 `Codex` 插件封装，它们共用同一个 Android 逆向 skill。

这是一个用于 Claude Code 的技能，能够反编译 Android APK/XAPK/JAR/AAR 文件，**提取应用使用的 HTTP API**，并为更深入的逆向分析提供方法指引，例如 Frida 前期侦察、运行时请求检查、JNI/SO 分析，以及签名定位分析。

## 项目来源

本项目基于原项目 [SimoneAvogadro/android-reverse-engineering-skill](https://github.com/SimoneAvogadro/android-reverse-engineering-skill) 演化而来。

原项目主要聚焦于：

- Android APK/XAPK/JAR/AAR 的反编译
- API 提取与调用链分析
- Claude Code skill 形态下的基础工作流

本仓库在此基础上做了面向当前使用场景的扩展和调整。

## 与原项目的差异

相比原项目，这个仓库目前的主要不同点是：

- 增加了 `Codex` 插件封装与相关元数据，不再只面向 Claude Code
- README 已改为中文，更适合中文使用者直接阅读和上手
- skill 描述范围已经扩展到 Frida 前期侦察、运行时分析、JNI/SO 分析、签名定位分析
- 新增了 `dynamic-analysis.md` 与 `native-analysis.md` 两份参考文档，用于指导动态分析与 Native 分析流程
- 文档中明确区分了“支持 Frida 分析方法”与“尚未内置 Frida 执行脚本”这两件事
- 保留原有反编译与 API 提取能力的同时，更强调“先静态侦察，再决定是否进入动态分析”

## 功能概览

- 使用 jadx 和 Fernflower/Vineflower 反编译 APK、XAPK、JAR、AAR，支持单引擎或双引擎对比
- 提取并整理 API：Retrofit 接口、OkHttp 调用、硬编码 URL、鉴权头和 token
- 从 Activity/Fragment 沿着 ViewModel、Repository 一路跟踪到 HTTP 调用
- 分析应用结构：Manifest、包结构、架构模式
- 处理混淆代码：提供在 ProGuard/R8 输出中定位逻辑的策略，并区分“运行时真实类名”与“反编译器可读别名”
- 提供运行时分析指引：什么时候该用 Frida、该 hook 哪一层、如何收窄请求/签名链路
- 提供 Native 分析指引：JNI/SO 侦察、签名生成边界识别，以及什么时候值得上 unidbg

## Frida 支持范围

这个 skill 现在已经包含面向 Frida 的分析方法文档，但**还没有**内置可直接执行的 Frida 辅助脚本，也没有自动 attach 的命令。

当前状态：

- 可以指导你在 hook 之前先完成静态侦察
- 可以帮助你判断哪个 Java 或 JNI 边界最适合 hook
- 可以组织运行时分析结论和 Native 签名分析结论
- 仓库里**还没有**用于 `frida`、设备连接、自动生成 hook 模板的 `scripts/`

一句话概括：当前项目支持的是 **Frida 分析流程指导**，不是 **内置 Frida 执行工具链**。

## 依赖要求

**必需：**

- Java JDK 17+
- [jadx](https://github.com/skylot/jadx) 命令行工具

**可选但推荐：**

- [Vineflower](https://github.com/Vineflower/vineflower) 或 [Fernflower](https://github.com/JetBrains/fernflower)，用于在复杂 Java 代码上获得更好的反编译结果
- [dex2jar](https://github.com/pxb1988/dex2jar)，用于在 APK/DEX 场景下配合 Fernflower 工作

详细安装说明见 `plugins/android-reverse-engineering/skills/android-reverse-engineering/references/setup-guide.md`。

## 安装

### 用于 Codex

这个仓库已经包含 Codex 兼容的插件元数据：

- `.agents/plugins/marketplace.json`
- `plugins/android-reverse-engineering/.codex-plugin/plugin.json`

skill 的主体内容位于：

- `plugins/android-reverse-engineering/skills/android-reverse-engineering/`

### 从 GitHub 安装（推荐）

在 Claude Code 中运行：

```
/plugin marketplace add SimoneAvogadro/android-reverse-engineering-skill
/plugin install android-reverse-engineering@android-reverse-engineering-skill
```

安装后，这个 skill 会在之后的会话中持续可用。

### 从本地仓库安装

```bash
git clone https://github.com/SimoneAvogadro/android-reverse-engineering-skill.git
```

然后在 Claude Code 中运行：

```
/plugin marketplace add /path/to/android-reverse-engineering-skill
/plugin install android-reverse-engineering@android-reverse-engineering-skill
```

## 使用方式

### Slash 命令

```
/decompile path/to/app.apk
```

这会执行完整流程：检查依赖、执行反编译，并做初步结构分析。

### 自然语言触发

以下表达都可能触发这个 skill：

- “反编译这个 APK”
- “逆向分析这个 Android 应用”
- “提取这个应用的 API 接口”
- “从 LoginActivity 开始跟调用链”
- “分析这个 AAR 库”
- “找出这个应用的 sign 是在哪生成的”
- “用 Frida 分析请求链路”
- “分析这个 token 背后的 JNI 或 SO”

### 手动执行脚本

这些脚本也可以单独运行：

```bash
# 检查依赖
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/check-deps.sh

# 安装缺失依赖（自动识别操作系统和包管理器）
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/install-dep.sh jadx
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/install-dep.sh vineflower

# 使用 jadx 反编译 APK（默认）
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/decompile.sh app.apk

# 需要保留运行时真实类名时，第一次反编译建议不要加 --deobf
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/decompile.sh app.apk

# 只在更重视源码可读性时再启用 --deobf
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/decompile.sh --deobf app.apk

# 反编译 XAPK（自动解包并逐个处理其中的 APK）
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/decompile.sh app-bundle.xapk

# 使用 Fernflower 反编译
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/decompile.sh --engine fernflower library.jar

# 同时运行两个引擎并对比结果
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/decompile.sh --engine both --deobf app.apk

# 查找 API 调用
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/find-api-calls.sh output/sources/
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/find-api-calls.sh output/sources/ --retrofit
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/find-api-calls.sh output/sources/ --urls
```

## 反编译命名策略

默认建议先使用 **不带 `--deobf`** 的 `jadx` 做第一轮反编译，原因是：

- 更接近 dex 里的运行时类名
- 更适合 Frida、JNI `FindClass`、unidbg、运行时 `Class.forName`
- 避免把只存在于反编译结果中的可读别名误当成运行时类名

`--deobf` 仍然有价值，但更适合这些场景：

- 代码高度混淆，只想先提升阅读体验
- 当前任务以静态阅读为主，不需要马上做运行时 hook
- 想用第二份输出和原始命名版本对照分析

## 仓库结构

```
android-reverse-engineering-skill/
├── .agents/
│   └── plugins/
│       └── marketplace.json                # Codex 插件市场目录
├── .claude-plugin/
│   └── marketplace.json                    # Claude 插件市场目录
├── plugins/
│   └── android-reverse-engineering/
│       ├── .codex-plugin/
│       │   └── plugin.json                 # Codex 插件清单
│       ├── .claude-plugin/
│       │   └── plugin.json                 # Claude 插件清单
│       ├── skills/
│       │   └── android-reverse-engineering/
│       │       ├── SKILL.md                # 核心流程（静态分析 + 动态侦察）
│       │       ├── references/
│       │       │   ├── setup-guide.md
│       │       │   ├── jadx-usage.md
│       │       │   ├── fernflower-usage.md
│       │       │   ├── api-extraction-patterns.md
│       │       │   ├── call-flow-analysis.md
│       │       │   ├── dynamic-analysis.md
│       │       │   └── native-analysis.md
│       │       └── scripts/
│       │           ├── check-deps.sh
│       │           ├── install-dep.sh
│       │           ├── decompile.sh
│       │           └── find-api-calls.sh
│       └── commands/
│           └── decompile.md                # /decompile 命令说明
├── LICENSE
└── README.md
```

## 参考项目

- [jadx — Dex to Java 反编译器](https://github.com/skylot/jadx)
- [Fernflower — JetBrains 反编译器](https://github.com/JetBrains/fernflower)
- [Vineflower — Fernflower 社区分支](https://github.com/Vineflower/vineflower)
- [dex2jar — DEX 转 JAR 工具](https://github.com/pxb1988/dex2jar)
- [apktool — Android 资源解码工具](https://apktool.org/)
- [Frida — 动态插桩工具](https://frida.re/)

## 免责声明

这个插件仅可用于**合法用途**，包括但不限于：

- 安全研究与经授权的渗透测试
- 适用法律允许范围内的互操作性分析
- 恶意软件分析与应急响应
- 教学用途与 CTF 比赛

**你需要自行确保** 对本工具的使用符合所在司法辖区的法律、法规以及目标软件的服务条款。对无授权的软件进行逆向，可能触犯知识产权或计算机相关法律。

作者不对任何滥用行为承担责任。

## 许可证

Apache 2.0，见 [LICENSE](LICENSE)
