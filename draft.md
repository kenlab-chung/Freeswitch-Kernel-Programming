# draft
## 1 启动流程
## 2 状态机
## 3 事件
## 4 跨平台设计
FreeSWITCH内部采用Apahce APR库实现程序的跨平台运行机制，为了防止FreeSWITCH内部命名空间冲突问题，开发者在switch_apr.c文件中对APR库进行二次封装，并将APR改名为fspr
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
