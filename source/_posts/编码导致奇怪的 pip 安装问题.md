---
title: "编码导致奇怪的 pip 安装问题"
date: 2018-01-31 12:27:20
tags: [misc, python]
---

因为用的 VisualStudio, 所以 python 由 VisualStudio 下载放在 `C:\\Program Files (x86)\\Microsoft Visual Studio\\Shared\\Python36_64\` 中, 然而有时候用pip安装扩展会出现

<!-- more -->

```
Exception:
Traceback (most recent call last):
File "C:\\Program Files (x86)\\Microsoft Visual Studio\\Shared\\Python36_64\\lib\\site-packages\\pip\\compat\\__init__.py", line 73, in console_to_str
return s.decode(sys.__stdout__.encoding)
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xa1 in position 43: invalid start byte

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
File "C:\\Program Files (x86)\\Microsoft Visual Studio\\Shared\\Python36_64\\lib\\site-packages\\pip\\basecommand.py", line 215, in main
status = self.run(options, args)
File "C:\\Program Files (x86)\\Microsoft Visual Studio\\Shared\\Python36_64\\lib\\site-packages\\pip\\commands\\install.py", line 342, in run
prefix=options.prefix_path,
File "C:\\Program Files (x86)\\Microsoft Visual Studio\\Shared\\Python36_64\\lib\\site-packages\\pip\\req\\req_set.py", line 784, in install
**kwargs
File "C:\\Program Files (x86)\\Microsoft Visual Studio\\Shared\\Python36_64\\lib\\site-packages\\pip\\req\\req_install.py", line 878, in install
spinner=spinner,
File "C:\\Program Files (x86)\\Microsoft Visual Studio\\Shared\\Python36_64\\lib\\site-packages\\pip\\utils\\__init__.py", line 676, in call_subprocess
line = console_to_str(proc.stdout.readline())
File "C:\\Program Files (x86)\\Microsoft Visual Studio\\Shared\\Python36_64\\lib\\site-packages\\pip\\compat\\__init__.py", line 75, in console_to_str
return s.decode('utf-8')
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xa1 in position 43: invalid start byte
```

的错误, 问题就出在 `proc.stdout.readline()` 中, 因为中文版 windows 在控制台中字符页是 gbk, 所以默认的 utf-8 自然会解码错误, 所以把 `C:\Program Files (x86)\Microsoft Visual Studio\Shared\Python36_64\lib\site-packages\pip\compat__init__.py` 中的所有 utf-8 换成 gbk 就解决了. 