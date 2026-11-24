# PdfCraft 0.22.0 使用说明：从压缩包到跑起来

这份说明对应 `PdfCraft-完整代码-0.22.0.zip`：PdfCraft 0.22.0 源码 + 本地 AI（Ollama / gemma3:4b）+ 简体中文界面 + 右键翻译 + 可换界面字体。包里**没有现成的 exe**，第一次要自己编译；想要直接能跑的，用同目录的 MSI 或便携版。

---

## 0. 包里有什么

```
pdfcraft-main\
├─ Cargo.toml / Cargo.lock        工作区定义与锁定版本（0.22.0）
├─ .cargo\config.toml             CRAFT_I18N_DIR = craft-i18n（相对工作区根）
├─ craft-i18n.rev                 语言包应处的 commit
├─ craft-i18n\locales\zh-hans\    中文语言包（common.tsv + pdfcraft.tsv）
├─ crates\ai\                     本地 AI crate（手写 HTTP，打 127.0.0.1:11434）
├─ crates\ui-egui\                界面，含 i18n\extra\zh-hans.tsv（补充译文）
├─ crates\ui-egui\src\ui_font.rs  运行时界面字体（PDFCRAFT_UI_FONT / fonts\）
├─ apps\pdfcraft\                 图形界面入口
├─ apps\pdfcraft-cli\             命令行入口
└─ packaging\windows\             做 MSI / 便携版的脚本（可选）
```

