<!-- TOC -->

- [简介](#简介)
- [格式类型](#格式类型)
- [转换 doc 为 docx 文件](#转换-doc-为-docx-文件)
- [读写 docx 文件](#读写-docx-文件)
- [小结](#小结)

<!-- /TOC -->

## 简介

Word 是非常常见的文件格式, 可以使用 python 来操作 Word 文档.

## 格式类型

Word 有两种类型的文档, 文件后缀分别为 `.doc` 和 `.docx`.
前者是 Office 2003 时的格式, 后者是 Office 2007 以后推出的新格式.
通常来说, 我建议大家使用新版本, 但是免不了很多时候会使用老版本.

## 转换 doc 为 docx 文件

这是在 linux 上使用的, 需要借助 soffice.
如果需要在 windows 上操作, 建议使用 win32com.
这里有一个[参考连接](https://stackoverflow.com/questions/38468442/multiple-doc-to-docx-file-conversion-using-python).

```python
import glob
import subprocess
from pathlib import Path

"""
借助 soffice 将 doc 转换为 docx
"""

base_dir = Path(__file__).resolve().parent
print(base_dir)

doc_list = glob.glob(base_dir.as_posix() + "/**/*.doc", recursive=True)
print(doc_list)


for doc in doc_list:
    subprocess.call(
        [
            "soffice",
            "--headless",
            "--convert-to",
            "docx",
            "--outdir",
            Path(doc).parent.as_posix(),
            doc,
        ]
    )


doc_list = glob.glob(base_dir.as_posix() + "/**/*.docx", recursive=True)
print(doc_list)s
```

## 读写 docx 文件

这里推荐使用 `python-docx`.

```bash
pip install python-docx
```

简单使用

```python
from docx import Document

doc = Document(doc_path)

# 读取表格列表
table_list = doc.tables
# 读取段落
paragraph_list = [x.text.strip() for x in doc.paragraphs if x.text.strip()]
```

更多的内容可以参考[官方文档](https://python-docx.readthedocs.io/en/latest/index.html).

## 小结

现在, 我们已经可以使用 python 转换 doc 为 docx 文件, 并从中读取内容了.
