---
layout: post
title:  "ubantu中安装jenkins"
date:   2019-05-15 19:33:42
categories: ubantu
tags: jenkins ubantu
author: ddmcc
---

* content
{:toc}


## 安装


原来把jenkins安装docker容器里，太不方便了，还有权限的问题。


- `wget -q -O - https://pkg.jenkins.io/debian/jenkins-ci.org.key | sudo apt-key add -`
- `sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'`
- `apt-get update`
- `apt-get install jenkins`




```java
root@ddmcc:~# wget -q -O - https://pkg.jenkins.io/debian/jenkins-ci.org.key | sudo apt-key add -
OK
root@ddmcc:~# sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
root@ddmcc:~# apt-get update
Hit:1 https://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial InRelease
Hit:2 http://security.ubuntu.com/ubuntu xenial-security InRelease
Ign:3 http://pkg.jenkins.io/debian-stable binary/ InRelease
Get:4 http://pkg.jenkins.io/debian-stable binary/ Release [2,042 B]
Get:5 http://pkg.jenkins.io/debian-stable binary/ Release.gpg [181 B]
Get:6 http://pkg.jenkins.io/debian-stable binary/ Packages [14.9 kB]
Hit:7 http://us.archive.ubuntu.com/ubuntu xenial InRelease
Get:8 http://us.archive.ubuntu.com/ubuntu xenial-updates InRelease [109 kB]
Get:9 http://us.archive.ubuntu.com/ubuntu xenial-backports InRelease [107 kB]
Get:10 http://us.archive.ubuntu.com/ubuntu xenial-updates/main amd64 Packages [956 kB]
Get:11 http://us.archive.ubuntu.com/ubuntu xenial-updates/main i386 Packages [823 kB]
Get:12 http://us.archive.ubuntu.com/ubuntu xenial-updates/universe amd64 Packages [748 kB]
Get:13 http://us.archive.ubuntu.com/ubuntu xenial-updates/universe i386 Packages [684 kB]
Fetched 3,444 kB in 42s (80.7 kB/s)
Reading package lists... Done
root@ddmcc:~# apt-get install jenkins
```


然后就等待下载完成后自动安装后，打开 `http://ip:8080/`  进入jenkins。密码在`/var/lib/jenkins/secrets/initialAdminPassword` ，登陆后安装插件。


## 遇到的问题


安装后启动，显示启动失败了。



```java
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  daemon
The following NEW packages will be installed:
  daemon jenkins
0 upgraded, 2 newly installed, 0 to remove and 137 not upgraded.
Need to get 76.8 MB of archives.
After this operation, 77.7 MB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://pkg.jenkins.io/debian-stable binary/ jenkins 2.164.3 [76.7 MB]                               
Get:2 http://us.archive.ubuntu.com/ubuntu xenial/universe amd64 daemon amd64 0.6.4-1 [98.2 kB]
Fetched 76.8 MB in 1min 54s (672 kB/s)
Selecting previously unselected package daemon.
(Reading database ... 60101 files and directories currently installed.)
Preparing to unpack .../daemon_0.6.4-1_amd64.deb ...
Unpacking daemon (0.6.4-1) ...
Selecting previously unselected package jenkins.
Preparing to unpack .../jenkins_2.164.3_all.deb ...
Unpacking jenkins (2.164.3) ...
Processing triggers for man-db (2.7.5-1) ...
Processing triggers for systemd (229-4ubuntu21.21) ...
Processing triggers for ureadahead (0.100.0-19) ...
Setting up daemon (0.6.4-1) ...
Setting up jenkins (2.164.3) ...
Job for jenkins.service failed because the control process exited with error code. See "systemctl status jenkins.service" and "journalctl -xe" for details.
invoke-rc.d: initscript jenkins, action "start" failed.
dpkg: error processing package jenkins (--configure):
 subprocess installed post-installation script returned error exit status 1
Processing triggers for systemd (229-4ubuntu21.21) ...
Processing triggers for ureadahead (0.100.0-19) ...
Errors were encountered while processing:
 jenkins
E: Sub-process /usr/bin/dpkg returned an error code (1)
```


