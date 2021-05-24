---
layout: post
title: "Coding.net上的Jenkins简单使用总结"
tags: jenkins, ci, coding.net
---

# coding.net上的jenkins CI 简单使用


## Jenkins
公司把所有的项目代码仓库都挪到了coding.net上。公司买的套餐有Jenkins功能，放着就浪费了，正好原来的本地GitLab服务器有点问题，就试着用一下coding的CI，编一个前后端分离的项目。

前端webpack，后端gradle，都比较简单，编译过程就是命令行执行两个命令（是的，就是 `npm run build` 和 `./gradlew clean build`）。
## git仓库
因为编译流程需要用到两个仓库，好像目前（2021.5）coding还不能够用图形界面配出来，所以还是需要写jenkinsfile。如果编译只有一个代码仓库的项目，甚至可以纯鼠标操作。

### oauth
coding自动生成了一些访问令牌，直接拼到url里就可以了：
```groovy
script {
    def backend_url = "https://${env.PROJECT_TOKEN_GK}:${env.PROJECT_TOKEN}@e.coding.net/seer-group/seed/SeerLaRDS.git"
    env.BACKEND_URL = backend_url
    def frontend_url = "https://${env.PROJECT_TOKEN_GK}:${env.PROJECT_TOKEN}@e.coding.net/seer-group/seed/SeerLaRDS-UI.git"
    env.FRONTEND_URL = frontend_url
}
```
在构建快照里可以找到 `PROJECT_TOKEN_GK` 和 `PROJEC_TOKEN` ，用这个组合就可以访问项目中的所有代码仓库，很方便。
![img](../img/post20210524-tokens.png)
一开始我没找到这个办法，直接生成了一个访问令牌放在CI流程的启动参数里，也能编译，但不安全。

### GitSCM
jenkins的[git插件](https://plugins.jenkins.io/git/)，可以在图形界面编辑，也可以在coding内嵌的monaco文本编辑器中编辑：
```groovy
checkout([$class: 'GitSCM',
          branches: [[name: GIT_BUILD_REF]],
          userRemoteConfigs: [[url: env.BACKEND_URL,credentialsId: env.PROJECT_TOKEN]]])
```
主要就三个参数：分支、url、用户凭证。分支默认为`master`，url是`env.BACKEND_URL`，就是上面拼出来的url，凭证是自动生成的项目凭证`env.PROJEC_TOKEN`。简单的流程配起来也比较简单。

## 制品
coding有制品库，上传到制品库的命令叫`codingArtifactsGeneric`，有两个必填参数：文件列表、制品库名称，还有好几个可选参数，这次只用了版本号：
```groovy
codingArtifactsGeneric(files: env.FILENAME, repoName: 'test', version: env.SEED_VERSION)
```
这条命令指定了文件名为 `env.FILENAME`，用环境变量是因为加了版本号和时间；制品库名称为`test`，版本号为`env.SEED_VERSION`，是从gradle文件里取出来的。

## 总结
coding对jenkins的支持还是可以的。简单的流程配起来很方便，而且还有个检查功能：如果写完jenkinsfile不能切换到图形化编辑器，那就是语法不对，比如少个括号什么的。

附完整的jenkinsfile，以后老了记不起来怎么写了可以回来看看：
```groovy
pipeline {
  agent any
  stages {
    stage('检出后端') {
      steps {
        script {
          def backend_url = "https://${env.PROJECT_TOKEN_GK}:${env.PROJECT_TOKEN}@e.coding.net/seer-group/seed/SeerLaRDS.git"
          env.BACKEND_URL = backend_url
          def frontend_url = "https://${env.PROJECT_TOKEN_GK}:${env.PROJECT_TOKEN}@e.coding.net/seer-group/seed/SeerLaRDS-UI.git"
          env.FRONTEND_URL = frontend_url
        }

        sh 'mkdir -p backend'
        dir('backend') {
          checkout([$class: 'GitSCM',
          branches: [[name: GIT_BUILD_REF]],
          userRemoteConfigs: [[url: env.BACKEND_URL,credentialsId: env.PROJECT_TOKEN]]])
        }

      }
    }
    stage('编译后端') {
      steps {
        dir('backend') {
          sh './gradlew clean'
          sh './gradlew :app:assembleDist'
        }

      }
    }
    stage('检出前端') {
      steps {
        sh 'mkdir -p frontend'
        dir('frontend') {
          checkout([$class: 'GitSCM',
          branches: [[name: GIT_BUILD_REF]],
          userRemoteConfigs: [[url: env.FRONTEND_URL,credentialsId: env.PROJECT_TOKEN]]])
        }

      }
    }
    stage('编译前端') {
      steps {
        dir('frontend') {
          sh 'npm install'
          sh 'cd app && npm run release && zip ui.zip -r build'
        }

      }
    }
    stage('打包上传') {
      steps {
        script {
          def zip_name = sh(returnStdout: true, script:'''ls backend/app/build/distributions -Art | grep zip | tail -n 1''')
          env.SEED_ZIP_NAME=zip_name
          println "zip_name: ${zip_name}"
          // TODO shell parameter expansion in groovy script
          // def fullname = sh(returnStdout: true, script: ''' echo ${SEED_ZIP_NAME%.zip} ''').trim()
          // println "fullname: ${fullname} "
          def folder_name = zip_name.minus('.zip').trim()
          println "folder name: |${folder_name}|"
          env.SEED_FOLDER_NAME = folder_name
          def seed_version=sh(returnStdout: true, script: ''' echo ${SEED_ZIP_NAME} | cut -d - -f 2 | cut -d . -f 1-3 ''').trim()
          println "seed version: |${seed_version}|"
          def build_date = sh(returnStdout: true, script:''' date +%m%d-%H%M''').trim()
          def snapshot_name = "${folder_name}-test-${build_date}"
          env.SEED_VERSION=seed_version
          env.FOLDER="seed-with-ui"
          env.FILENAME="${snapshot_name}.zip"
          env.SNAPSHOT_NAME=snapshot_name
        }

        sh 'echo $SEED_ZIP_NAME'
        sh 'mkdir -p $FOLDER'
        sh 'unzip backend/app/build/distributions/$SEED_ZIP_NAME -d $FOLDER'
        sh 'unzip frontend/app/ui.zip -d $FOLDER/$SEED_FOLDER_NAME'
        sh 'cd $FOLDER/$SEED_FOLDER_NAME && mv build ui'
        sh 'cd $FOLDER && mv $SEED_FOLDER_NAME $SNAPSHOT_NAME'
        sh 'cd $FOLDER && zip $FILENAME -r $SNAPSHOT_NAME'
        sh 'mv $FOLDER/$FILENAME ./'
        sh 'rm -r $FOLDER'
        sh 'echo $SEED_VERSION'
        codingArtifactsGeneric(files: env.FILENAME, repoName: 'test', version: env.SEED_VERSION)
      }
    }
  }
}
```