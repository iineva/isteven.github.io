---
title: Android中运行使用termux运行frp
date: 2018-04-13 13:28:30
tags: [android, termux, frp]
---

# 用termux编译的方法如下

* 如果不想自己编译，可以直接下载我编译好的二进制文件即可，后面的一大堆东西可以不用看了
[frps.zip](https://github.com/fatedier/frp/files/1893727/frps.zip)
[frpc.zip](https://github.com/fatedier/frp/files/1893729/frpc.zip)


* 安装 [com.termux.apk.zip](https://github.com/fatedier/frp/files/1893666/com.termux.apk.zip)， [com.termux.boot.apk.zip（可选，这个是用来执行自启动脚本的）](https://github.com/fatedier/frp/files/1893668/com.termux.boot.apk.zip)，然后打开`termux` APP

* 安装并启动`openssh`，由于手机端不方便输入命令，使用电脑ssh到手机，方便操作(可选)

1. 在电脑端执行
```shell
cd ~/.ssh
# 如果~/.ssh 目录下没有id_rsa.pub文件，执行命令 ssh-keygen 即可生成
php -S 0.0.0.0:5000
```

2. termux 端执行
```shell
pkg install openssh curl
sshd
# http://xxxxxx/id_rsa.pub为你的电脑公钥
curl http://YOUR_IP_ADDRESS:5000/id_rsa.pub >> $HOME/.ssh/authorized_keys
```
3. 在电脑端执行，以下命令即可远程连接到手机
```shell
ssh root@YOUR_IP_ADDRESS -p 8022
```

* 安装`golang`

```shell
pkg install golang
```
* 获取`frp`源码

```shell
go get github.com/fatedier/frp
```

* 编译`frps`和`frpc`

```shell
cd $HOME/go/src/github.com/fatedier/frp/cmd/frpc
go build
cd $HOME/go/src/github.com/fatedier/frp/cmd/frps
go build
```
