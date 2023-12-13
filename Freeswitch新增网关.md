# FreeSWITCH 新增网关
本文介绍认证模式网关配置

配置demo如下，文件存放路径/freeswitch/conf/sip_profiles/external/gw01.xml：
```
<include>
  <gateway name="gw01">
  <param name="username" value="10000"/>
  <param name="realm" value="ims.domain.com"/>
  <param name="from-domain" value="ims.domain.com"/>
  <param name="from-user" value="10000"/>
  <param name="password" value="66771"/>
  <param name="outbound-proxy" value="172.10.16.168:5060"/>
  <param name="register-proxy" value="172.10.16.168:5060"/>
  <param name="expire-seconds" value="60"/>
  <param name="register" value="true"/>
  <param name="ping" value="10"/>
  </gateway>
</include>
```
配置说明如下：
```
<gateways>
  <!-- 网关名称，建议使用代表性的名称 -->
  <gateway name="test"/>
    <!-- realm：对接方的域名或ip加端口的形式，如 81.70.88.88:9060 -->
    <param name="realm" value="www.example.com"/>
    <!-- 表示注册的地址 -->
    <param name="register-proxy" value="192.168.1.8"/>
    <!-- 用户名，用于开启鉴权时进行的注册验证 -->
    <param name="username" value="4444"/>
    <!-- 分机的密码 -->
    <param name="password" value="!@#qwe123"/>
    <!-- 指定在SIP消息中的源用户信息，没有配置则默认和username相同 -->
    <param name="from-user" value="4444"/>
    <!-- 是指定域，它们会影响SIP中的“From”头域。有时第三方会要求我们固定 from头中内容 -->
    <param name="from-domain" value="www.example.com"/>
    <!-- 是否注册，认证模式为true，非认证模式为false -->
    <param name="register" value="true"/>
    <!-- 注册的间隔时间 -->
    <param name="expire-seconds" value="120"/>
    <!-- ping网关地址保持存活，有时需要，主动注册对方时，可能总是掉线刷新网关恢复，可以使用ping保持存活 -->
    <param name="ping" value="10"/>
 </gateway>
<gateways/>
```
修改完网关信息后，可以重启freeswitch自动生效或者在控制台使用如下命令使网关配置生效：
```
sofia profile internal killgw 网关名
sofia profile internal rescan　　
```
注册完成后可以在控制台使用sofia status gateway gw01 查看网关是否注册成功。网关状态分为两种：
- NOREG 没有开启认证模式。
- REGED 开启认证模式。

配置完后，可以在default.xml配置外呼路由：
```
<extension name="call_out">
    <condition field="destination_number" expression="^(\d{10,13})$">
        <action application="set" data="RECORD_TITLE=Recording ${destination_number} ${caller_id_number} ${strftime(%Y-%m-%d %H:%M)}"/>
        <action application="set" data="RECORD_COPYRIGHT=(c) 2011"/>
        <action application="set" data="RECORD_SOFTWARE=FreeSWITCH"/>
        <action application="set" data="RECORD_ARTIST=FreeSWITCH"/>
        <action application="set" data="RECORD_COMMENT=FreeSWITCH"/>
        <action application="set" data="RECORD_DATE=${strftime(%Y-%m-%d %H:%M)}"/>
        <action application="set" data="RECORD_STEREO=true"/>
        <action application="set" data="media_bug_answer_req=true"/>
        <action application="record_session" data="$${base_dir}/recordings/archive/${strftime(%Y-%m-%d-%H-%M-%S)}_${destination_number}_${caller_id_number}.wav"/>
        <!-- 以上内容都是录音的路由写死即可无需修改 -->
        <!-- 超时时间 60s 对方不接听时持续振铃60S -->
        <action application="set" data="call_timeout=60"/>
        <!-- 规定坐席保持时播放的录音，可以单独使用其他录音 -->
        <action application="set" data="temp_hold_music=local_stream://alternate_moh"/>
        <!-- 设置外呼时的主叫名称和号码为 xxx-->
        <action application="set" data="effective_caller_id_name=xxx" />
        <action application="set" data="effective_caller_id_number=xxx" />
        <!-- 使用刚所配置路由的地方 -->
        <action application= "bridge" data="sofia/gateway/gw01（网关名）/${destination_number}" />
        <action application="set" data="test=${hangup_cause}"/>
        <action application="execute_extension" data="hangup_reason-${hangup_cause} XML features"/>
    </condition>
</extension>
```
