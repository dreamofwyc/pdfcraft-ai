<img width="1113" height="123" alt="image" src="https://github.com/user-attachments/assets/56bf223f-b90a-4377-9618-4cf6fbbf44fb" /># pdfcraft-ai
将storytold/pdfcraft中的项目，结合ollama本地部署模型，实现AI的翻译，总结功能

PdfCraft 便携版（解压即用，不需要安装 Rust / Cargo）

启动
----
双击 pdfcraft.exe 即可（首次启动约 3–5 秒）。
也可以把 PDF 拖到 pdfcraft.exe 上，或用命令行：
    pdfcraft.exe "D:\文档\某个文件.pdf"
想看界面参数：pdfcraft.exe --help

界面已经是简体中文，不需要任何设置。
如果显示英文：首选项 ▸ 语言 ▸ 简体中文。

AI（摘要 / 询问 / 翻译）
------------------------
用的是本机 Ollama，不需要联网、不需要 API Key。
前置条件（只需一次）：
    1) 装好 Ollama 并保持运行（托盘里的 Ollama 不要退出）
    2) ollama pull gemma3:4b
用法：
    1) 打开一个 PDF（文件 ▸ 打开，或 Ctrl+O）
    2) 左栏「所有工具」→「AI 助手」
    3) 三个页签：摘要 / 询问 / 翻译，点按钮才会开始
       - 摘要：要点列表，每条带 [p12] 页码，可点击跳页
       - 询问：按文档正文回答；文档里没有的它会说不知道
       - 翻译：默认译成简体中文，页码标记保留
注意：AI 只读取文档、不改动文件；结果可一键复制。
     扫描件（没有文字层）请先做 OCR（工具组「转换」→ OCR 文字识别）。

OCR 模型
--------
models\ 目录里已经打包了 OCR 模型（text-detection / text-recognition）。
删掉这个目录会让 OCR 不可用，其余功能不受影响。

命令行工具
----------
同目录的 pdfcraft-cli.exe：
    pdfcraft-cli.exe --version
    pdfcraft-cli.exe --help
    pdfcraft-cli.exe info 某个.pdf
    pdfcraft-cli.exe text 某个.pdf
    pdfcraft-cli.exe render --page 1 --out 第1页.png
AI 无头用法（doc 只认 doc_open 返回的整数 id，要两步脚本）：
    @'
    [{"tool":"doc_open","args":{"path":"D:\\某个文档.pdf"}},
     {"tool":"ai_summarize","args":{"doc":1}}]
    '@ | Set-Content $env:TEMP\steps.json -Encoding utf8
    pdfcraft-cli.exe run --script $env:TEMP\steps.json --report

数据与设置存在哪里
------------------
本目录里的 portable.txt 让程序把设置、日志、崩溃恢复文件写进同目录的
PdfCraftData\，不碰 %APPDATA%。整个文件夹可以拷到 U 盘或别的电脑继续用。
删掉 portable.txt 就改回 Windows 的标准用户目录。

卸载
----
直接删掉整个文件夹即可，没有注册表项、没有服务、没有后台进程。
