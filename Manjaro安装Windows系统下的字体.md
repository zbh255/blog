博主最近安装了Manjaro系统，来替代win系统作为主力系统，一番使用体验下来，Manjaro写代码是比win好的多，配置环境和调试代码都很方便，终端的体验比win下好不少，也没有win下的迷之错误，比如`bpython`的`I/O`错误，我至今都无法解决这个错误，Manjaro下的`bpython`就没有这个问题。

---
那么，回归正题，Manjaro自带的字体还是不怎么好看，自带的等宽字体更是不忍直视，我就因为等宽字体的问题没打开过vs code一个星期，我还是喜欢win下的字体。
要把win系统的字体换过来其实很简单，只要把win系统存放字体的目录拷贝到Manjaro存放字体的目录，最后刷新字体缓存就ok了，具体操作如下：

> 一般来说Manjaro可以访问win系统的系统盘，win系统的字体文件一般都放在`C:\\Windows\\Fonts`，Manjaro的字体文件在`/usr/share/fonts`

> 因为我们是在Manjaro系统上操作，目录应该按照当前系统的标准，`打开你的文件管理器->进入你win系统所在的硬盘->找到字体文件所在的文件夹并复制该文件夹路径`，比如我的是：`/run/media/harder/8A7C32007C31E819/Windows/Fonts/`

- 将该文件夹复制到Manjaro存放字体的目录:
```shell{.line-numbers}
sudo mkdir /usr/share/fonts/vista #新建一个文件夹，用于存放复制的字体文件
sudo cp -r /run/media/harder/8A7C32007C31E819/Windows/Fonts/ /usr/share/fonts/vista
```
- 随后切换到该目录，刷新字体的缓存
```shell{.line-numbers}
sudo cd /usr/share/fonts/vista
sudo fc-cache -fv
```
- 至此，字体安装完成!

---

![test](http://images.xiao-hui.net/gogs_565584326/Files/raw/master/20191127/Screenshot_20191127_214341.png)
