---
title: Github Action Java CI with Maven&Docker(Aliyun)
date: 2022-12-19 9:31
categories: 框架搭建
---

# 前言
重点：`Actions secrets`隐藏账号、密码等敏感信息。  
为了简化发布流程，决定研究下`Github Action`，之前也一直用过`hexo action`只是简单的使用。  

# 一、 注册Docker镜像仓库
为了国内方便起见，选择<a href="https://cr.console.aliyun.com/cn-hangzhou/instances">阿里云docker镜像仓库</a>。
根据页面提示，注册**命名空间**和**镜像仓库**。

# 二、 配置Actions secrets
```text
# aliyun docker仓库
DOCKER_REPOSITORY
# aliyun docker密码
DOCKER_PASSWORD
# aliyun docke用户名
DOCKER_USERNAME
# 服务器IP
HOST
# 服务器密码
HOST_PASSWORD
# ssh 端口号
HOST_PORT
# 服务器用户名
HOST_USERNAME
# 环境变量，我这里是nacos地址
NACOS_SERVER
```

# 三、 创建Action
`仓库` -> `Actions` -> `New workflow` -> 选择`Publish Java Package with Maven`

```yaml
# This workflow will build a Java project with Maven, and cache/restore any dependencies to improve the workflow execution time
# For more information see: https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-java-with-maven

# This workflow uses actions that are not certified by GitHub.
# They are provided by a third-party and are governed by
# separate terms of service, privacy policy, and support
# documentation.

name: Java CI with Maven/Docker

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: Set up JDK 11
      uses: actions/setup-java@v3
      with:
        java-version: '11'
        distribution: 'temurin'
        cache: maven
    - name: Build with Maven
      run: mvn -B package --file pom.xml

    # Optional: Uploads the full dependency graph to GitHub to improve the quality of Dependabot alerts this repository can receive
    - name: Update dependency graph
      uses: advanced-security/maven-dependency-submission-action@571e99aab1055c2e71a1e2309b9691de18d6b7d6

    - name: Login to Docker Aliyun
      uses: docker/login-action@v2
      with:
        registry: registry.cn-hangzhou.aliyuncs.com
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push
      uses: docker/build-push-action@v3
      with:
        context: .
        file: ./Dockerfile
        push: true
        tags: ${{ secrets.DOCKER_REPOSITORY }}:latest

  pull-docker:
      needs: [build]
      name: Pull Docker
      runs-on: ubuntu-latest
      steps:
        - name: Deploy
          uses: appleboy/ssh-action@master
          with:
            host: ${{ secrets.HOST }}
            username: ${{ secrets.HOST_USERNAME }}
            password: ${{ secrets.HOST_PASSWORD }}
            port: ${{ secrets.HOST_PORT }}
            script: |
              docker stop $(docker ps --filter ancestor=${{ secrets.DOCKER_REPOSITORY }} -q)
              docker rm -f $(docker ps -a --filter ancestor=${{ secrets.DOCKER_REPOSITORY }}:latest -q)
              docker rmi -f $(docker images ${{ secrets.DOCKER_REPOSITORY }}:latest -q)
              docker login --username=${{ secrets.DOCKER_USERNAME }} --password ${{ secrets.DOCKER_PASSWORD }} ${{ secrets.DOCKER_REPOSITORY }}
              docker pull ${{ secrets.DOCKER_REPOSITORY }}:latest
              docker run -d --name mlb-gateway -p 8000:8080 -e NACOS_SERVER=${{secrets.NACOS_SERVER}} ${{ secrets.DOCKER_REPOSITORY }}:latest
```
# 四、总结
<a href="https://docs.docker.com/build/ci/">Docker Githun文档</a>，一定要先看文档！！！！！！！