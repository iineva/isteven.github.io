---
title: 通过Docker中运行rsync备份服务器
date: 2018-04-27 11:04:30
tags: [docker, rsync]
---

# 背景

服务器备份是必做的日常工作之一，而备份工作这是一个立体的过程，并没有什么完美的方案，需要根据实际情况确定备份方案。而笔者此次给大家介绍的方案是通用型的，比较适合小型服务器备份，如个人服务器。

# 适用场景

* 服务器数据量不大，数据更新频率不高
* 希望将自己的个人云服务器定期备份到本地电脑，如MacBook中（笔者就是这么用的）
* 公司内部使用的一些小型服务做备份工作

# 思路

`rsync`是很高效的文件同步工具，可以对大量文件进行远程差量同步，`rsync`的特性，差量同步，断点续传，让文件同步过程变得非常高效。并且他通过使用`ssh`作为传输通道，使得你不管在什么网络中进行备份传输都可以很安全地进行。

通常我们使用`rsync`的方法是，通过`cron`设置定时任务，唤起`rsync`进行备份。但是这种做法在实际使用过程中会有一些问题：

* 需要对指定文件进行编辑配置，不方便copy方案
* 需要额外安装`rsync`和`cron`，对于用惯`docker`的人来说，“装软件”简直太麻烦！
* 备份时间间隔过短时，如果上次备份还未完成，就会同时启动两个备份进程，之前因为此原因，导致备份服务器卡死，不过这个问题笔者也有解决方案，下次再聊

能否用一切都在docker中运行呢，一个`docker-compose.yml`，一行`docker-compose up`，就能`Just work`

# 方案

下面内容保存成`docker-compose.yml`，进入此文件目录运行`docker-compos up`

```yaml
version: "3"

services:
  rsync:
    image: instrumentisto/rsync-ssh
    restart: always
    container_name: rsync
    volumes:
      - ${HOME}/.ssh:/root/.ssh
      - ${PWD}/backup:/backup
    entrypoint:
      - /bin/sh
      - -c
      - |
        # 输出包含当前时间的日志
        function log() {
          echo -e "`date -d @$$((\`date +%s\`+3600*8)) '+%Y-%m-%d %H:%M:%S'` $$@"
        }

        # 备份服务器，用法：
        # backup <备份名> <服务器地址> [备份目录默认/docker/.]
        function backup() {
          log "============================"
          log "开始备份: $$1"
          rsync -az --timeout=3600 -P --partial --delete -e ssh root@$$2:$${3:-/docker/.} /backup/$$2.backup
          log "结束备份: $$1"
        }

        # 开始备份任务
        while true
        do
        	
          # 根据实际情况类推，添加服务器
          backup "Gitlab服务器" "192.168.2.2" /root/.
          backup "ineva.cn服务器" "ineva.cn"

          # 休眠5分钟
          log "休眠5分钟"
          sleep 300
        done
```

# Docker化遇到的问题

* 极轻量化的镜像，最好是基于`alpine`

最终找到了完美match的镜像 [instrumentisto/rsync-ssh](https://hub.docker.com/r/instrumentisto/rsync-ssh/) , 只有`4M`大小，有所有我们需要的功能

* `rsync`使用的是`ssh`进行连接，需要指定私钥

这个问题一个技巧解决，先配置`备份载体服务器A`可以连接到`需要备份的主机B`，然后将A的`.ssh`目录映射到`container`中即可，这样做的好处是你只需要确定A能连接到B中即可，容器内也会得到相同的权限。

* 如何在不添加额外文件的情况下运行多个脚本，并且分行书写

这个之前尝试了几个方案，可以执行多个脚本，但是都是必须一行写完，脚本多的话就不方便维护阅读了，最终找到了解决方案：<https://github.com/docker-library/redmine/issues/52>，唯一需要注意的是脚本里面的`$`会被`yaml`语法认为是环境变量名前缀，需要转义，改成`$$`即可

* 日志输出的时间是0时区的，中国在+08:00区，输出时间不便于阅读

为此笔者特地写了篇文章介绍这个问题的解决办法，有兴趣的朋友可以看看：

{% post_link docker-timezone %}

# 脚本优化

备份服务器列表直接在脚本里面修改，有点dirty，如果要进行改进的话，把备份列表放到环境变量里面，脚本通过读取环境变量里面的值确定备份目标，这样脚本就更漂亮了！这一步笔者就不做了，留给大家发挥。