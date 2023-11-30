# FreeSWITCH内核编程
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

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/61d53f62-da03-4f52-8876-1272e49e8e31)
