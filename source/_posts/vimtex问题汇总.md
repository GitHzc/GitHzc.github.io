---
title: vimtex问题汇总
date: 2019-12-11 21:11:16
tags:
- latex
- 问题解决
categories: 软件
---
__这个文档主要用来记录在mac上使用[vimtex](https://github.com/lervag/vimtex)时遇到问题的解决方法__

- Critical Package ctex Error: CTeX fontset 'mac' is unavailable in current mode.  
__solution__：vimtex默认使用pdflatex作为编译器，编译中文时要改成xelatex。在tex文件的开头加上  
```%! TEX program = xelatex```  
重启vimtex!!!

