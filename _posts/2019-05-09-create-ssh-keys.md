---
layout: post
title:  "生成git ssh key"
date:   2019-05-09 20:27:29
categories: git
tags: ssh git
author: ddmcc
---

* content
{:toc}




## 检查是否已经存在ssh key

```java
$ ls -al ~/.ssh
```


通常名称为以下这几种：
- id_dsa.pub
- id_ecdsa.pub
- id_ed25519.pub
- id_rsa.pub

如不存在则生成新的key，已存在key则直接复制到github账号上


## 生成新的ssh key

```java
root@ddmcc:~# ssh-keygen -t rsa -b 4096 -C "308119975@qq.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:o8juUP0rMeU0hq1S9+EVegAgwblVtRJezVB0xNSGiSs 308119975@qq.com
The key's randomart image is:
+---[RSA 4096]----+
|   .oo.o+o+*o=++ |
|    o... o..=.+ o|
|     o oo .o o . |
|    ..o B.E +    |
|    ...BS+ =     |
|   o..+o..o      |
|  . o..o.        |
|   o  .  .       |
|   .o  ..        |
+----[SHA256]-----+
root@ddmcc:~# ls
total 32
drwx------  4 root root 4096 May  9 20:36 ./
drwxr-xr-x 23 root root 4096 Apr  8 21:28 ../
-rw-------  1 root root    0 Apr 14 15:43 .bash_history
-rw-r--r--  1 root root 3106 Oct 23  2015 .bashrc
drwx------  2 root root 4096 Apr 14 15:42 .cache/
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
drwx------  2 root root 4096 May  9 20:37 .ssh/
-rw-------  1 root root  679 Apr 18 23:15 .viminfo
-rw-------  1 root root   51 May  9 20:17 .Xauthority
root@ddmcc:~# cd .ssh
root@ddmcc:~/.ssh# ls
id_rsa  id_rsa.pub
root@ddmcc:~/.ssh# cat id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDA6EeAUcQXeV/G+prk9yTgS8VD1C9XY7GgOH7/X6AXaNc6VfFUg5oKrA2bwJZ36WyJSEwuYfrRkBDrAOWsVG+fZkY11Ia2ReEDQISkLdaklDX9SRNJZUUpjPeBs3VLOB+IFO4f+RWB4M9ojsHC2HNty5XpKBEnU8u5I4nmK7qLrEeT2RoypCLQEQ86fM9ofJi49/qmt4jOka+x8+PKFItx7SdpnhQDG18oEov8qdDOgoijLTe066ELDS/ZTc658P42Op4FN+6rJptYYrNPHcn7RhgFdXCVT9sUS7Wtgmsa65yiyrRtLiw0H7aGV8NLis4AWHfLwXEHF3GXequmraH6p620e/uUB92dS9dvwkPJyWOFf7YHJpxNrB5GBy6b4iPv4UiKFDNfeBdahIejtCIMqm8u6UX0EmQxR4orgPCY6oCfozwdFF2q+ZLs/sIEgT8F1/GrU5C1w+gxuNs+oIVQqJ/wj3QhM+w33Hsha+EZuIFMHcYZTOo2vAi43gNREMX6SxOh8ZLx5QVD4vaW66mVNhxYtTXH3fr/YXFa8/Or/l666vyB7V3j3cqjLoWvueYpbLhqumvqi/OIqDAJST94M/dz1Lip8G97AFxbG41Z7bPZIPq52PvzLS5gOiafUmTTRg4eHseOn5lgLoQSxKmgpgSzDz1kqk+kP4M2shjzWQ== 308119975@qq.com
root@ddmcc:~/.ssh# 
root@ddmcc:~# 
```

## 复制key到github上

### 在个人账号-Settings

![](https://help.github.com/assets/images/help/settings/userbar-account-settings.png)


### SSH and GPG keys

![](https://help.github.com/assets/images/help/settings/settings-sidebar-ssh-keys.png)


### New SSH key

![](https://help.github.com/assets/images/help/settings/ssh-add-ssh-key.png)


### Paste your key into the "Key" field

![](https://help.github.com/assets/images/help/settings/ssh-key-paste.png)


### Add SSH key

![](https://help.github.com/assets/images/help/settings/ssh-add-key.png)
