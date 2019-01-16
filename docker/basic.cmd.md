---
title: "Docker 常用命令"
date: 2019-01-15T23:28:27+08:00
categories:
    - Docker
tags: 
    - Docker
    - tips
---

下面是一些比较常用的 Docker 命令
## 镜像类基本命令
安装镜像
```docker
docker pull ${IMAGE}
```

显示已经安装镜像的详细内容
```docker
docker images --no-trunc
```

构建镜像
```docker
docker build --rm=true .
```

删除指定镜像
```docker
docker rmi ${IMAGE_ID}
```

删除未使用的镜像
```docker
docker rmi $(docker images --quiet --filter &quot;dangling=true&quot;) 
```

删除所有没有标签的镜像
```docker
docker rmi $(docker images | grep “^” | awk “{print $3}”)
```

删除所有的镜像
```docker
docker rmi $(docker images)
```

构建自己的镜像
```docker
docker build -t <镜像名> <Dockerfile 路径>
```

导出镜像
```docker
docker save docker.io/tomcat:7.0.77-jre7 >/root/mytomcat7.tar.gz
```

导入镜像
```docker
docker load < /root/mytomcat7.tar.gz
```

为自定义的镜像打上 tag
```docker
docker tag <image> <username>/<repository>:<tag>
```

将自定义的镜像发布到仓库
```docker
docker push <username>/<repository>:<tag>
```

查看某个镜像详情
```docker
docker inspect <镜像名称>
```

## 容器类基本命令
查看容器
```docker
docker ps    # 正在运行的容器
docker ps -a # 所有的容器
```

停止、启动、杀死一个容器
```docker
docker stop <容器名 or ID>
docker start <容器名 or ID>
docker kill <容器名 or ID>
```

交互方式启动
```docker
docker run -it ubuntu:latest /bin/bash
```

守护进程方式启动容器
```docker
docker run -d ubuntu:latest
```

守护进程方式启动容器并且进行端口映射
```docker
docker run -d -p 9999:8080 docker.io/tomcat:latest
```

查看容器的root用户密码
```docker
docker logs <容器名orID> 2>&1 | grep '^User: ' | tail -n1
```

查看容器日志
```docker
docker logs -f <容器名orID>
```

进入容器
```docker
docker attach ${CID}
```

已有容器中运行命令/启动交互进程
```docker
docker exec -it ${CID} <执行命令>
```

使用正则表达式删除容器
```docker
docker ps -a | grep wildfly | awk '{print $1}' | xargs docker rm -f
```

删除所有退出的容器
```docker
docker rm -f $(docker ps -a | grep Exit | awk '{ print $1 }')
```

删除所有的容器
```docker
docker rm $(docker ps -aq)
```

显示指定容器的IP
```docker
docker inspect --format '{{ .NetworkSettings.IPAddress }}' ${CID}
```

通过正则表达式查找容器的镜像ID
```
docker ps | grep wildfly | awk '{print $1}'
```

## 用户和组
登录
```docker
docker login
```