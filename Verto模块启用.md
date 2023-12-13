# Verto模块启用
## 1 环境
- debian 11
- gcc 10.2.1
- openssl 1.1.1n 
- freeswitch 1.10.10
- 测试终端 windows 10 64 位 (浏览器：Microsoft edge 115.0.1901.183 64 位 ，Chrome 112.0.5615.138 64位)
## 2 Verto配置
修复Verto配置文件：conf/autoload_configs/verto.conf.xml
```
<configuration name="verto.conf" description="HTML5 Verto Endpoint">
  
  <settings>
    <param name="debug" value="10"/>
    <!-- seconds to wait before hanging up a disconnected channel -->
    <param name="detach-timeout-sec" value="120"/>
    <!-- enable broadcasting all FreeSWITCH events in Verto -->
    <!-- <param name="enable-fs-events" value="false"/> -->
    <!-- enable broadcasting FreeSWITCH presence events in Verto -->
    <param name="enable-presence" value="false"/>
  </settings>
  <profiles>
    <profile name="default-v4">
      <param name="bind-local" value="$${local_ip_v4}:8081"/>
      <param name="bind-local" value="$${local_ip_v4}:8082" secure="true"/>
      <param name="force-register-domain" value="$${domain}"/>
      <param name="secure-combined" value="/usr/local/freeswitch/certs/wss.pem"/> -证书的位置，一会儿存放证书时要用
      <param name="secure-chain" value="/usr/local/freeswitch/certs/wss.pem"/>
      <param name="userauth" value="true"/>
      <!-- setting this to true will allow anyone to register even with no account so use with care -->
      <param name="blind-reg" value="false"/>
      <param name="mcast-ip" value="224.1.1.1"/>
      <param name="mcast-port" value="1337"/>
      <param name="rtp-ip" value="$${local_ip_v4}"/>
      <!--  <param name="ext-rtp-ip" value=""/> -->
      <param name="local-network" value="localnet.auto"/>
      <param name="outbound-codec-string" value="opus,vp8,h264"/>
      <param name="inbound-codec-string" value="opus,vp8,h264"/>
      <param name="apply-candidate-acl" value="localnet.auto"/>
      <param name="apply-candidate-acl" value="wan_v4.auto"/>
      <param name="apply-candidate-acl" value="rfc1918.auto"/>
      <param name="apply-candidate-acl" value="any_v4.auto"/>
      <param name="timer-name" value="soft"/>
    </profile>
    <profile name="default-v6">
      <param name="bind-local" value="[$${local_ip_v6}]:8081"/>
      <param name="bind-local" value="[$${local_ip_v6}]:8082" secure="true"/>
      <param name="force-register-domain" value="$${domain}"/>
      <param name="secure-combined" value="$${certs_dir}/wss.pem"/>
      <param name="secure-chain" value="$${certs_dir}/wss.pem"/>
      <param name="userauth" value="true"/>
      <!-- setting this to true will allow anyone to register even with no account so use with care -->
      <param name="blind-reg" value="false"/>
      <param name="rtp-ip" value="$${local_ip_v6}"/>
      <!--  <param name="ext-rtp-ip" value=""/> -->
      <param name="outbound-codec-string" value="opus,vp8"/>
      <param name="inbound-codec-string" value="opus,vp8"/>
      <param name="apply-candidate-acl" value="wan_v6.auto"/>
      <param name="apply-candidate-acl" value="rfc1918.auto"/>
      <param name="apply-candidate-acl" value="any_v6.auto"/>
      <param name="apply-candidate-acl" value="wan_v4.auto"/>
      <param name="apply-candidate-acl" value="any_v4.auto"/>
      <param name="timer-name" value="soft"/>
    </profile>
  </profiles>
</configuration>
```
为用户配置verto支持，修改directory/default.xml，在<params>和</params>中添加如下：
```
<param name="jsonrpc-allowed-methods" value="verto"/>
<param name="jsonrpc-allowed-event-channels" value="demo,conference,presence"/>
```
为每个用户的xml配置文件添加verto相关内容（以1000.xml为例）：
```
<user id="1001">
    <params>
      <param name="password" value="$${default_password}"/>
      <param name="vm-password" value="1001"/>
      <param name="verto-context" value="public"/>
      <param name="verto-dialplan" value="XML"/>
</params>
```
如果使用会议功能，则需修改配置文件conf/autoload_configs/conference.conf.xml

在<profiles>和</profile>中检查conference-flags项目，确保其中包含livearray-sync和livearray-json-status拨号计划配置例如：
```
<extension name="HTML5 Verto">
   <condition field="destination_number" expression="^(10[0-9][0-9])$">
      <action application="export" data="dialed_extension=$1"/>
      <action application="set" data="call_timeout=30"/>
      <action application="bridge" data="${verto_contact ${dialed_extension}@${dialed_domain}}"/>
   </condition>
</extension>
```
至此freeswitch针对verto的配置已经完成，想要测试的话，需要使用freeswitch自带的verto demo，安装使用过程如下文：
## 3 创建证书
因为wss方式的访问是加密的，所以需要配置https方式运行demo，先创建一个自签名证书，以供freeswitch和web服务使用，注意二者需要使用同一套证书才能顺利的访问freeswitch。创建自签名证书，过程中按提示输入各种信息，过程中需要openssl的支持，如果未安装请提前自行安装。
```
wget http://files.freeswitch.org/downloads/ssl.ca-0.1.tar.gz
tar zxfv ssl.ca-0.1.tar.gz
cd ssl.ca-0.1/
perl -i -pe 's/md5/sha256/g' *.sh
perl -i -pe 's/1024/4096/g' *.sh
./new-root-ca.sh
./new-server-cert.sh self.verto
./sign-server-cert.sh self.verto
cat self.verto.crt self.verto.key > /usr/local/freeswitch/certs/wss.pem /*注意此路径和verto配置文件中的相同*/
```
## 4 安装web服务（apache）
```
sudo apt-get install apache2
sudo a2enmod ssl
sudo a2enmod rewrite
```
修改/etc/apache2/sites-enabled/000-default.conf
```
<VirtualHost *:443> -- 修改为443
…………
--增加下面几行
SSLEngine On
SSLOptions +StrictRequire
SSLCertificateFile /usr/local/freeswitch/certs/wss.pem
SSLCertificateKeyFile /usr/local/freeswitch/certs/wss.pem
SSLCertificateChainFile /usr/local/freeswitch/certs/wss.pem
</VirtualHost>
```
重启apache
```
sudo service apache2 restart
```
## 5 测试
将verto demo放到apache的web页面目录中
```
cp -rf /opt/freeswitch-1.8.7/html5/verto/video_demo  /var/www/html
```
此外verto还提供了不带视频的demo和verto_communicator。

打开浏览器（我用的Chrome）访问
```
https://IP/video_demo
```
提示未信任时，选高级，点击继续。

## 6 nginx服务配置
如下图所示：
其中wss.key为私钥
![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/f3d95042-4445-46cb-8155-61378f8af136)