- 不含 `target\`（构建产物）和 `craft-i18n\.git`（版本历史），所以包只有 30 MB 左右。
- `craft-i18n` **不能删**：文件在，界面才是中文；删了编译出来是英文界面。
- 0.22.0 与 0.21.x 的结构差异：上游把 `vendor\hayro-*` 换成了 `[patch.crates-io]` 指向 `craft-libs` 仓库（rev `9eb02be…`），首次编译需要联网拉一次依赖。

---

## 1. 环境要求

| 项 | 要求 | 本机现状 |
| --- | --- | --- |
| Rust | 1.100 左右，target `x86_64-pc-windows-msvc` | `C:\Users\26823\.cargo\bin`，**没进 PATH** |
| C++ 链接器 | MSVC 的 `link.exe` | VS 2022 Community 已装 |
| 网络 | 首次编译要拉 crates 与 `craft-libs`（git 依赖） | 已拉过，本地缓存可用 |
| 磁盘 | 首次编译 10 GB 以上 | — |
| 时间 | 第一次 10–30 分钟；之后增量 1–2 分钟 | — |
| AI | 本机 Ollama + `gemma3:4b` | 已确认：`gpt-oss:20b` / `gemma3:4b` / `mxbai-embed-large:335m` |

不需要 Node、不需要 Python、不需要任何云 API Key。

---

## 2. 解压

建议放到短路径，例如 `D:\PdfCraft\`（路径里少放空格和中文，能避开一堆工具链的小毛病）。解压后应能在目录里直接看到 `.cargo`、`crates`、`craft-i18n`、`Cargo.toml`。

---

## 3. 启动图形界面（第一次要编译）

```powershell
cd D:\PdfCraft\pdfcraft-main
$env:Path = "C:\Users\26823\.cargo\bin;" + $env:Path      # 本机 cargo 没进 PATH，每次新开窗口都要加
cargo run -p pdfcraft -- "D:\某个文档.pdf"
```

- 第一次 `cargo run` 会把几百个依赖编译出来（屏幕一直在滚 `Compiling …`，属正常），**之后启动是秒级**。
- 编译完成后就可以直接跑 exe，不必每次都经 cargo：

  ```powershell
  .\target\debug\pdfcraft.exe "D:\某个文档.pdf"
  ```

- 不带文件也能启动，进界面再 `文件 ▸ 打开`（Ctrl+O）：

  ```powershell
  cargo run -p pdfcraft
  ```

- 日常用的 release 版（首次编译更久，运行更快）：

  ```powershell
  cargo build --release --locked -p pdfcraft -p pdfcraft-cli
  # 产物在 target\release\  （或 target\x86_64-pc-windows-msvc\release\）
  ```

- 可用启动参数（`pdfcraft --help` 里也有）：`--page N`、`--zoom 150`、`--layout continuous`、`--theme dark`、`--new-window`、`--two-paged`、`--combine files…`、`--create-images images…`。
  `--control <文件>` 是给**自动化**用的控制通道（`pdfcraft-cli ui --control …`），日常使用不需要。

---

## 4. 界面：默认就是简体中文

中文不是运行时可选项，而是编译时嵌进去的：`.cargo/config.toml` 把 `CRAFT_I18N_DIR` 指到包内 `craft-i18n`，`build.rs` 把 `zh-hans` 编进二进制，并合并 `crates\ui-egui\i18n\extra\zh-hans.tsv` 里补的译文。所以只要目录完整，编出来就是中文。

**如果界面还是英文**，是偏好文件里存过旧的语言值（代码默认值被覆盖了）：

- 界面：`首选项 ▸ 语言 ▸ 简体中文`；
- 或命令行改偏好（可选）：

  ```powershell
  .\target\debug\pdfcraft.exe --control C:\TEMP\pdfcraft-ctl.sock "D:\某个文档.pdf"
  .\target\debug\pdfcraft-cli.exe ui --control C:\TEMP\pdfcraft-ctl.sock ui.set key=language value=zh-hans
  .\target\debug\pdfcraft-cli.exe ui --control C:\TEMP\pdfcraft-ctl.sock ui.quit
  ```

> `zh` / `zh_CN` / `zh-Hans*` 都会识别成 `zh-hans`；繁体是 `zh-hant`。

---

## 5. 用本地 AI（摘要 / 询问 / 翻译）

### 5.1 先确认模型服务在跑

```powershell
ollama list                                                   # 应看到 gemma3:4b
curl http://127.0.0.1:11434/api/tags                          # 或在浏览器打开这个地址
```

没有这个模型就先 `ollama pull gemma3:4b`；服务没起就 `ollama serve`（或在托盘里启动 Ollama）。

### 5.2 打开面板

左栏工具组「**所有工具**」→「**AI 助手**」，或用命令「**摘要**」「**询问**」「**翻译**」（命令只是打开面板，不会直接发请求）。

面板上有三个页签，一次只跑一件：

| 页签 | 做什么 | 怎么用 |
| --- | --- | --- |
| 摘要 | 把文档读成要点，每条带 `[p12]` 页码 | 点「摘要」，长文档会分块读，进度显示「正在读取页面 418/1004」 |
| 询问 | 根据文档正文回答你的问题 | 输入问题再点按钮；文档里没有的，它会说不知道 |
| 翻译 | 把正文译成目标语言（默认简体中文） | 选目标语言再点「翻译」；页码标记 `[pN]` 保留在同一段 |

细节：

- **只有点按钮才发请求**，切换页签会清掉上一轮结果。
- 设置里的「服务器」默认 `127.0.0.1:11434`，「模型」默认 `gemma3:4b`——模型名要和 `ollama list` 里**完全一致**。
- 回答语言默认就是简体中文，不用每次选。
- 结果只在面板里显示、可一键复制，**不会写回 PDF**（这是刻意设计，避免动到内容、批注与签名）。
- 页码 `[p3]` 可以点，点了会跳到那一页。
- 前提：文档要有**文字层**。纯扫描件先做 OCR（工具组「转换」里的 OCR「文字识别」），否则会提示先 OCR。
- 4B 模型在本机是每段 10–30 秒，页数越多越慢；单次请求超时 600 秒。

### 5.3 右键翻译选中的文字（只译这一段）

不必整篇翻。**先用「选择」工具在页面上拖选一段文字，再在选中的地方右键**，菜单里就有「**用 AI 翻译选中文字**」。

点它之后：

- 面板自动切到「翻译」页签并**立刻发一次请求**（这一步就是你点的同意，不用再按按钮）；
- 面板顶部出现一条横幅：`第 N 页 · M 个字符` + 「**清除选中文字**」，下面提示「只把选中的文字发给模型，一次请求完成」；
- 结果只这一段，页码标记 `[pN]` 保留；翻译目标语言用「译为」里的值（留空就是界面的语言，即简体中文）。

几条边界：

- 菜单项**只在真的有文字选区时出现**。选不中文字、或者拖出来的是对象/图片选择框，就不会有这一项——这是刻意设计（没选中就没有东西可发）。
- 选中后切到「摘要」或「询问」页签，选区会被清掉（选区只对翻译有意义）。
- 选中的文字**只发给本机模型**，和整篇翻译一样不写回 PDF。
- 扫描件没有文字层，选不出字；先做 OCR。

命令行里也能翻译整篇（`pdfcraft-cli tools` 里的 `ai_translate`，参数 `target_language`），但那条路径没有「只译选区」的入口——选区翻译是界面里点出来的动作。

---

## 6. 命令行（可选）

```powershell
.\target\debug\pdfcraft-cli.exe --help
```

常用：`info`（文档概况 JSON）、`text`（抽正文）、`render --page N --out x.png`、`edit`（旋转/删页/移动/插空白页）、`combine`、`extract`、`split`、`check`（批量体检）、`batch`（批量转换）。

**AI 的无头用法**有个坑：`doc` 参数只认 `doc_open` 返回的整数 id，所以要两步脚本：

```powershell
@'
[{"tool":"doc_open","args":{"path":"D:\\某个文档.pdf"}},
 {"tool":"ai_summarize","args":{"doc":1}}]
