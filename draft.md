# draft
## 1 启动流程
## 2 状态机
## 3 事件
## 4 常用函数
## 5 模块开发

FreeSWITCH中模块类型声明：

- SWITCH_STANDARD_CHAT_APP
- SWITCH_STANDARD_APP
- SWITCH_STANDARD_DIALPLAN
- SWITCH_STANDARD_API
- SWITCH_STANDARD_JSON_API
  
FreeSWITCH中模块函数声明：
- SWITCH_MODULE_LOAD_FUNCTION：模块启动是加载此回调函数。
- SWITCH_MODULE_RUNTIME_FUNCTION：模块启动后常驻线程，回循环调用此回调函数。
- SWITCH_MODULE_SHUTDOWN_FUNCTION：模块卸载时调用此回调函数。

上传模块声明定义在switch_types.h文件中。
### 5.1 app开发
### 5.2 api开发
### 5.3 lua 开发
## 6 调试
## 7 云部署
