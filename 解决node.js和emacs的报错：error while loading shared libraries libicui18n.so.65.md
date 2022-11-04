# 解决node.js和emacs的报错：error while loading shared libraries: libicui18n.so.65

> 博主安装了npm和emacs，安装完成之后在命令行启动的时候遇到了一下错误。

- emacs: error while loading shared libraries: libMagicWand.so.5: cannot open shared object file: No such file or directory.
- node: error while loading shared libraries: libicui18n.so.65: cannot open shared object file: No such file or directory.

```
根据错误得知是缺少icu的65动态链接库，网上搜索了一下，大多数都是说安装icu-dev，不过我在manjaro的包管理器中和aur的源中并没有找到这个软件，先解决npm的问题先
```

> 在决定要做什么之前，我先查看了系统中icu的版本，使用`sudo pacman -Q icu`的到的结果是`icu 64.1-2`，猜测是共享库的版本不对，node.js无法调用，于是尝试更新icu。

```shell
sudo pacman -Syy    # 刷新源列表
sudo pacman -Sy icu # 安装icu
sudo pacman -Q icu  # 查看icu的版本
```

```shell
icu 65.1-2
```

**安装成功，尝试重新启动npm，成功！**

```shell
npm -v
```

![](http://images.xiao-hui.net/gogs_565584326/Files/raw/master/20191229/Screenshot_20191228_231918.png)

---

> 接下来解决emacs的报错；更新了icu之后，启动emacs的时候还是报错，让我疑惑了很久，我想着可能是emacs依赖文件的库版本不对，我试过想要找到是哪个库，找到之后重新创建链接，但是没有成功，在stackecchange上的一个帖子得到了启发，于是我想着更新系统，把所有软件都更新，没准就解决emacs的问题了

```shell
sudo pacman -Sc   #删除缓存中的软件
sudo pacman -Syuu #更新系统
```

**在更新的途中还遇到了一下问题：**

```json
错误：无法提交处理 (无效或已损坏的软件包)
发生错误，没有软件包被更新。
```

> 使用`vim`打开`/etc/pacman.conf`修改一下内容，没有一下内容则添加

- [ ] **原文件：**

  ```powershell
  [archlinuxcn]
  SigLevel = Optional TrustedOnly
  Server = http://mirrors.163.com/archlinux-cn/$arch
  ```

- [x] **修改后的文件：**

  ```powershell
  [archlinuxcn]
  #SigLevel = Optional TrustedOnly
  SigLevel = Never
  Server = http://mirrors.163.com/archlinux-cn/$arch
  ```

- [ ] **没有则在文件末尾添加**

  ```powershell
  [archlinuxcn]
  #SigLevel = Optional TrustedOnly
  SigLevel = Never
  Server = http://mirrors.163.com/archlinux-cn/$arch
  ```

**这个时候在执行更新：`sudo pacman -Syuu`，等待更新完成后在命令行键入`emacs`**

![](http://images.xiao-hui.net/gogs_565584326/Files/raw/master/20191229/Screenshot_20191228_231726.png)

---

