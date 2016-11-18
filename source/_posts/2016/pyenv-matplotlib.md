---
title: 用pyenv重新整了个开发环境
tags:
  - 折腾
  - 装逼利器
categories:
  - python
date: 2016-11-16 19:57:42
---


好久没写博客了，因为一直不干正事。最近受老婆胁迫帮她做事，顺便自己也学习学习，值得记录一下，备忘。


[pyenv][pyenv]，一个模仿[rbenv][rbenv]而产生的工具，可以方便的管理各种python版本。命令基本和[rbenv][rbenv]差不多。用[pyenv][pyenv]和它的[virtualenv插件][pyenv-virtualenv]，来替代[virtualenv][virtualenv]是个不错的选择。

安装`pyenv`时报错，报错记录已经遗失，好象是系统缺少一些库文件之类的，运行`xcode-select --install`解决。

`pyenv`的配置方法文档上都有，没什么好说的，完成之后装[pandas][pandas]，[matplotlib][matplotlib]也很顺利，然后打开我的新宠`ipython notebook`输入
```python
import matplotlib.pyplot as plt
```
咦，报错了
```
UserWarning: Python is not installed as a framework. The MacOSX backend may not
work correctly if Python is not installed as a framework. Please see the
Python documentation for more information on installing Python as a framework on Mac OS X
```


原来`pyenv`安装的python并不是作为`Mac OS X`的framework安装的。`matplotlib`文档上给出了[解决方案][faq]，虽然这个方案是针对`virtualenv`的，不过`pyenv`应该也差不多，于是选择了看起来比较顺眼的在`zshrc`中添加函数的方法，修改函数如下
```bash
function frameworkpython {
    if [[ ! -z "$PYENV_VIRTUAL_ENV" ]]; then
        PYTHONHOME=$PYENV_VIRTUAL_ENV `which python` "$@"
    else
        /usr/local/bin/python "$@"
    fi
}
```
其实也就是把环境变量`VIRTUAL_ENV`换成了`PYENV_VIRTUAL_ENV`而已。然而不知道什么原因，并没有生效（当然运行`exec $SHELL`我是没有忘记的）


只好继续google，发现`pyenv`可以把python作为framework来安装:
```bash
env PYTHON_CONFIGURE_OPTS="--enable-framework CC=clang" pyenv install 3.5.2
```
用这个命令重装python之后，一切都OK了。



[rbenv]: https://github.com/rbenv/rbenv
[pyenv]: https://github.com/yyuu/pyenv
[virtualenv]: https://virtualenv.pypa.io/en/stable/
[pyenv-virtualenv]: https://github.com/yyuu/pyenv-virtualenv
[faq]: http://matplotlib.org/faq/virtualenv_faq.html#working-with-matplotlib-in-virtual-environments
[pandas]: http://pandas.pydata.org/
[matplotlib]: http://matplotlib.org/