---
layout: post
title: "CentOS 8 配置Samba记录"
tags: linux, centos, samba
---

## 安装samba
使用命令行进行配置简单的文件共享服务器只要安装 `samba` 和 `samba-common` 就够了。
```bash
sudo dnf install samba samba-common
```
默认安装的是samba 4
## smb 服务
启动systemd中的smb服务。
```bash
sudo systemctl enable --now smb
```
 `--now` 参数的作用是注册开机启动的同时立即启动服务，这样就不用再输一条 `systemctl start smb` 了。
## 添加用户
先添加linux用户。例如，想要添加samba用户`yda`，密码是`samba`的话，需要先在samba服务器上创建一个
    linux用户，名字叫`yda`，密码是`samba`，并且将`yda`用户添加到一个用户组中，方便samba的权限管理。使用`id` 命令可以查看用户所在的组。

然后将用户添加到samba中。使用`smbpasswd -a`命令来添加用户到samba中。
```bash
$ id yda
uid=1001(yda) gid=1000(test) 组=1000(test),1007(smbgroup)
```

## 配置共享文件夹
需要密码的文件夹：
```
[private]
	inherit permissions = Yes
	path = /samba/share/private
	read only = No
	force group = +smbgroup
	valid users = @smbgroup
```
`valid users = @smbgroup`表示只有`smbgroup`用户组的用户可以使用这个共享文件夹。
`force group = +smbgroup`表示共享文件夹中的文件都会被强制加上`smbgroup`这个所有者。
## SELinux 和 firewalld
开启samba的端口：
```bash
sudo firewall-cmd --add-service=samba --permanent
sudo firewall-cmd --reload
```
selinux:
```bash
sudo chcon -R -t samba_share_t /samba/share
sudo semanage fcontext -a -t samba_share_t "/samba/share(/.*)?"
sudo setsebool samba_enable_home_dirs=1
```
