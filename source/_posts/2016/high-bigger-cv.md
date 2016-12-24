---
title: 高逼格简历制作指北
tags:
  - 折腾
  - 装逼利器
  - 工具
  - pandoc
categories:
  - 折腾
date: 2016-12-24 15:48:42
---



物以稀为贵，说的是越少的东西逼格越高，譬如本文标题用指北而不是指南，瞬间就高出了一格。简历也一样嘛，pdf的后缀名的总感觉比doc高贵那么几分，想来HR定然会高看一眼。
所以，本指北将教给读者这样一种方法，只要简单写写，就可以生成漂亮的在线版的HTML简历和一模一样的pdf版本。

##### 开始：
* 安装[Pandoc][pandoc]和[wkhtmltopdf][wkhtmltopdf]，[Pandoc][pandoc]用来做格式转换，多说两句，这货可牛逼了，[Haskell][haskell]写的杀手级应用，支持各种文档格式互转。[wkhtmltopdf][wkhtmltopdf]作为pdf后端，看名字就知道是HTML转pdf用的
* 简历写好，运行下面的命令，就能生成两种格式的简历了
```bash
pandoc -s <简历文件> -o 简历.html
pandoc -s -t html5 <简历文件> -o 简历.pdf
```

##### 那么：
* 上面的两步生成的简历只有最简单的排版怎么办。网上找一个适配[Pandoc][pandoc]的[CSS](http://pandoc.org/demo/pandoc.css)改改，命令里加上`-c pandoc.css`，排版和色彩就都解决了
* 甚至可以在简历里生成流程图什么的，文档上有例子，一下子没看明白就不看了，反正放图还有其他方法

##### 最后:
* 只能说[Pandoc][pandoc]太牛逼，基本上输入什么格式都可以，当然作为一个连简历都要生成的懒人你一定会选择一个简单的格式。
* 这其实不过是**用生成文档的方法来生成简历**罢了，以前用[Sphinx][sphinx]处理[reST][rst]格式的文本，对样式的操作很有限，生成pdf也要填很多坑。现在鸟枪换炮，[Pandoc][pandoc]通吃一切
* 简历最重要的还是内容，长得再漂亮也不过是一副臭皮囊，而已...



[haskell]: https://www.haskell.org/
[pandoc]: http://pandoc.org/
[wkhtmltopdf]: http://wkhtmltopdf.org/
[sphinx]: http://www.sphinx-doc.org/
[rst]: http://docutils.sourceforge.net/rst.html