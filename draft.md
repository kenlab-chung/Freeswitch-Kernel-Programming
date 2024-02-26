# draft
## 1 启动流程
main启动时，调用核心函数switch_core_init_and_modload(),在switch_core_init_and_modload()内部分别调用switch_core_init()和switch_loadable_module_init()函数初始化内核和加载外围模块。
## 2 核心数据结构
### 2.1 Session与Channel
当有电话呼入FreeSWITCH或者FreeSWITCH发起一路电话，建产生一个Session，同时生产一个Channel。Session与Channel一一对应，前者更多关注控制信令层，后者更关注媒体层。一个switch_core_session结构体实例代表一个session。switch_core_session结构体中包含Channel结构体指针。

对于在FreeSWITCH库或者FreeSWITCH内部模块，Seesion内部的东西是私有的，想要获取session中Channel不能直接调用session‑>channel而只能调用switch_core_session_get_channel(session)来获取Channel信息。

### 2.2 状态机
状态机处理函数为switch_core_session_run，而整个状态机工作在Channel上面的。switch_core_session_run内部为一个循环，只要Session所对应的Channel状态不是CS_DESTROY，则一直循环。状态机所有状态定义如下：
```
typedef enum {
	CS_NEW,             //新建
	CS_INIT,            //已初始化
	CS_ROUTING,         //路由
	CS_SOFT_EXECUTE,    //准备好执行，可由第三方控制
	CS_EXECUTE,         //执行Dialplan中的App
	CS_EXCHANGE_MEDIA,  //与另一个Channel交换媒体
	CS_PARK,            //等待进一步的命令指示
	CS_CONSUME_MEDIA,   //消费掉媒体并丢弃
	CS_HIBERNATE,       //sleep
	CS_RESET,           //重置
	CS_HANGUP,          //挂机，结束信令和媒体交换
	CS_REPORTING,       //收集呼叫信息，如写CDR等
	CS_DESTROY,         //待销毁，退出状态机
	CS_NONE             //无效
} switch_channel_state_t;
```
调用switch_channel_set_state()函数修改状态机的状态。
当状态发生变化事，内核通过switch_channel_set_running_state()函数来改变running_state，并执行相关的回调来通知相应终端通知其状态已经发送改变：
```
endpoint_interface->io_routines->state_run
```

## 4 跨平台设计
### 4.1 APR库重构
FreeSWITCH使用Apahce APR库调用计算机资源操作接口。APR主要目的包括：

- 为不同操作系统平台提供文件系统访问、网络编程、进程管理、线程管理、共享内存等一致性功能接口。
- 封装字符串处理、队列、socket、内存、内存池、哈希、互斥锁等各种资源的管理和抽象。

为解决FreeSWITCH内部命名空间冲突问题，APR改名为fspr_，并在switch_apr.c文件中对APR库进行二次封装，冠以“switch_”命名空间，这样所有的内核函数都有了一致的命名空间。

### 4.2 APR 函数返回值
在APR库中规定：函数返回一个状态值，为调用者提供一个fspr_status_t返回值。这个值与FreeSWITCH中switch_status_t对应，因为我们看到在fspr_前缀函数在switch_apr.c文件中switch_前缀函数中返回时，直接转换为switch_status_t类型。

要注意switch_前缀函数返回状态值SWITCH_STATUS_SUCCESS与APR_SUCCESS对应，而且他们的枚举值都为0。所以在判断函数返回值是否成功时应该这样判断：
```
switch_status_t status =switch_dir_make();
if(status==SWITCH_STATUS_SUCCESS)
{
  // Success 
}
else
{
  //Failure
}
```
另外某些函数返回一个字符串(char* 或者const char*)或者函数返回类型为void*或者void时，函数发生错误时返回一个空指针或者默认函数执行成功。

### 4.2 APR 内存池

APR使用内存池来管理内存。建议用于小内存分配。在APR中人为从内存池中申请的内存永远不会分配失败。如果内存分配失败，那么系统是不可恢复的。

## 5 媒体操作

## 6 模块开发

FreeSWITCH中模块类型声明：

- SWITCH_STANDARD_CHAT_APP
- SWITCH_STANDARD_APP
- SWITCH_STANDARD_DIALPLAN
- SWITCH_STANDARD_API
- SWITCH_STANDARD_JSON_API
  
