---
1. 博主最近在内网的服务器部署了`gogs`作为管理`git`的管理系统，安装过程很简单，下载好二进制包，设置好选项就安装成功了，但是每次都要运行`./gogs web`颇为麻烦，于是参考了`官方文档`的方法制作自启动文件：

    - 我使用了官方的启动脚本，使用这条命令：`cp /home/git/gogs/scripts/systemd/gogs.service /usr/lib/systemd/system/`，切换到git用户`su git`，运行服务`systemctl start gogs.service`，使用`systemctl status gogs.service`查看服务有没有启动成功：

```json{.line-numbers}
● gogs.service - Gogs
   Loaded: loaded (/usr/lib/systemd/system/gogs.service; enabled; vendor preset: disabled)
   Active: failed (Result: start-limit) since 二 2020-01-14 16:17:32 CST; 6s ago
  Process: 49975 ExecStart=/home/git/gogs/gogs web (code=exited, status=1/FAILURE)
 Main PID: 49975 (code=exited, status=1/FAILURE)

1月 14 16:17:32 bogon systemd[1]: Unit gogs.service entered failed state.
1月 14 16:17:32 bogon systemd[1]: gogs.service failed.
1月 14 16:17:32 bogon systemd[1]: gogs.service holdoff time over, scheduling restart.
1月 14 16:17:32 bogon systemd[1]: Stopped Gogs.
1月 14 16:17:32 bogon systemd[1]: start request repeated too quickly for gogs.service
1月 14 16:17:32 bogon systemd[1]: Failed to start Gogs.
1月 14 16:17:32 bogon systemd[1]: Unit gogs.service entered failed state.
1月 14 16:17:32 bogon systemd[1]: gogs.service failed.
```
---
2. 我受到了思否一个帖子的启发：([传送门]("https://segmentfault.com/q/1010000008292281"))，博主在`git`用户下使用命令行`./gogs web`来启动gogs服务报错：
```shell
2020/01/14 16:18:02 [FATAL] [...g/setting/setting.go:591 NewContext()] Expect user 'git' but current user is: root
```
   - > 博主想着可能是我的gogs配置文件设置的用户为`root`，才导致`git`用户无法启动，于是尝试修改gogs的配置文件，gogs的配置文件在`gogs安装目录/custom/conf/api.ini`, 将`RUN_USER`的值该为`git`

```ini
APP_NAME = Gogs
RUN_USER = root #改为git
RUN_MODE = prod

[database]
DB_TYPE  = mysql
HOST     = 127.0.0.1:3306
NAME     = gogs
USER     = gogs
PASSWD   = Cok774..
SSL_MODE = disable
PATH     = data/gogs.db

[repository]
ROOT = /root/gogs-repositories

[repository.upload]
FILE_MAX_SIZE = 5000

[server]
DOMAIN           = localhost
HTTP_PORT        = 3000
ROOT_URL         = http://localhost:3000/
DISABLE_SSH      = false
SSH_PORT         = 22
START_SSH_SERVER = false
OFFLINE_MODE     = false

[mailer]
ENABLED = false

[service]
REGISTER_EMAIL_CONFIRM = false
ENABLE_NOTIFY_MAIL     = false
DISABLE_REGISTRATION   = false
ENABLE_CAPTCHA         = true
REQUIRE_SIGNIN_VIEW    = false

[picture]
DISABLE_GRAVATAR        = false
ENABLE_FEDERATED_AVATAR = false

[session]
PROVIDER = file

[log]
MODE      = file
LEVEL     = Info
ROOT_PATH = /server/gogs/log

[security]
INSTALL_LOCK = true
SECRET_KEY   = n7g4p9Rt6HgXD5m
```
---
3. 尝试再次启动`./gogs web`，还是报错。各种方法都是过了，使用`git`用户运行总是启动不了。考虑到是内网的服务器，不用考虑安全性的问题，使用`root`用户启动也可以，我根据刚才那个帖子的`gogs.service`的配置来修改这个文件，将所有关于`git`用户的配置项都换成`root`用户。

- 原文件

  ```ini{.line-numbers}
  [Unit]
  Description=Gogs
  After=syslog.target
  After=network.target
  # 数据库，需要的就取消注释吧
  #After=mysqld.service
  #After=postgresql.service
  #After=memcached.service
  #After=redis.service
  
  [Service]
  # 修改工作目录「WorkingDirectory」和启动命令「ExecStart」
  # 如果不需要使用git用户和git用户组来启动的话就把User和Group注释掉，注意Environment也对应要修改
  ###
  Type=simple
  User=git
  Group=git
  WorkingDirectory=/home/git/gogs
  ExecStart=/home/git/gogs/gogs web
  Restart=always
  Environment=USER=git HOME=/home/git
  
  [Install]
  WantedBy=multi-user.target
  ```

- 修改后的文件

  ```ini{.line-numbers}
  [Unit]
  Description=Gogs
  After=syslog.target
  After=network.target
  After=mariadb.service mysqld.service postgresql.service memcached.service redis.service
  
  [Service]
  # Modify these two values and uncomment them if you have
  # repos with lots of files and get an HTTP error 500 because
  # of that
  ###
  #LimitMEMLOCK=infinity
  #LimitNOFILE=65535
  Type=simple
  User=root
  Group=root
  WorkingDirectory=/server/gogs
  ExecStart=/server/gogs/gogs web
  Restart=always
  Environment=USER=root HOME=/root
  
  # Some distributions may not support these hardening directives. If you cannot start the service due
  # to an unknown option, comment out the ones not supported by your version of systemd.
  ProtectSystem=full
  PrivateDevices=yes
  PrivateTmp=yes
  NoNewPrivileges=true
  
  [Install]
  WantedBy=multi-user.targe
  ```

---
> 更改成功后，使用`systemctl daemon-reload`重新加载，然后使用`systemctl start gogs.service`启动gogs服务，ps：别忘了把gogs的配置改回，使用`systemctl status gogs.service`查看gogs的运行情况：

```cpp
● gogs.service - Gogs
   Loaded: loaded (/usr/lib/systemd/system/gogs.service; enabled; vendor preset: disabled)
   Active: active (running) since 二 2020-01-14 16:24:38 CST; 5s ago
 Main PID: 50980 (gogs)
    Tasks: 10
   Memory: 30.3M
   CGroup: /system.slice/gogs.service
           └─50980 /server/gogs/gogs web

1月 14 16:24:38 bogon systemd[1]: Started Gogs.
1月 14 16:24:38 bogon gogs[50980]: 2020/01/14 16:24:38 [TRACE] Custom path: /server/gogs/custom
1月 14 16:24:38 bogon gogs[50980]: 2020/01/14 16:24:38 [TRACE] Log path: /server/gogs/log
1月 14 16:24:38 bogon gogs[50980]: 2020/01/14 16:24:38 [TRACE] Log Mode: File (Info)
1月 14 16:24:38 bogon gogs[50980]: 2020/01/14 16:24:38 [ INFO] Gogs 0.11.91.0811
```
---
- **启动成功！**
- 打开浏览器访问：`服务器IP:3000/`，出现gogs默认首页！
- 问题解决，记录一下，希望找到使用git用户自启动gogs服务的办法
- 本文章最后编辑时间:2020-01-14-17:29