'@ | Set-Content $env:TEMP\steps.json -Encoding utf8

.\target\debug\pdfcraft-cli.exe run --script $env:TEMP\steps.json --report
```

三个 AI 工具：`ai_summarize`（`doc`）、`ai_ask`（`doc`+`question`）、`ai_translate`（`doc`+`target_language`）；都只读，`language` 缺省即简体中文。

给别的 AI 客户端接入：`.\target\debug\pdfcraft-cli.exe mcp`（MCP 走 stdin/stdout）。

---

## 7. 自己打包（可选）

```powershell
pwsh packaging\windows\package.ps1 -Arch x64
```

产出在 `dist\release\`：

- `pdfcraft-0.22.0-windows-x64.msi`（每台机器的安装包，WiX v5）
- `pdfcraft-0.22.0-windows-x64-portable.zip`（`pdfcraft.exe` + `pdfcraft-cli.exe` + `models\` + `portable.txt`）

脚本需要：Rust（MSVC 工具链）、Windows SDK（`rc.exe` / `signtool.exe`）、WiX v5（`dotnet tool install --global wix --version 5.0.2`）、联网下载 OCR 模型。exe 静态链接 C 运行库，机器上不需要装 VC++ 可再发行组件。

便携提示：把 `portable.txt` 放在 exe 旁边，设置、日志、崩溃恢复文件都会写进同目录的 `PdfCraftData`，不碰 `%APPDATA%`。

---

## 8. 常见问题

| 现象 | 原因与处理 |
| --- | --- |
| `cargo : 无法将“cargo”项识别为…` | 先跑 `$env:Path = "C:\Users\26823\.cargo\bin;" + $env:Path`（每次新开 PowerShell 都要加） |
| `link.exe` 找不到 / LNK 报错 | 装 VS「使用 C++ 的桌面开发」工作负载（本机已装） |
| 卡在 `Updating git repository craft-libs` | 0.22.0 的 hayro 走 git 依赖，GitHub 偶发超时；设 `$env:CARGO_NET_RETRY=10`、`$env:CARGO_HTTP_TIMEOUT=120` 后重试。**别用 `--offline`**（会直接报找不到 hayro）。 |
| 编译很慢、磁盘爆 | 属正常量级；腾空间用 `cargo clean`（会删掉 16 GB 的 `target`，下次要重新编译） |
| 界面是英文 | ①偏好里语言是旧值 → 首选项切「简体中文」；②确认解压目录里 `craft-i18n\locales\zh-hans\pdfcraft.tsv` 存在（只拷 `crates` 出来会丢语言包） |
| AI 报 `… is Ollama running?` | Ollama 没起或端口不是 11434；`ollama serve` 后重试 |
| AI 报模型不存在 | 面板里模型名与 `ollama list` 不一致；`gemma3:4b` 与 `gemma3` 在代码里按 `latest` 处理，但 `gemma3:4b` ≠ `gemma3:12b` |
| AI 提示先 OCR | 文档没有文字层，先用 OCR 加一层再试 |
| 摘要很慢、像卡住 | 4B 模型逐块读，页数多就是线性增长；进度行会一直更新 |
| 右键没有「用 AI 翻译选中文字」 | 只有真的选中了**文字**才出现；先切「选择」工具拖选一段，再右键 |
| 关了窗口 AI 还在跑 | 不会，进程退出即停；工作线程在页与页之间检查取消位 |

---

## 9. 自检（可选，确认这份代码是完整的）

```powershell
$env:Path = "C:\Users\26823\.cargo\bin;" + $env:Path
cargo check --workspace --all-targets --locked
cargo test -p pdfcraft-ui-egui --lib          # 0.22.0 实测 1045 passed / 1 ignored
cargo test -p pdfcraft-ui-egui --test it -- text_select   # 41 passed（含右键翻译）
cargo test -p pdfcraft-automation --lib       # 72 passed
cargo test -p pdfcraft-ai                     # 12 passed
cargo xtask i18n-report                       # zh-hans 缺口统计（多为次级文案）
```

中文覆盖率想看细一点：`cargo xtask i18n-report`，报告里的 `zh-hans … missing` 就是还没译的条数；补译文只需往 `crates\ui-egui\i18n\extra\zh-hans.tsv` 追加行，不用改代码（规则：`context<TAB>英文<TAB>译文`，`{占位符}`与结尾 `…` 两列都要有）。

> 注：0.22.0 新增的重命名对话框有一条上游测试依赖**操作系统自己的**错误文案；在中文 Windows 上、测试进程没有 CJK 字体时会因缺字形失败。这套源码里已在测试 harness 设置了 tofu 兜底，不影响正常使用。

---

## 10. 界面字体：换成任意字体（PingFang 等）

界面字体可换，**不用重编译**：程序启动时先看 `PDFCRAFT_UI_FONT` 这个环境变量，再看 exe 旁边的 `fonts\` 文件夹，把它当作界面主字体（Proportional 家族的第一位，中英文都先试它）。

### 10.1 放文件就行

```
PdfCraft-便携版\
├─ pdfcraft.exe
└─ fonts\
   └─ PingFangUI.ttc        ← 放这里，重启程序即生效
