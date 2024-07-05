# 基于MRCP协议对接科大讯飞ASR和TTS
## 1 需求
使用科大讯飞原子能力来提供ASR和TTS技术实现智能语音机器人。本文采用MRCP协议对接科大讯飞ASR和TTS功能。而FreeSWITCH刚好实现了支持MRCP协议的模块mod_unimrcp。
## 2 MRCP协议交换流程
![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/492ee7bd-f363-4384-9c67-36cd43cae4f3)
mrcp一般由客户端发起invite请求与服务器协商语音编码和mrcp通道信息，然后基于mcrp协议的asr信息和tts信息交互，最后进行rtp语音流交换并返回识别信息直到挂机为止。
## 3 FreeSWITCH 配置
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
其中最主要的是`default-tts-profile`和`default-asr-profile`中的配置，它对应了ASR和TTS的配置文件，这个配置文件一般在`conf/mrcp_profiles/`目录下，参考配置如下：
```
<include>
  <!-- 科大讯飞 Speech Server 5.0 MRCPv2 -->
  <profile name="bjzk-mrcp2" version="2">
    <param name="client-ip" value="172.28.105.14"/>
    <param name="client-port" value="5191"/>
    <param name="server-ip" value="172.28.105.13"/>
    <param name="server-port" value="7010"/>
    <!--param name="force-destination" value="1"/-->
    <param name="sip-transport" value="udp"/>
    <!--param name="ua-name" value="FreeSWITCH"/-->
    <!--param name="sdp-origin" value="FreeSWITCH"/-->
    <!--<param name="rtp-ext-ip" value="172.28.105.15"/>-->
    <param name="rtp-ip" value="172.28.105.14"/>
    <param name="rtp-port-min" value="4000"/>
    <param name="rtp-port-max" value="5000"/>
    <!-- enable/disable rtcp support -->
    <param name="rtcp" value="1"/>
    <!-- rtcp bye policies (rtcp must be enabled first)
             0 - disable rtcp bye
             1 - send rtcp bye at the end of session
             2 - send rtcp bye also at the end of each talkspurt (input)
    -->
    <param name="rtcp-bye" value="2"/>
    <!-- rtcp transmission interval in msec (set 0 to disable) -->
    <param name="rtcp-tx-interval" value="5000"/>
    <!-- period (timeout) to check for new rtcp messages in msec (set 0 to disable) -->
    <param name="rtcp-rx-resolution" value="1000"/>
    <!--param name="playout-delay" value="50"/-->
    <!--param name="max-playout-delay" value="200"/-->
    <!--param name="ptime" value="20"/-->
    <param name="codecs" value="PCMU PCMA L16/96/8000"/>
 
    <!-- Add any default MRCP params for SPEAK requests here -->
    <synthparams>
    </synthparams>
 
    <!-- Add any default MRCP params for RECOGNIZE requests here -->
    <recogparams>
      <!--param name="start-input-timers" value="false"/-->
    </recogparams>
  </profile>
</include>
```
其中，server-ip和server-port就是科大讯飞mrcp服务器的地址和端口。

配置好后，则可以启动FreeSWITCH进行调用。我们可以在Dialplan或者esl中调用App函数play_and_detect_speech来进行放音并识别操作。典型命令如下：
```
<extension name="play_and_detect_speech example">
  <condition field="destination_number" expression="^(1888)$">
    <action application="set" data="tts_engine=unimrcp"/>
    <action application="set" data="tts_voice=donna"/>
    <action application="play_and_detect_speech" data="say:please say yes or no. please say no or yes. please say something! detect:unimrcp {start-input-timers=false,no-input-timeout=5000,recognition-timeout=5000}builtin:grammar/boolean?language=en-US;y=1;n=2"/> 
    <action application="log" data="CRIT ${detect_speech_result}"/>
  </condition>
</extension>
```
最终我们可以通过获取SWITCH_EVENT_DETECTED_SPEECH的事件信息得到ASR的文本。

以上是关于语音原子能力引擎的对接，至于智能机器人的对话系统，科大讯飞提供了DCM系统，它是通过http restful接口来交互的。
