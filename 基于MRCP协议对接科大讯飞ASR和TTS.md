# 基于MRCP协议对接科大讯飞ASR和TTS
## 1 需求
使用科大讯飞原子能力来提供ASR和TTS技术实现智能语音机器人。本文采用MRCP协议对接科大讯飞ASR和TTS功能。而FreeSWITCH刚好实现了支持MRCP协议的模块mod_unimrcp。
## 2 MRCP协议交换流程
![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/492ee7bd-f363-4384-9c67-36cd43cae4f3)
mrcp一般由客户端发起invite请求与服务器协商语音编码和mrcp通道信息，然后基于mcrp协议的asr信息和tts信息交互，最后进行rtp语音流交换并返回识别信息直到挂机为止。
## 3 FreeSWITCH配置
FreeSWITCH使用mod_unimrcp模块来处理与mrcp服务器的交互，配置文件在`conf/autoload_config/unimrcp.xml`中。主要配置如下：
```
<configuration name="unimrcp.conf" description="UniMRCP Client">
  <settings>
    <!-- UniMRCP profile to use for TTS -->
    <param name="default-tts-profile" value="kdxf-mrcp2"/>
    <!-- UniMRCP profile to use for ASR -->
    <param name="default-asr-profile" value="kdxf-mrcp2"/>
    <!-- UniMRCP logging level to appear in freeswitch.log.  Options are:
         EMERGENCY|ALERT|CRITICAL|ERROR|WARNING|NOTICE|INFO|DEBUG -->
    <param name="log-level" value="DEBUG"/>
    <!-- Enable events for profile creation, open, and close -->
    <param name="enable-profile-events" value="true"/>
 
    <param name="max-connection-count" value="100"/>
    <param name="offer-new-connection" value="1"/>
    <param name="request-timeout" value="3000"/>
  </settings>
 
  <profiles>
    <X-PRE-PROCESS cmd="include" data="../mrcp_profiles/*.xml"/>
  </profiles> 
</configuration>
```

