> **最近由于有高性能异步操作的需求，`Cpython`不能满足我的需求，于是安装了`PyPy`作为`python`解释器，`PyPy`确实比`Cpython`快了不少。但是平时要测试代码想要实时查看输出结果还是交互式`shell`友好，于是想到为PyPy安装一个交互式`shell`，baidu了一圈，没有找到安装教程，于是自己摸索摸索，安装成功了，写篇博客记录一下！**

  - 博主使用的pythonShell是`ptpython`，其他交互式shell的安装方法雷同
  - 博主的PyPy解释器版本是3.6.9
---

> **为PyPy安装`ptpython`**

[shell]
sudo pypy3 -m pip install python
[/shell]

![](http://images.xiao-hui.net/gogs_565584326/Files/raw/master/20191221/Screenshot_20191221_011510.png)

---
> **启动`ptpython`**

[shell]
sudo pypy3 -m ptpython
[/shell]

![](http://images.xiao-hui.net/gogs_565584326/Files/raw/master/20191221/Screenshot_20191221_011658.png)


