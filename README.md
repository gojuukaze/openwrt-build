# 利用 Docker 编译 OpenWrt

由于网络原因，编译openwrt时经常下载失败，因此想到通过docker镜像，结合阿里云、Github action等远程构建docker镜像，将下载步骤放在远程进行。

目前还有一个方法就是利用Github action直接远程编译，但这样做有3个问题：

1. 编译需要config文件，生成这个文件又需要下载一些库，这就造成了循环=。=。而通过docker你拉下来的镜像中已经下好了所有库，不需要下载。
2. 如果你要对config进行一些小的改动，通过Github action就需要从头开始编译。而利用docker镜像可以直接进入docker容器修改文件后编译，不需从头开始。
3. Github action有一定学习成本，而dockerfile和shell差不多，就算没接触过过也能快速上手

# 开始之前

本教程所有代码在： 

* https://github.com/gojuukaze/openwrt-build   
* https://gitee.com/gojuukaze/openwrt-build 

关于Docker的一些概念，以及如何使用阿里云、Github action等来构建docker镜像不在本教程范围，你可以自行学习。

# 编译步骤


首选说明下，我是分成了4个镜像来编译的，`base` , `v1` , `v2` , `v3` ，你完全可以只构建一个镜像。  
这样的好处是有些镜像可以重复利用，编译有问题时可以使用上一步的镜像来再次构建，避免从头开始。

每个镜像说明：

| tag  | Dockerfile   | 说明                                                 |
| ---- | ------------ | ------------------------------------------------------ |
| base | Dockerfile   | 基础镜像，包含编译所需的包。             |
| v1   | Dockerfile_1 | 编译准备1，clone代码，并执行基础的update与install |
| v2   | Dockerfile_2 | 编译准备2，修改源码，增加功能，再次执行update与install应用修改|
| v3   | Dockerfile_3 | 编译，执行download并编译                       |


## 1. 构建 base

使用 `Dockerfile` 构建 `base` 镜像，此镜像是基于 `immortalwrt/opde:base` 在此基础上通过immortalwrt的脚本更新了一些包。  

理论上来说，这个镜像能用于编译其他openwrt系统，不过建议在dockerfile的最后再安装一次你系统所需的包

## 2. 构建 v1

使用 `Dockerfile_1` 构建 `v1` 镜像，这个镜像主要clone代码，执行update与install。

如果你不用immortalwrt系统，修改代码的地址。

## 2. 构建 v2

使用 `Dockerfile_2` 构建 `v2` 镜像，这个镜像主要进行`make`前的最终准备修改（修改源码，添加功能，再次执行update与install）。

这里有的代码我是通过 git patch 来修改的，你也可以通过sed或其他方式来修改，或者最后一步编译前拉取镜像本地修改。  

如果你要使用我的patch，请注意：我的patch是基于`Orange Pi - R1 Plus LTS`以及`immortalwrt 21.02`修改的，这些patch不一定适用于你，请甄别后使用。

每个patch说明如下：

* add_pwm_fan.sh.patch : 风扇脚本。我的板子用这个脚本风扇会滋滋响，如果你有同样问题，取消这个试试。
* resizing_filesystem_at_first_boot.patch : 首次启动时扩展分区。注意这个依赖bash、f2fsck等命令，需要在config中添加。这个我测试有问题，因此我采用的是开机后手动执行一次 `resize_rootfs.sh` (这个脚本在仓库里)
* wan_ACCEPT.patch : 允许通过wan的ip口登录。immortalwrt默认只能通过lan口登录。
* swap_wan_and_lan.patch : wan lan口互换
* 1.5g.patch : cpu超频至1.5g

> 这些path是从[R1-Plus-LTS](https://github.com/mingxiaoyu/R1-Plus-LTS)复制的，有问题请向他反馈

另外，`Dockerfile_2` 里还单独install了几个包，这时因为我在执行`./scripts/feeds install -a` 会出现这个warning `287#9 2.046 WARNING: Makefile 'package/utils/busybox/Makefile' has a dependency on 'libpam', which does not exist`  
如果你的没报错可以删除这几个install

## 3. 创建config

```
docker run xxx/openwrt-build:v2 zsh

make menuconfig
```

创建完后通过`docker cp`把`.config`复制到本地并上传仓库

## 4. 构建 v3，编译系统

使用 `Dockerfile_3` 构建 `v3` 镜像，这个镜像就是执行最终的编译了。  

不过因为远程编译可能会因时间太久而被判断为超时，且我本地性能更好。
因此`dockerfile`里只是执行了`make download`，构建镜像后拉取到本地编译的，你可以根据需要自行选择。  

如果你选择本地编译，需要注意：这一步仍然可能需要下载一些包，这取决于你的config文件。不过一般这些下载一般不会太慢，如果你这一步卡了，可以ctrl-c后再编译，一般都能下载成功。  

另外，建议你修改下go以及npm的源，你config中有的选项可能依赖它们。

```
export GOSUMDB=off
export GO111MODULE=on
export GOPROXY=https://goproxy.cn

npm config set registry http://registry.npmmirror.com

```

> 执行`make V=s`时有的教程建议首次编译时添加`-j 1`，这是为了更好的发现编译错误  
> 但如果你满足下面条件，首次完全可以多线程编译的。  
> 1. 对源码改动不是更大。  
> 2. 你的硬件之前有人编译成功或受官方支持。  
> 3. config中没有选择太多或偏门选项。  
> 多线程编译时，若这一步需要下载一些文件，下载进度条可能无法正确显示，你过你看到一直卡主不动，或是有一个数字一直闪烁且没有变化那多半是下载卡主了，直接终止后重新编译。

## 5. 从docker中取出镜像

最终的系统镜像是在 `bin/targets/xx` 下，然后通过`docker cp`取出来。（若非本地编译你需要先运行镜像）

