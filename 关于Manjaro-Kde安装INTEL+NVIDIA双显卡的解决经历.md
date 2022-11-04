# 关于Manjaro-Kde安装INTEL+NVIDIA双显卡的解决经历

> ~~参阅文章与资源~~
>
> - [显卡切换脚本](https://github.com/linesma/manjaroptimus-appindicator)
> - [Csdn](https://blog.csdn.net/sherpahu/article/details/103193009)
> - https://www.v2ex.com/t/630119
> - https://tieba.baidu.com/p/6340530678
> - [Arch Wiki](https://wiki.archlinux.org/)
>
> ~~个人电脑配置如下，提供个参考~~
>
> ​                               **Terminal**: konsole  
> ​                               **Terminal Font**: Source Code Pro [ADBO] 11  
> ​                               **CPU**: Intel i5-8250U (8) @ 3.400GHz  
> ​                               **GPU**: NVIDIA GeForce MX150  
> ​                               **GPU**: Intel UHD Graphics 620  
> ​                               **Memory**: 4564MiB / 7846MiB  

---

##### ◐ 一次失败的尝试 --optimus-manager

> 起初，博主跟着百度贴吧和v2ex的帖子安装`optimus-manager`作为双显卡切换程序，安装NVIDIA闭源驱动错误提示`An error occurred while performing the step: "Building kernel modules". See `，在此之前要做点准备工作----把原来的开源驱动禁用
>
> - 在`/etc/modprobe.d`目录下创建`blacklist-nouveau.conf`，并写入以下内容并Save，之后重启
>
>   ```http
>   blacklist nouveau
>   options nouveau modeset=0
>   ```
>
> - 之后卸载`Nouveau`，重启以应用更改
>
>   ```shell
>   sudo pacman -Rsn xf86-video-nouveau
>   reboot
>   ```
>
>   ```shell
>   lsmod | grep nou # 使用该命令查看是否禁用成功，无输出则成功
>   ```

> >---
>
> 后来才发现Manjaro设备管理器中有官方提供的闭源驱动，博主自己安装的是：`video-nvidia-440xx``optimus-manager`没有`gui`界面，管理起来比较麻烦，于是下载了`optimus-manager-qt-kde`，该软件位于`pacman`官方库中，安装之后一定要开启服务，该软件的服务默认是关闭状态
>
> ```shell
> systemctl enable optimus-manager.service # 开机自启动
> systemctl start optimus-manager.service  # 启动
> systemctl status optimus-manager.service # 查看运行状态
> ```
>
> 如果没有的话，请使用`yay`，其他的桌面环境请使用`optimus-manager-qt`。我通过设置界面swich nvidia之后，`reboot`之后再也进不了`Gui`桌面。
>
> 于是我猜测是`NVIDIA`的显卡驱动不兼容，于是进入Manjaro的`tty`界面（Ctrl+Alt+F3）尝试切换为核显，
>
> ```shell
> optimus-manager --switch intel
> optimus-manager --print-start
> reboot
> ```
>
> 我通过上述命令尝试切换为`intel`显卡，但是重启之后并没有成功，尝试使用：
>
> ```shell
> sudo pacman -Rsn optimus-manager-qt-kde
> ```
>
> 卸载了`optimus-manager-qt-kde`重启之后能够正常显示桌面，于是尝试：
>
> ```shell
> sudo pacman -Syu
> ```
>
> 更新系统，卸载带有`bumblebee`的驱动都卸载，卸载`video-linux`，禁用`Nouveau`将在尝试安装切换显卡，记得重新安装之后启动`optimus-manager`的服务
>
> - 在`/etc/modprobe.d`目录下创建`blacklist-nouveau.conf`，并写入以下内容并Save，之后重启
>
>   ```http
>   blacklist nouveau
>   options nouveau modeset=0
>   ```
>
>   ---
>
> ```shell
> systemctl enable optimus-manager.service # 开机自启动
> systemctl start optimus-manager.service  # 启动
> optimus-manager --switch nvidia
> reboot
> ```
>
> 很遗憾，开机还是黑屏。

---

##### ☼第二次尝试，通过切换脚本

> 作者的这个脚本是针对不同发行版，博主是KDE桌面环境，所以应该安装`optimus-switch-sddm`，如果你是Gnome，xfce等请参照项目说明，同第一次一样，同样要卸载开源驱动（`Bumblebee` and `Nouveau`），之后在硬件设定中（设置->硬件设定）就可以快捷的安装（右击你想安装的版本即可）闭源驱动，博主安装的版本是`video-nvidia-440xx`，接下来安装各种所需依赖

```shell
linuxxx-headers - linux内核的头文件，编译驱动时会用到，xxx表示你当前使用的linux内核的版本：uname -a 可以查看，比如博主（Linux harder-the-pc 5.5.16-1-MANJARO #1 SMP PREEMPT Wed Apr 8 10:07:00 UTC 2020 x86_64 GNU/Linux）就填linux55-headers
acpi_call-dkms -显卡电源管理工具的依赖项
xorg-xrandr - 这个不太清楚
xf86-video-intel -Intel显卡驱动
git - 版本控制工具，一会把脚本clone到本地会用到
```

**`Open Install`**

 ```shell
sudo pacman -S linuxXXX-headers acpi_call-dkms xorg-xrandr xf86-video-intel git
sudo modprobe acpi_call
 ```

**`我的所有操作都在我的主目录下操作，即：~/，使用cd ~/切换`**

```shell
git clone https://github.com/dglt1/optimus-switch-sddm.git
cd ~/optimus-switch-sddm
chmod +x install.sh
sudo ./install.sh
```

> **等待安装完成后可使用`sudo set-intel.sh`切换intel核显，使用`sudo set-nvidia.sh`切换nvidia显卡**

*使用`reboot`重启生效后会发现桌面字体变小，标题栏字体全乱，这是因为`NVIDIA`显卡的`DPI`没有设置好，修改`脚本安装目录/switch/nvidia`下的`nvidia-xorg.conf`和`/etc/X11/xorg.conf.d`目录下的`99-nvidia.conf`，取消注释`#Option  "DPI" "96 x 96"    #adjust this value as needed to fix scaling`；即改为`Option  "DPI" "96 x 96"    #adjust this value as needed to fix scaling`重启便可恢复正常*

---

<u>**`使用nvidia-smi查看显卡运行情况`**</u>

```shell
Fri Apr 17 01:29:49 2020       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 440.82       Driver Version: 440.82       CUDA Version: 10.2     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce MX150       Off  | 00000000:01:00.0 Off |                  N/A |
| N/A   42C    P0    N/A /  N/A |    679MiB /  2002MiB |      4%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    0      1012      G   /usr/lib/Xorg                                286MiB |
|    0      1414      G   /usr/bin/kwin_x11                            131MiB |
|    0      1428      G   /usr/bin/plasmashell                          63MiB |
|    0      1434      G   /usr/bin/latte-dock                           21MiB |
|    0      1860      G   ...uest-channel-token=16734543256808203157     4MiB |
|    0      2330      G   ...AAAAAAAAAAAACAAAAAAAAAA= --shared-files   145MiB |
|    0      3772      G   ...gram Files\Tencent\WeChat\WeChatApp.exe     2MiB |
|    0      4970      G   /usr/bin/systemsettings5                      14MiB |
+-----------------------------------------------------------------------------+

```









