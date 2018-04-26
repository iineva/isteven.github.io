---
title: Docker下修改时区的办法，特别是在alpine中
date: 2018-04-27 01:07:49
tags: [docker, alpine]
---

# 应用场景
脚本输出日志时，中国在+08:00区，默认情况下日志显示的时间少了8小时，为了解决这问题，找到如下解决方案

## 直接使用系统的时区设置


```yaml
volumes:
  - /etc/localtime:/etc/localtime
```

## 安装时区文件（alpine中）

在Dockerfile或者容器中执行下面两条命令

```shell
apk add tzdata
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime 
```

## 基于比较完整的系统镜像如Ubuntu，CentOS等

只需要设置环境变量即可

```shell
ENV TZ=Asia/Shanghai
```

## 最简单的办法，不需要修改系统环境

前面的办法都需要对镜像进行定制，下面使用更简单的办法，直接利用`date`命令修正时区后输出

```shell
# 3600*8为+08:00区，其他时区以此类推
date -d @$((`date +%s`+3600*8))
# Fri Apr 27 01:20:23 UTC 2018
```

格式化输出格式

```shell
date -d @$((`date +%s`+3600*8)) '+%Y-%m-%d %H:%M:%S'
# 2018-04-27 01:20:34
```

封装成函数

```shell
function log() {
  echo -e "`date -d @$((\`date +%s\`+3600*8)) '+%Y-%m-%d %H:%M:%S'` $@"
}
log "hello word"
log "this is right timezone now!"
# 2018-04-27 01:21:36 hello world
# 2018-04-27 01:23:48 this is right timezone now!
```

