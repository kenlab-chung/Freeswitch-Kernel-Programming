# FreeSWITCH 内核编程
## 1 字符串操作函数

## 2 正则表达式函数

## 3 内存函数
## 4 线程函数 
- switch_thread_create

## 5 缓冲区读写函数
### 5.1 缓冲区类型
- 固定类型(默认)：固定长度，在内存池中申请内存，完成申请内存后不可改变内存大小。
- 动态类型(SWITCH_BUFFER_FLAG_DYNAMIC)：使用malloc函数在堆上申请，可以使用realloc函数增加能内。
- 分区类型(SWITCH_BUFFER_FLAG_PARTITION)：指向现有内存区域，不会自动申请内存。
### 5.2 缓冲区操作函数