发现启动失败了，根据提示输入 `journalctl -xe`



```java
root@ddmcc:~# journalctl -xe
-- Unit acpid.service has finished starting up.
-- 
-- The start-up result is done.
May 15 19:40:23 ddmcc jenkins[26604]: ERROR: No Java executable found in current PATH: /bin:/usr/bin:/sbin:/usr/sbin
May 15 19:40:23 ddmcc jenkins[26604]: If you actually have java installed on the system make sure the executable is in the aforementioned path and that 'type -p java' returns the java executable path
May 15 19:40:23 ddmcc systemd[1]: Starting LSB: Start Jenkins at boot time...
-- Subject: Unit jenkins.service has begun start-up
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
-- 
-- Unit jenkins.service has begun starting up.
May 15 19:40:23 ddmcc systemd[1]: jenkins.service: Control process exited, code=exited status=1
May 15 19:40:23 ddmcc systemd[1]: Failed to start LSB: Start Jenkins at boot time.
-- Subject: Unit jenkins.service has failed
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
-- 
-- Unit jenkins.service has failed.
-- 
-- The result is failed.
May 15 19:40:23 ddmcc systemd[1]: jenkins.service: Unit entered failed state.
May 15 19:40:23 ddmcc systemd[1]: jenkins.service: Failed with result 'exit-code'.
May 15 19:40:23 ddmcc systemd[1]: Reloading.
May 15 19:40:23 ddmcc systemd[1]: Started ACPI event daemon.
-- Subject: Unit acpid.service has finished start-up
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
-- 
-- Unit acpid.service has finished starting up.
-- 
-- The start-up result is done.
May 15 19:44:20 ddmcc dhclient[1075]: DHCPREQUEST of 192.168.171.135 on ens33 to 192.168.171.254 port 67 (xid=0x1fd4248f)
May 15 19:44:20 ddmcc dhclient[1075]: DHCPACK of 192.168.171.135 from 192.168.171.254
May 15 19:44:20 ddmcc dhclient[1075]: bound to 192.168.171.135 -- renewal in 830 seconds.
lines 1962-1995/1995 (END)
```



发现是Java环境的问题，提示没找到,输入 `echo $PATH` 查看环境变量。建立软连接 

`ln -s /opt/Java/jdk1.8.0_152/bin/java /usr/bin/java` ,重启jenkins,然后输入 

`systemctl status jenkins.service` jenkins已正常启动。



```java
root@ddmcc:~# echo $PATH
/opt/Maven/apache-maven-3.5.3/bin:/opt/Java/jdk1.8.0_152/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
root@ddmcc:~# ln -s /opt/Java/jdk1.8.0_152/bin/java /usr/bin/java
root@ddmcc:~# service jenkins restart
root@ddmcc:~# systemctl status jenkins.service
● jenkins.service - LSB: Start Jenkins at boot time
   Loaded: loaded (/etc/init.d/jenkins; bad; vendor preset: enabled)
   Active: active (exited) since Wed 2019-05-15 19:50:09 CST; 10s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 26691 ExecStart=/etc/init.d/jenkins start (code=exited, status=0/SUCCESS)

May 15 19:50:07 ddmcc systemd[1]: Stopped LSB: Start Jenkins at boot time.
May 15 19:50:07 ddmcc systemd[1]: Starting LSB: Start Jenkins at boot time...
May 15 19:50:07 ddmcc jenkins[26691]: Correct java version found
May 15 19:50:07 ddmcc jenkins[26691]:  * Starting Jenkins Automation Server jenkins
May 15 19:50:08 ddmcc su[26726]: Successful su for jenkins by root
May 15 19:50:08 ddmcc su[26726]: + ??? root:jenkins
May 15 19:50:08 ddmcc su[26726]: pam_unix(su:session): session opened for user jenkins by (uid=0)
May 15 19:50:09 ddmcc jenkins[26691]:    ...done.
May 15 19:50:09 ddmcc systemd[1]: Started LSB: Start Jenkins at boot time.
root@ddmcc:~# 
```


