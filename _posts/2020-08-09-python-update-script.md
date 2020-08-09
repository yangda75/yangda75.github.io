---
layout: post
title: "使用python脚本自动更新服务器程序"
tags: linux, python, sysadmin, git, crontab
---
项目需要将程序部署到一台本地的服务器上进行一些简单的测试。这个任务当然是给本菜鸡来做:-]。

首先搞来一台服务器，装上CentOS8，配好git公钥，装好各种依赖，拉代码到服务器，进行一次手动部署。

手动部署后端很简单：在执行`gradle assemble`之后，复制文件到指定路径，后端程序就部署好了 ;-)。

手动部署前端也类似：执行`yarn`安装依赖，然后用`npm run build` 执行一下`package.json`里定义好的步骤就行了。

## python脚本
虽说手动部署一次只要不到十条命令，但是每当程序更新就要重新执行一次，加上目前处于功能开发阶段，每天都有很多代码提交上去，手动更新花费大量时间，需要用脚本进行自动化部署、更新。

之前写类似的自动化脚本都是用bash,powershell，没用过python，但是听说Python可以做，而且这次的自动化任务很简单，所以就打算用Python尝试一下。

最后写出来的脚本是这样的：
```python
import os

def changeDirectory(directory):
    os.chdir(directory)
    print("pwd: ")
    os.system('pwd')

# update backend
changeDirectory('/home/test/seer-la')
os.system('git pull')
os.system('bash ./gradlew publishToMavenLocal')

changeDirectory('/home/test/seer-la/general')
os.system('bash ../gradlew assemble')
os.system('cp build/distributions/general.tar /home/test/')

changeDirectory('/home/test')
os.system('cp general/bin/boot.json ./')
os.system('rm -r general')
os.system('tar -xf general.tar')
os.system('rm general.tar')

changeDirectory('/home/test/seer-la-ui')
os.system('git pull')
os.system('yarn')

changeDirectory('/home/test/seer-la-ui/general')
os.system('rm -r build')
os.system('yarn release')

changeDirectory('/home/test/seer-la-node')
os.system('git pull')
os.system('yarn')
os.system('npm run build')

changeDirectory('/home/test/general/bin')
os.system('mkdir ui')
os.system('cp /home/test/boot.json ./')
os.system('cp -r /home/test/seer-la-ui/general/build/* ./ui/')
os.system('pkill java')
os.system('./general & disown')

```
`os.system`确实多了点，还不如直接用bash写，用python纯属脱裤子放屁;-\。可能是因为这个自动化任务太简单了，不需要计算和复杂的逻辑，不适合用python写吧。

## crontab
用定时任务来执行自动更新的脚本。

```bash
crontab -e
```
打开编辑器，写一个每小时一次的定时任务
```crontab
*/60 * * * * python3 update.py > update.log
```
定义定时任务的时候，如果不知道该怎么写才能满足想要的触发频率，可以参考[这个网站](https://crontab.guru/#*/60_*_*_*_*)