```

- 支持 `.ttc`（字体集合）、`.ttf`、`.otf`；`fonts\` 里有多个文件时按扩展名顺序取第一个。
- `.ttc` 里有多个字面（face）时，程序自动挑**能画简体中文**的那一个（探测字 `欢` U+6B22），所以 PingFang 的 TTC 直接用，不用指明第几个。
- 日志里会写清楚用了谁：`…\PdfCraftData\logs\pdfcraft.log` 里有一行
  `INFO … system_fonts: interface font: …\fonts\PingFangUI.ttc (face 0)`；没有这行说明没找到文件。
- 上限：单个文件 256 MB，最多试 16 个字面。

### 10.2 用环境变量指定别处的字体

```powershell
$env:PDFCRAFT_UI_FONT = "D:\字体\SourceHanSansSC-Regular.otf"
$env:PDFCRAFT_UI_FONT_INDEX = "2"     # 可选：指定 ttc 的第几个字面，从 0 数
.\pdfcraft.exe
```

### 10.3 说明两点

- 字体是**运行时从磁盘读**的，没有编进 exe；删掉 `fonts\` 就回到自带的 Inter + 系统中文字体。
- 随包分发别人的商业字体（PingFang 是苹果系统字体）要自己确认授权；只在自己电脑上用没问题。

---

## 11. MSI 安装包

`packaging\windows` 这套脚本已改好，可以直接出带字体的 MSI（内置 WiX v5）：

```powershell
# 1) 先编出 release exe（第 3 节）
# 2) 把要随安装包附带的界面字体放到一个目录，比如 C:\TEMP\ui-fonts\PingFangUI.ttc
# 3) 让 wix 在 PATH 里（dotnet 有 SDK 时：dotnet tool install --global wix --version 5.0.2）
$env:Path = "C:\TEMP\wix\pkg\tools\net6.0\any;" + $env:Path

pwsh packaging\windows\package.ps1 -Arch x64 -SkipBuild -BuiltWithoutCraftFonts -UiFontDir C:\TEMP\ui-fonts
```

产物在 `dist\release\`：

| 文件 | 说明 |
| --- | --- |
| `pdfcraft-0.22.0-windows-x64.msi` | 每机器安装：程序 + `models\` + `fonts\`（若传了 `-UiFontDir`） |
| `pdfcraft-0.22.0-windows-x64-portable.zip` | 绿色版 zip：exe×2 + `models\` + `fonts\` + `portable.txt` |

要点：

- **不传 `-UiFontDir` 就不装字体**；脚本还会清掉上次留在 stage 里的 `fonts\`，不会把上次的字体带进来。
- `-SkipBuild` 必须配 `-BuiltWithoutCraftFonts`（或 `CRAFT_FONTS_DIR`），这是脚本在确认「exe 里到底有没有内嵌 craft-fonts」。
- 脚本会依次做：`cargo xtask models` 取 OCR 模型 → 签名（没有证书就告警跳过）→ `wix build` → `test-msi.ps1`（安装场景与注册表契约，**不真的安装**）→ `check-install-policy.ps1` → 打便携 zip。
- 没有 git 的机器上，`package.ps1` 取不到 commit，先设 `$env:PDFCRAFT_BUILD_SHA`、`$env:PDFCRAFT_BUILD_DATE` 再跑。
- MSI 里的字体装到 `…\Program Files\ArtCraft\PdfCraft\fonts\`，和便携版的 `fonts\` 是同一个约定，程序照旧自动加载。
- 本机没有代码签名证书，产物是**未签名**的：首次运行 SmartScreen 要选「更多信息 ▸ 仍要运行」。
- 安装包需要管理员权限（写 Program Files + 注册表 + 开始菜单/资源管理器右键菜单）。