FreeSWITCH中模块函数声明：
- SWITCH_MODULE_DEFINITION：模块定义函数，告知系统核心加载模块时调用load函数进行初始化，启动新线程执行runtime，卸载时调用shutdown函数清理资源。
- SWITCH_MODULE_LOAD_FUNCTION：模块加载函数，负责系统启动时或运行时加载模块，可以进行配置读取及资源初始化。
- SWITCH_MODULE_RUNTIME_FUNCTION：模块运行时函数，可以启动线程处理请求，监听socket等。
- SWITCH_MODULE_SHUTDOWN_FUNCTION：模块卸载函数，负载模块卸载及相关资源回收。

上传模块声明定义在switch_types.h文件中。

### 6.1 app开发
### 6.2 api开发
### 6.3 lua 开发
## 7 调试
## 8 云部署
### 8.1 docker 常用操作命令
- 构建docker镜像
  ```
  sudo docker build -f Dockerfile.switch-v1.0.3 -t bsoft-switch:v1.0.3 .
  ```
- 导出镜像
  ```
  sudo docker save -o docker-bsoft-fs-x64-v1.0.3.tar bsoft-switch:v1.0.3
  ```
- 导入镜像
  ```
  sudo docker load -i ./docker-bsoft-fs-x64-v1.0.3.tar
  ```
### 8.2 docker网络
#### 8.2.1 网桥工具安装
  
```
wget https://www.kernel.org/pub/linux/utils/net/bridge-utils/bridge-utils-1.7.1.tar.xz
tar -xvf bridge-utils-1.7.1.tar.xz
cd bridge-utils-1.7.1
autoconf
./configure
make
make install
```
#### 8.2.1 默认网络
  
安装docker时，会自动创建三个网络：bridge（创建容器默认连接到此网络）、host、none。
```
$ docker network ls
NETWORK ID          NAME                DRIVER
c531a28c2221        bridge              bridge
6f1e18f3b5dd        none                null
5f6d09e593d7        host                host
```
##### 8.2.1.1 Host模式
与宿主机同在一个网络中，没有独立IP地址。

Docker使用Linux的Namespaces技术来进行资源隔离，如PID Namespaces隔离进程，Mount Namespaces隔离文件系统，Network Namespaces隔离网络等。

一个Network Namespaces 提供了一份独立的网络环境，包括网卡、路由、Iptable规则等都与其他的Network Namespaces隔离。

一个docker容器一般会分配一个独立的Network Namespaces。

如果启动容器的时候使用host模式，那么这个容器将不会获得独立的Network Namespaces，而是和宿主机共用一个Network Namespaces。也就是说，容器不会虚拟出自己的网卡，配置自己的IP等，而是使用宿主机的IP和端口号。

例如：我们在10.10.0.178/24的主机上用host模式启动一个nginx容器，监听tcp 80端口。
```
#运行容器
$ docker run --name=nginx-host --net=host -p 80:80 -d nginx
c531a28c222145a79870577839b30a4e63a4a0e928cd0325eafaad46eb590162

#查看容器
$ docker ps

CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
c531a28c2221        nginx               "nginx -g 'daemon ..."   25 seconds ago      Up 25 seconds                           nginx-host
```
当我们在容器中执行ifconfig命令查看网络环境时，看到的都是宿主机上的信息。而外界访问容器中的应用，则直接使用10.10.0.178:80即可，不用任何NAT转换，如同容器中的应用直接运行在宿主机上。但是容器的其他方面，如文件系统、进程列表还是和宿主机隔离的。
```
$ netstat -nplt | grep nginx
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      27340/nginx: master
```
##### 8.2.1.2 Container模式
这个模式指定新创建的容器和已经存在的一个容器共享一个Network Namespaces，而不是和宿主机共享。新建的容器不会创建自己的网卡，也不会配置自己的IP，而是和一个指定的容器共享IP、端口范围。同样，两个容器处理网络方面，其它的如文件系统、进程列表还是隔离的。两个容器可以通过IO网卡设备进行通讯。
##### 8.2.1.3 None模式
该模式将容易放置在它自己的网络栈中，但是并不进行任何配置。实际上，该模式关闭了容器的网络功能，在以下情况是有用的：

容器并不需要网络（例如，值需要写磁盘卷的批处理任务）

overlay

在docker1.7代码进行了重构，单独把网络部分独立出来编写，所以在docker1.8新加入的一个overlay网络模式。dokcer对于网络访问的控制也是在逐渐完善的。
##### 8.2.1.4 Bridge模式
相当于VMware中的NAT模式，容器使用独立的Network Namespaces，并连接到docker0虚拟网卡（默认模式）。通过docker0网卡以及iptables nat表于宿主机通信。

bridge模式是docker默认的网络设置，此模式会为每个容器分配Network Namespaces、设置IP等，并行主机上的docker容器连接到一个虚拟网桥上。

#### 8.2.1 Bridge模式详解

