---
layout: post
title: "Jenkins用docker编译"
mathjax: true
tags: jenkins, jdk, docker, pipeline
---

前一阵在coding上搞了jenkins自动编译，结果最近总是编不过，今天在coding的文档上找到了解决方法。

编不过是因为coding的编译服务器只有jdk8，而我们的项目用了 jdk8 没有，jdk11才有的api，在编kotlin代码的stage里用jdk11的docker就行了:

```groovy
    stage('编译后端') {
      agent {
        docker {
          image 'adoptopenjdk:11-jdk-hotspot'
          args '-v /root/.gradle/:/root/.gradle/ -v /root/.m2/:/root/.m2/'
          reuseNode true
        }
      }
      steps {
        dir('backend') {
          sh './gradlew clean'
          sh './gradlew :app:assembleDist'
        }
      }
    }
```