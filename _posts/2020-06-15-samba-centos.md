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

## 添加用户
先添加linux用户

## 配置共享文件夹

## SELinux 和 firewalld
开启samba的端口：
```bash
sudo firewall-cmd --add-service=samba --permanent
sudo firewall-cmd --reload
```
