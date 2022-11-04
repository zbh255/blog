# Linux下按键截屏工具-Screenkey的使用方法

------

> **`参考文章`**
>
> - https://www.maketecheasier.com/display-keystrokes-screencasts/
>
> **`源-AUR`**
>
> - https://aur.archlinux.org/packages/screenkey-git/
>
> **`官方网址`**
>
> - https://seminar.io/projects/screenkey/
>
> **`GitLab地址`**
>
> - https://gitlab.com/screenkey/screenkey

> `screenkey官网的介绍`
>
> - __Screenkey是一个截屏视频工具，用于显示受Screenflick for Mac OS启发并且最初基于key-mon项目的键。创建截屏视频非常有用，它也是功能强大的教学工具。__

~~**`安装到你的Linux电脑，博主的电脑系统是Manjaro`**~~

```shell
# Manjaro使用Pacman安转或者AUR
sudo pacman -S screenkey

# Ubuntu使用官方源即可
sudo apt-get install screenkey
```
---
### **`<启动>`**

`使用命令行启动`
```shell
nohup screenkey #后台运行screenkey
```
> 或者通过搜索找到screenkey，点击启动即可，启动之后在托盘区看到![](/home/harder/桌面/Screenshot_20200428_195438.png)之后就点击该图标，点击首选项，大概是这样的
>
> ![](/home/harder/桌面/Screenshot_20200428_195833.png)

![](/home/harder/桌面/Screenshot_20200428_195718.png)

**`接下来我们来解析每一项设置的用处与含义＞＜`**


> - **`Time`**
>   - `Display for {  } seconds`，这个选项的含义是触发了按键显示在屏幕上后，显示块在屏幕中停留的时间，记是为==（S/秒）==
>   - `Persistent window`，这个选项开启之后，显示块就会常驻显示屏，不会消失
> - **`Position`**
>   - `Screen`~~设置不了，盲猜是多屏显示？~~
>   - `Position`，设置显示块在屏幕中的==方位==，默认`bottom`为下方，`top`为上方，`center`为中间
>   - `select window/region`~~也设置不了，盲猜是多屏支持的选项？~~
> - **`Aspect`**
>   - `Font`，设置显示块中显示的字体==样式==，有多种字体可以`选择`
>   - `Size`，该选项能设置字体的大小，有三种选项，默认为`small`
>   - `Font color`，该选项可以设置`字体和显示块`的颜色，++**`第一个设置字体，第二个设置显示块背景色`**
>   - `Opacity`，该选项是设置显示块的`透明度`
> - **`Keys`**
> - `Keyboard mode`，监听键盘键位的模式，默认（`composed`）模式下不能输入中文，`translated`表示监听部分功能键，`keysyms`表示全部输入键都会被监听，`raw`则是表示，输出到显示块中的键位都会被转换成英文大写
>   - `5Backspace mode`，不清楚，设置没效果
>   - `Modifiers mode`，键位显示在`显示块`中的样式，测试
>   - [ ] `Show Modifier sequences only`
>   - [ ] `Alay show Shift`，`Shift`键在默认的情况下组合敲击是不显示的，你可以开启它
>   - [x] `Show Whitespace characters`，显示空白字符
>   - [x] `Compress repeats after { }`，改选项表示`screenkey`会在你在显示块持续时间内连续敲入多个快捷键之后，在`n`个之后就不会在显示完整的内容，取而代之的是：**~...7x~**，诸如此类表示更多的信息

> **`注意：`**
>
> - 在输入密码的时候，要是你在使用`录屏工具`的话，或者你不想让别人看到你的密码，可以使用`Ctrl+Ctrl（左右两个Ctrl一起按）`快捷键关闭显示，之后在重新打开



<video id="video" controls="" preload="none" poster="/home/harder/桌面/Screenshot_20200428_205246.png">
<source id="mp4" src="/home/harder/2020-04-28 20-39-55.mp4" type="video/mp4">
</video>

