opendataloader-pdf 使用指南
================================

本文档记录 opendataloader-pdf 的安装、使用过程及踩坑经验。

环境信息
--------

- OS: Ubuntu 24.04
- Python: 3.12
- opendataloader-pdf: 2.0.2

安装过程
--------

系统 Python 为 externally-managed 环境（Ubuntu 24.04 默认），无法直接 ``pip install``，
需要先创建虚拟环境::

    # 创建虚拟环境
    python3 -m venv .venv

    # 激活虚拟环境
    source .venv/bin/activate

    # 安装 opendataloader-pdf
    pip install -U opendataloader-pdf

验证安装::

    python -c "import opendataloader_pdf; print('import success')"

模块导出::

    convert / convert_generated  - 转换功能
    run / run_jar / runner       - 运行功能
    cli_options_generated        - CLI 选项
    wrapper                      - 包装器

convert 函数签名
-----------------

``opendataloader_pdf.convert`` 主要参数：

============= ============ =====================================
参数           类型          说明
============= ============ =====================================
input_path    str / list    输入 PDF 文件路径或目录（必填）
output_dir    str           输出目录，默认与输入文件同目录
format        str / list    输出格式，见下方支持格式
password      str           加密 PDF 的密码
quiet         bool          静默模式，抑制控制台日志
pages         str           提取指定页，如 ``1,3,5-7``
keep_line_breaks  bool      保留原始换行
image_output  str           图片模式：off / embedded / external
image_format  str           图片格式：png / jpeg
image_dir     str           图片输出目录
table_method  str           表格检测：default / cluster
reading_order str           阅读顺序：off / xycut
sanitize      bool          敏感数据脱敏
============= ============ =====================================

支持的输出格式：

- ``json`` （默认）
- ``text``
- ``html``
- ``pdf``
- ``markdown``
- ``markdown-with-html``
- ``markdown-with-images``

pdf_convert.py 封装脚本
------------------------

路径：``scripts/pdf_convert.py``

该脚本对 opendataloader-pdf 的 ``convert`` 函数做了命令行封装，便于日常使用。

基本用法::

    # 激活虚拟环境
    source .venv/bin/activate

    # 转 markdown，输出到指定目录
    python scripts/pdf_convert.py docs/pdf/file.pdf -f markdown -o output/

    # 转 text，提取指定页码
    python scripts/pdf_convert.py docs/pdf/file.pdf -f text --pages 1,3,5-7

    # 静默模式
    python scripts/pdf_convert.py docs/pdf/file.pdf -f json --quiet

    # 查看完整帮助
    python scripts/pdf_convert.py -h

踩坑记录
--------

1. **externally-managed-environment 错误**

   Ubuntu 24.04 的系统 Python 受 PEP 668 保护，直接 ``pip install`` 会报错。
   解决方案：使用 ``python3 -m venv .venv`` 创建虚拟环境，在虚拟环境中安装。

2. **PDF 文件内部格式不规范导致转换失败**

   测试 ``BPF.Performance.Tools.2019.12.pdf`` 时，verapdf 解析器报 cross-reference
   table 格式错误，导致 Java 后端异常退出。这是 PDF 文件本身的问题，与工具无关。
   解决方案：尝试用其他工具修复 PDF，或使用 ``--pages`` 参数跳过问题页面。

3. **图片输出**

   默认 ``image_output`` 为 ``external``，会生成独立的图片文件目录。
   如不需要图片，使用 ``--image-output off``；如需内嵌 Base64，使用
   ``--image-output embedded``。
