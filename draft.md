# draft
## 1 启动流程

## 2 核心数据结构
### 2.1 Session与Channel
### 2.2 状态机

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

## 5 常用函数

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
