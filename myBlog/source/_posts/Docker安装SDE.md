---
title: Docker安装SDE
date: 2021-06-08 15:30:23
tags:
---

# Docker 中安装SDE

首先我们需要安装Docker，在这里我们推荐一台可以连接外网的设备，配置好终端代理以及git代理加速后执行脚本完成Docker安装。

```shell
curl -sSL https://get.docker.com/ | sh
```

## 配置容器并设置代理

1.首先docker容器能连上网有bridge模式和host模式，为了能公用宿主主机的代理，我们采用host模式

2.在启动容器前，我们先需要配置docker的全局代理，进入到docker的安装用户跟目录下，我这里是root。

新建./docker文件夹，编辑~/.docker/config.json文件，内容如下：

```
{
"proxies":
{
   "default":
   {
     "httpProxy": "http://127.0.0.1:8889",
     "httpsProxy": "http://127.0.0.1:8889",
     "noProxy": "localhost"
   }
}
}
```

之后使用docker run -it --net host ubuntu /bin/bash指令即可将docker容器配置为host模式，进入后采用apt-get update查看下速率是否正常。

update完成后可以采用curl cip.cc指令查看地址

### 常用的指令

#### 进入容器

docker attach Container id

#### 复制文件到容器内

```csharp
docker cp bf-sde-9.3.x 2685ffacecb7:/root
```

#### 根据打包好的镜像启动容器

```
docker run --net host -v /usr/local/:/usr/local -v /lib/modules/:/lib/modules/ -v /usr/src:/usr/src -v /usr/include:/usr/include   --privileged -it --name=foxing sde_v1 /bin/bash
```

#### 更新镜像源

```
 # 备份原镜像软件源
 mv /etc/apt/sources.list /etc/apt/sources.list.bak
 # 更改镜像为阿里镜像源
 echo "deb http://mirrors.aliyun.com/ubuntu/ trusty main multiverse restricted universe" >> /etc/apt/sources.list &&
 echo "deb http://mirrors.aliyun.com/ubuntu/ trusty-backports main multiverse restricted universe" >> /etc/apt/sources.list &&
 echo "deb http://mirrors.aliyun.com/ubuntu/ trusty-proposed main multiverse restricted universe" >> /etc/apt/sources.list &&
 echo "deb http://mirrors.aliyun.com/ubuntu/ trusty-security main multiverse restricted universe" >> /etc/apt/sources.list &&
 echo "deb http://mirrors.aliyun.com/ubuntu/ trusty-updates main multiverse restricted universe" >> /etc/apt/sources.list &&
 echo "deb-src http://mirrors.aliyun.com/ubuntu/ trusty main multiverse restricted universe" >> /etc/apt/sources.list &&
 echo "deb-src http://mirrors.aliyun.com/ubuntu/ trusty-backports main multiverse restricted universe" >> /etc/apt/sources.list &&
 echo "deb-src http://mirrors.aliyun.com/ubuntu/ trusty-proposed main multiverse restricted universe" >> /etc/apt/sources.list &&
 echo "deb-src http://mirrors.aliyun.com/ubuntu/ trusty-security main multiverse restricted universe" >> /etc/apt/sources.list &&
 echo "deb-src http://mirrors.aliyun.com/ubuntu/ trusty-updates main multiverse restricted universe" >> /etc/apt/sources.list
```

#### 编译指令

```
$SDE/p4studio_build/p4studio_build.py --use-profile $SDE/p4studio_build/profiles/all_profile.yaml
```

#### 软链接的镜像

```
build -> /usr/src/linux-headers-5.4.0-73-generic/
ln -s /usr/src/linux-headers-5.4.0-73-generic ./build
```

#### 编译过程中遇到的问题

1.由于在18.04之后失去了对python2的支持，所以我们需要手动安装python2。但是我们不建议更换源，因为这样有概率导致python-pip安装失效。

2.在进行bf-drivers的编译过程中会出错。原因是缺少一个软链接导致的。

首先我们在启动容器的时候对/lib/modules建立了一个映射，使用uname -a查看我们具体的内核版本,进入对应的文件夹，本次为：/lib/modules/5.4.0-73-generic

使用ll命令查看其下的build文件夹，这个文件夹链接到了/usr/src/linux-headers-5.4.0-73-generic/，而/usr/src我们在启动的时候并未创立链接，所以我们需要将/usr/src目录下与我们当前内核有关的文件夹拷贝进入docker中，主要为以下两个文件：

![image-20210609153354719](C:\Users\86155\Desktop\myBlog\source\_posts\Docker安装SDE\image-20210609153354719.png)

3.由于系统在启动的时候需要挂载大页内存，所以我们需要在启动容器的时候加上--privileged参数。

#### docker启动多终端

```
docker exec -it foxing /bin/bash
```

使用docker attach的多个终端之间会互相同步，采取这个指令可以多执行一个bash终端

#### docker打包容器

```
docker commit -a "pengxin" -m "base" 48ea337c0e68 sde_v1
```

#### docker 生成镜像

导出镜像：依据容器ID导出tar文件

docker export f299f501774c > hangger_server.tar

导入镜像：

docker import - new_hangger_server < hangger_server.tar

#### docker 编译BSP指令

```
cd $BSP/bf-platforms
./autogen.sh
./configure --prefix=$BSP_INSTALL --enable-thrift
make
make install
```

