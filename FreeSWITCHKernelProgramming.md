# FreeSWITCH内核编程
---
## 1 环境
FreeSWITCH : FreeSWITCH-1.10.11-dev
OS: debian bullseye
gcc:10.2.1
gdb:10.1.99
## 2 FreeSWITCH启动流程
### 2.1 main函数的主要工作
FreeSWITCH在main函数中除了初始化异常处理程序，解析软交换启动参数（比如：-nc -nonat 参数和 -conf -mod -log -run  -db -scripts -htdocs -base  -storage -temp -cache -recordings -grammar -certs -sounds等工作目录）之外，其核心就是调用switch_core_init_and_modload()函数初始化FreeSWITCH内核以及加载外围模块。
switch_core_init_and_modload()函数调用完毕，也就是FreeSWITCH内核和外围模块启动完毕后，main函数会调用switch_core_runtime_loop()函数进入无限循环。对于后台启动的实例，它基本上什么都没做。对于从前台启动的系统，switch_core_runtime_loop()会调用switch_console_loop()函数以启动一个控制台接收用户的键盘输入并打印系统运行信息，比如：命令行输出和日志等。
当实例退出switch_core_runtime_loop()函数后，调用switch_core_destroy()函数清理系统资源。
### 2.2 switch_console_loop函数主要工作
switch_console_loop()函数分两种情况启动，一种是使用跨平台库libedit库接收用户按键并在控制台上打印信息，一种是直接启动while循环接收用户按键以及在控制台上打印信息。不通的是使用libedit库时，函数内部启动了console_thread()线程，在这里检测用户命令合法性，并将命令存入历史记录，以备将来再执行该命令时可以使用键盘上的箭头按键翻看历史命令。同样，使用非libedit库的switch_console_loop()函数中也实现了相同的功能。
二者最终都是调用switch_console_process()函数，在switch_console_process函数内部执行switch_console_execute()函数执行命令行解析，并最终调用switch_api_execute()函数，执行命令输入，并将执行结果存放到istream中，最后会被取出并打印到控制台上。
### 2.3 switch_api_execute函数主要工作
通过switch_loadable_module_get_api_interface()查找外面模块中实现的api接口。并执行api->function回调函数。外围模块加载参考后续章节。
### 2.4 switch_core_init_and_modload函数主要工作
- 调用switch_core_init()函数初始FreeSWITCH内核。
- 调用switch_load_network_lists()函数初始化FreeSWITCH工作网络。
- 调用switch_loadable_module_init()函数外围模块。
## 3.4 模块开发实例
### 3.4.1 实现一个api
以模块mod_bsoft为例。实现一个hello_bsoft api接口。
在src/mod/endpoints目录下创建mod_bsoft目录。
把模块名加入到modules.conf中，make时根据此文件选择编译哪些模块，并生成相应的makefile文件。
![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/61d53f62-da03-4f52-8876-1272e49e8e31)
在configure.ac中加入mod_bsoft的Makefile配置：
![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/d2dea185-25d0-4cf6-aa48-6d1e5ec80f90)
在mod_bsoft目录中创建mod_bsoft.c文件。代码如下：
```
/*************************************************************************
  > File Name: mod_bsoft.c
  > Author: zhongqf
  > Mail: zhongqf.cn@gmail.com
  > Created Time: 2023-11-22 15:42:03
 ************************************************************************/

#include<switch.h>
SWITCH_MODULE_LOAD_FUNCTION(mod_bsoft_load);
SWITCH_MODULE_RUNTIME_FUNCTION(mod_bsoft_runtime);
SWITCH_MODULE_SHUTDOWN_FUNCTION(mod_bsoft_shutdown);
SWITCH_MODULE_DEFINITION(mod_bsoft,mod_bsoft_load,mod_bsoft_shutdown,mod_bsoft_runtime);

SWITCH_STANDARD_API(hello_bsoft)
{
    switch_log_printf(SWITCH_CHANNEL_LOG,SWITCH_LOG_NOTICE,"hello bsoft!\n");
    return SWITCH_STATUS_SUCCESS;
}

SWITCH_MODULE_LOAD_FUNCTION(mod_bsoft_load)
{
    switch_api_interface_t *api_interface;
    *module_interface = switch_loadable_module_create_module_interface(pool,modname);
    SWITCH_ADD_API(api_interface,"hello_bsoft","hello bosft API",hello_bsoft,"syntax");
    return SWITCH_STATUS_SUCCESS;
}
SWITCH_MODULE_RUNTIME_FUNCTION(mod_bsoft_runtime)
{
    int i=0;
    while(i<=10)
    {
        switch_log_printf(SWITCH_CHANNEL_LOG,SWITCH_LOG_NOTICE,"hello from bsoft!\n");
        switch_yield(100000);
        i++;
    }
    return SWITCH_STATUS_TERM;
}
SWITCH_MODULE_SHUTDOWN_FUNCTION(mod_bsoft_shutdown)
{
    switch_log_printf(SWITCH_CHANNEL_LOG,SWITCH_LOG_NOTICE,"mod_bsoft shutdown!\n");
    return SWITCH_STATUS_SUCCESS;

}
```
编写makefile.am文件： 
```
include $(top_srcdir)/build/modmake.rulesam
MODNAME=mod_bsoft

mod_LTLIBRARIES = mod_bsoft.la
mod_bsoft_la_SOURCES  = mod_bsoft.c
mod_bsoft_la_CFLAGS   = $(AM_CFLAGS)
mod_bsoft_la_LIBADD   = $(switch_builddir)/libfreeswitch.la
mod_bsoft_la_LDFLAGS  = -avoid-version -module -no-undefined -shared
```
执行freeswitch源码目录下执行
```
./bootstrap.sh && ./configure --prefix=/opt/freeswitch_install/
```
可以看到mod_bsoft目录下产生了一个Makefile文件。此时可以执行在此目录下编译单独编译模块，并部署到安装目录下。
```
make && make install
```
也可以在freeswitch元目录下执行：
```
make mod_bsoft && make mod_bsoft-install
```
在freeswitch安装目录下可以看到mod_bsoft模块相关文件：
![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/1c274c3e-0ef7-4ffe-a745-cacf4dc2d0ee)
在控制台中加载mod_bsoft模块，可以看到模块可以正常加载，并打印runtime中的日志。
![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/935a35f7-a763-4388-823d-a777ec76b36e)
可以调用模块中定义的hello_bsoft接口
![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/630e2f43-e5e7-4851-b449-8f49585a71a0)
可以正常卸载：
![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/bdd76918-d75d-4a84-8aaf-2e806b710555)
### 3.4.2 实现一个Dialplan


