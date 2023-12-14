# FreeSWITCH 内核编程
## 1 公共函数
- switch_test_flag

## 2 字符串操作函数

## 3 正则表达式函数

## 4 内存函数
## 5 线程函数 
- switch_thread_create

## 6 缓冲区读写函数
### 6.1 缓冲区类型
- 固定类型(默认)：固定长度，在内存池中申请内存，完成申请内存后不可改变内存大小。
- 动态类型(SWITCH_BUFFER_FLAG_DYNAMIC)：使用malloc函数在堆上申请，可以使用realloc函数增加能内。
- 分区类型(SWITCH_BUFFER_FLAG_PARTITION)：指向现有内存区域，不会自动申请内存。
### 6.2 缓冲区操作函数
## 7 媒体操作函数
- switch_core_session_write_frame：将音频数据写入session，发送至远端。在SIP协议中，一般为20毫秒音频数据。当读不到数据时，函数返回一个禁音包(CNG，即Comfort Noise Generation)。函数最终是调用Endpoints提供的read_fame回调函数读取数据。
- switch_core_session_read_frame：从session中读取音频数据

