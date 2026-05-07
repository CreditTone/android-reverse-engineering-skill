# android-reverse-engineering-skill

Android 逆向分析全家桶 — 从 APK 反编译到 Frida 动态插桩、从 HTTP 接口提取到 JNI/SO 原生层逆向，覆盖静态分析→动态分析→流量解密→签名追踪的完整链路。支持 Claude Code / Codex 双平台，兼容 macOS、Linux、Windows（PowerShell），一套技能打通 Android 安全研究与授权渗透测试的绝大部分需求。

## 亮点

- **双引擎反编译**：jadx + Fernflower/Vineflower，自动对比、择优输出
- **完整调用链追踪**：从 Activity/Fragment → ViewModel → Repository → Retrofit/OkHttp，一路不迷路
- **API 全量提取**：Retrofit 接口、OkHttp 请求、硬编码 URL、鉴权头、Token、签名参数一键搜出
- **Frida 脚本武装**：9 个实战级 Frida 脚本开箱即用 — SSL 日志、DEX/SO dump、JNI 追踪、加密 hook、Root/Frida 检测绕过、KeyStore 证书导出
- **Native 层分析**：JNI 注册追踪、SO 符号/导出分析、签名/加密边界定位、unidbg 模拟前的完整静态侦察
- **XAPK / 套壳 APK 自动解包**：自动识别分包结构，直取业务核心 base.apk
- **跨平台**：Bash 脚本 + PowerShell 脚本双栈，macOS/Linux/Windows 通吃
- **智能依赖安装**：一键检查 + 自动安装缺失工具，无 sudo 时自动降级为手动指引

## 与原项目的差异

