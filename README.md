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
### 7.1 结构体
- switch_frame_t：在SIP协议中即为一个RTP包。成员变量data指向读到的数据，packet指向RTP消息的开始（比 data 多一个RTP包头，一般是12个字节），packetlen为packet 的长度，seq为序号，ssrc为同步源标志，datalen媒体实际数据长度。
- switch_rtp_t：唯一标识一个RTP连接。

### 7.2 媒体操作函数
- switch_core_session_read_frame：从session中读取一帧媒体数据，同时负责检测DTMF,收到DTMF的情况下会调用相关回调函数。这里的一帧，在SIP协议中即为一个RTP包。当读不到数据时，函数返回一个禁音包(CNG，即Comfort Noise Generation)。函数最终是调用Endpoints提供的read_fame()回调函数读取数据。
- switch_core_session_write_frame：将音频数据写入session，发送至远端。在函数中调用perform_write()函数实现数据发送，而perform_write()函数最终调用Endpoint中的write_frame()回调函数实现数据发送功能。
- switch_core_media_read_frame：从底层的RTP中读取一帧媒体数据，其中type参数取值为：SWITCH_MEDIA_TYPE_AUDIO（音频数据）和SWITCH_MEDIA_TYPE_VIDEO（视频数据）。
- switch_core_media_write_frame：调用底层switch_rtp_write_frame()函数完成RTP数据发送功能。
- switch_core_media_negotiate_sdp：媒体协商。

 ### 7.3 RTP操作函数
  - switch_rtp_zerocopy_read_frame：读取一帧媒体数据。参数rtp_session唯一标志一个RTP连接，参数frame用于存放数据。
  - switch_rtp_write_frame：调用switch_socket_sendto()函数直接发送RTP数据。或者调用rtp_common_write()函数发送RTP数据。
  - switch_core_media_activate_rtp：启动RTP收发函数。  
  - read_rtp_packet：调用 switch_socket_recvfrom() 函数从真正的Socket中读取数据。
  - switch_socket_recvfrom：对APR库中apr_socket_recvfrom()的一个封装，根据不同平台调用不同函数读取Socket中的媒体数据。
  - switch_socket_sendto：发送RTP包。
  - switch_rtp_new：建立一个新的rtp_session。rx_host和rx_port为本端IP地址和端口号，tx_host和tx_port分别为远端IP地址和端口号。
  - switch_rtp_destroy：释放一个rtp_sesion。
  
## 8 Session操作函数

## 9 Channel操作函数
- switch_channel_ready：检测Channel是否正常，Channel正常建立时返回true，挂机或其他错误情况时返回false。