基于 [SimoneAvogadro/android-reverse-engineering-skill](https://github.com/SimoneAvogadro/android-reverse-engineering-skill) 做了大幅扩展：

| 方面 | 原项目 | 本仓库 |
|---|---|---|
| 平台支持 | Claude Code | Claude Code + Codex 双平台 |
| 反编译引擎 | jadx | jadx + Fernflower/Vineflower 双引擎对比 |
| 动态分析 | 概念提及 | 完整 Frida 实战指导 + 9 个开箱即用脚本 |
| Native 分析 | 无 | JNI/SO 完整分析流程 + rizin 命令参考 |
| Windows 支持 | 无 | 完整 PowerShell 脚本栈 |
| XAPK/套壳 | 基础支持 | 自动递归解包 + 分包识别 |
| 依赖管理 | 手动 | 一键检查 + 跨平台自动安装 |
| Frida 脚本 | 无 | 9 个实战脚本（SSL、DEX dump、SO dump、JNI trace、加密 hook、检测绕过、证书导出） |
| 文档语言 | 英文 | 中文 |

## 功能总览

### 静态分析
- jadx / Fernflower / Vineflower 三种引擎反编译 APK、XAPK、JAR、AAR
- 自动解析 AndroidManifest.xml、包结构、架构模式（MVP/MVVM/Clean Architecture）
- 混淆代码处理：ProGuard/R8 混淆下的字符串锚点定位策略

### API 与网络层
- 自动提取 Retrofit 接口、OkHttp 调用、硬编码 URL、鉴权头、Token
- 请求/响应结构还原，Base URL + Path + Headers + Body 完整文档化
- 签名参数与加密入口定位

### 调用链追踪
- Activity → ViewModel/Presenter → Repository → API Service → HTTP Call
- Dagger/Hilt 依赖注入绑定映射
- 混淆类名下的 Retrofit 注解锚点导航

### Frida 动态插桩（9 个脚本开箱即用）

| 脚本 | 功能 |
|---|---|
| `ssl_log.js` | TLS 密钥日志导出，解密 HTTPS 流量 |
| `dump_dex.js` | 内存 DEX dump，对抗加固/动态加载 |
| `dump_so.js` | 内存 SO dump，保留完整 ELF 结构 |
| `jni_method_trace.js` | JNI 调用全量追踪，可指定目标 SO |
| `hook_artmethod_register.js` | ART Method 注册 hook，追踪 Native→Java 绑定 |
| `hook_encryption_algo.js` | AES/DES/RSA/哈希等加密算法调用拦截 |
| `keystore_dump.js` | 双向认证场景下客户端证书导出为 p12 |
| `bypass_root_detect.js` | 通用 Root 检测绕过（含 Magisk 检测） |
| `bypass_frida_svc_detect.js` | Frida SVC 指令级检测绕过（ARM32/ARM64） |

### Native / JNI / SO 分析
- `native` 方法声明 → `System.loadLibrary` → SO 定位
- `JNI_OnLoad` → `RegisterNatives` 动态注册追踪
- rizin/readelf/nm/objdump 多工具栈 SO 分析
- 签名/加密生成边界判断：Java 层还是 Native 层？

## 依赖要求

**必需：**
- Java JDK 17+
- [jadx](https://github.com/skylot/jadx) 命令行工具

**可选但推荐：**
- [Vineflower](https://github.com/Vineflower/vineflower) / [Fernflower](https://github.com/JetBrains/fernflower) — 复杂 Java 代码更优输出
- [dex2jar](https://github.com/pxb1988/dex2jar) — APK/DEX 配合 Fernflower 使用
- [Rizin](https://rizin.re/) — SO / JNI / Native 导出符号与反汇编
- [Frida](https://frida.re/) — 运行时动态插桩
- [adb](https://developer.android.com/tools/adb) — 设备连接与脚本推送

## 安装

```bash
# 克隆仓库
git clone https://github.com/SimoneAvogadro/android-reverse-engineering-skill.git

# 创建技能符号链接
mkdir -p ~/.claude/skills
ln -sfn $(pwd)/android-reverse-engineering-skill/android-reverse-engineering/skills/android-reverse-engineering ~/.claude/skills/android-reverse-engineering

# 创建命令符号链接
mkdir -p ~/.claude/commands
ln -sfn $(pwd)/android-reverse-engineering-skill/android-reverse-engineering/commands/decompile.md ~/.claude/commands/decompile.md
```

## 使用方式

### Slash 命令

```
/decompile path/to/app.apk
```

自动完成：依赖检查 → 反编译 → 结构分析 → 下一步建议。

### 自然语言触发

- "反编译这个 APK"
- "逆向分析这个应用"
- "提取这个应用的接口和签名"
- "从 LoginActivity 开始跟调用链"
- "分析这个 AAR/JAR 库"
- "找出这个 sign 是在 Java 还是 native 层生成的"
- "用 Frida 分析请求加密链路"
- "dump 这个加固应用的 DEX"
- "绕过这个 app 的 SSL pinning 和 root 检测"
- "分析这个 SO 的 JNI 导出函数"

### 手动执行脚本

```bash
SKILL_DIR="android-reverse-engineering/skills/android-reverse-engineering/scripts"

# === 依赖管理 ===
bash $SKILL_DIR/check-deps.sh
bash $SKILL_DIR/install-dep.sh jadx
bash $SKILL_DIR/install-dep.sh vineflower
bash $SKILL_DIR/install-dep.sh rizin

# === 反编译 ===
bash $SKILL_DIR/decompile.sh app.apk                         # jadx 默认
bash $SKILL_DIR/decompile.sh --engine fernflower library.jar  # Fernflower
bash $SKILL_DIR/decompile.sh --engine both app.apk            # 双引擎对比
bash $SKILL_DIR/decompile.sh app-bundle.xapk                  # XAPK 自动解包

# === API 提取 ===
bash $SKILL_DIR/find-api-calls.sh output/sources/
bash $SKILL_DIR/find-api-calls.sh output/sources/ --retrofit
bash $SKILL_DIR/find-api-calls.sh output/sources/ --urls
bash $SKILL_DIR/find-api-calls.sh output/sources/ --auth
```

### Windows / PowerShell

```powershell
$SKILL_DIR="android-reverse-engineering/skills/android-reverse-engineering/scripts"

& "$SKILL_DIR/check-deps.ps1"
& "$SKILL_DIR/install-dep.ps1" jadx
& "$SKILL_DIR/decompile.ps1" app.apk
& "$SKILL_DIR/find-api-calls.ps1" output/sources/ -Retrofit
```

### Frida 脚本使用

```bash
# SSL 密钥日志（配合 Wireshark 解密 HTTPS）
frida -U -f com.example.app -l ssl_log.js --no-pause

# Dump 内存中的 DEX（对抗加固/动态加载）
frida -U -f com.example.app -l dump_dex.js --no-pause

# Dump 内存中的 SO（保留 ELF 完整结构）
frida -U -f com.example.app -l dump_so.js --no-pause

# 追踪指定 SO 的 JNI 调用
frida -U -f com.example.app -l jni_method_trace.js --no-pause

# Hook 加密/解密算法调用
frida -U -f com.example.app -l hook_encryption_algo.js --no-pause

# 绕过 Root 检测
frida -U -f com.example.app -l bypass_root_detect.js --no-pause

# 绕过 Frida SVC 指令级检测
frida -U -f com.example.app -l bypass_frida_svc_detect.js --no-pause

# 导出双向认证客户端证书
frida -U -f com.example.app -l keystore_dump.js --no-pause

# Hook ART Method 注册（追踪 Native→Java 绑定）
frida -U -f com.example.app -l hook_artmethod_register.js --no-pause
```

## SO / Native 分析

当 Java 层分析看到 `native` 方法、`System.loadLibrary()` 或 `RegisterNatives` 时：

```bash
# ELF 元数据
rz-bin -I libfoo.so && rz-bin -s libfoo.so && rz-bin -i libfoo.so

# 字符串速查
rz-strings -a libfoo.so | rg 'http|https|Java_|JNI_OnLoad|encrypt|sign|ssl|socket'

# JNI_OnLoad 反汇编
rizin -qc "aaa; pdf @ sym.JNI_OnLoad; q" libfoo.so

# 无 rizin 时回退
readelf -Ws libfoo.so && nm -D libfoo.so | rg 'JNI_OnLoad|Java_'
objdump -d libfoo.so > libfoo.asm
```

## 反编译命名策略

首次反编译**不要加 `--deobf`**：
- 类名更接近 dex 运行时真实名称
- 适合后续 `FindClass`、`Class.forName`、Frida hook、unidbg 模拟

`--deobf` 适用于：
- 代码严重混淆，优先提升阅读体验
- 纯静态分析、不需要对接运行时类名
- 作为第二份输出与原始版本对照

## 仓库结构

```
android-reverse-engineering-skill/
├── .claude-plugin/
│   └── marketplace.json
├── android-reverse-engineering/
│   ├── .codex-plugin/
│   │   └── plugin.json
│   ├── .claude-plugin/
│   │   └── plugin.json
│   ├── commands/
│   │   └── decompile.md
│   └── skills/
│       └── android-reverse-engineering/
│           ├── SKILL.md
│           ├── references/
│           │   ├── setup-guide.md
│           │   ├── jadx-usage.md
│           │   ├── fernflower-usage.md
│           │   ├── api-extraction-patterns.md
│           │   ├── call-flow-analysis.md
│           │   ├── dynamic-analysis.md
│           │   └── native-analysis.md
│           └── scripts/
│               ├── check-deps.sh / .ps1          # 依赖检查
│               ├── install-dep.sh / .ps1         # 自动安装
│               ├── decompile.sh / .ps1           # 反编译
│               ├── find-api-calls.sh / .ps1      # API 提取
│               ├── ssl_log.js                    # TLS 密钥导出
│               ├── dump_dex.js                   # 内存 DEX dump
│               ├── dump_so.js                    # 内存 SO dump
│               ├── jni_method_trace.js           # JNI 调用追踪
│               ├── hook_artmethod_register.js    # ART 注册 hook
│               ├── hook_encryption_algo.js       # 加密算法 hook
│               ├── keystore_dump.js              # 证书导出
│               ├── bypass_root_detect.js         # Root 检测绕过
│               └── bypass_frida_svc_detect.js    # Frida 检测绕过
├── LICENSE
└── README.md
```

## 参考项目

- [jadx](https://github.com/skylot/jadx) — DEX to Java 反编译器
- [Fernflower](https://github.com/JetBrains/fernflower) — JetBrains 反编译器
- [Vineflower](https://github.com/Vineflower/vineflower) — Fernflower 社区分支
- [dex2jar](https://github.com/ThexXTURBOXx/dex2jar) — DEX 转 JAR
- [apktool](https://apktool.org/) — Android 资源解码
- [Rizin](https://rizin.re/) — 开源二进制逆向工具链
- [Frida](https://frida.re/) — 动态插桩框架
- [unidbg](https://github.com/zhkl0228/unidbg) — 无设备 SO 模拟执行

## 免责声明

本工具仅可用于**合法用途**：
- 安全研究与经授权的渗透测试
- 适用法律范围内的互操作性分析
- 恶意软件分析与应急响应
- 教学用途与 CTF 比赛

**你需自行确保**使用行为符合所在司法辖区法律、法规及目标软件服务条款。对未授权软件进行逆向可能触犯知识产权或计算机相关法律。作者不对任何滥用行为承担责任。

## 许可证

Apache 2.0，见 [LICENSE](LICENSE)
