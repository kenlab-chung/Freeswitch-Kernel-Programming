# 基于Opensips+Rtpengine+Freeswitch实现语音通信系统
## 1 需求
支持SIP软/硬终端，支持webrtc终端，对接运营商IMS服务，媒体服务器双机负载均衡。考虑采用Opensips+Rtpengine+Freeswitch的架构来实现，其实如果项目无需支持webrtc的话，Rtpengine也是可以省去。

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/127ac448-5479-463c-b9b3-f79aceb71b91)

本文重点介绍的是Opensips+Rtpengine的相关安装和配置。为了简化部署，这次将Opensips和Rtpengine安装在同一台服务器上，操作系统为Centos7.6

## 2 源码安装配置opensips
- 从官网下载opensips，版本为2.4.11。

- 解压后执行make menuconfig（注意提前安装好mysql数据库）

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/8215c2c9-207e-4e6b-9881-e19729220a4c)

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/b5a90b3d-1943-468e-9cb4-40055a0fbb7b)

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/7fe2704a-2036-4d28-b3a3-ed73c5779747)

- 配置数据库并建表

  编辑 vim usr/local/etc/opensips/opensipsctlrc

  ![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/94106b19-0f00-4319-8f70-ca88e728ebd0)

执行脚本如下：
```
[root@os opensips]# cd /usr/local/sbin/
[root@os sbin]# ls
opensips  opensipsctl  opensipsdbctl  opensipsunix  osipsconfig  osipsconsole
[root@os sbin]#  ./opensipsdbctl  create
```
如果提示成功的话，安装就大功告成了！
## 3 源码安装rtpengine
```
#!/bin/bash
# Install the packages required to compile RTP engine
set -e
yum install iptables-devel kernel-devel kernel-headers xmlrpc-c-devel
yum install "kernel-devel-uname-r == $(uname -r)"
yum install glib glib-devel gcc zlib zlib-devel openssl openssl-devel pcre pcre-devel libcurl libcurl-devel xmlrpc-c xmlrpc-c-devel
yum install libevent-devel glib2-devel json-glib-devel gperf libpcap-devel git hiredis hiredis-devel redis perl-IPC-Cmd
# MariaDB ver 10+
yum install MariaDB-devel MariaDB-client MariaDB-shared
# Spandsp
yum install spandsp-devel spandsp
# epel
yum install epel-release
# libwebsockets
yum install libwebsockets libwebsockets-devel
# ffmpeg
rpm --import http://li.nux.ro/download/nux/RPM-GPG-KEY-nux.ro
rpm -Fvh http://li.nux.ro/download/nux/dextop/el7/x86_64/nux-dextop-release-0-1.el7.nux.noarch.rpm
# check for Nux desktop repo by yum repolist
yum -y install ffmpeg ffmpeg-devel
 
# Get The Latest Release Of RTPEngine Source From RTPEngine’s GitHub Repository
cd /usr/local/src
if [ -d rtpengine ]; then
    cd rtpengine
    make clean
    git pull
else
    git clone https://github.com/sipwise/rtpengine.git
fi
 
# Compile and install the daemon
cd /usr/local/src/rtpengine/daemon/
make
cp rtpengine /usr/sbin/rtpengine
 
# Compile and install iptables extension
cd /usr/local/src/rtpengine/iptables-extension
make all
cp libxt_RTPENGINE.so /usr/lib64/xtables/.
 
# Compile and install the kernel module xt_RTPENGINE
cd /usr/local/src/rtpengine/kernel-module
make
 
# determine kernel release
uname -a
kernel_ver=`uname -r`
cp xt_RTPENGINE.ko /lib/modules/${kernel_ver}/extra/xt_RTPENGINE.ko
depmod -a
 
# load module at boot time
# Add the following lines to /etc/modules-load.d/rtpengine.conf (add the following lines)
CONF_FILE=/etc/modules-load.d/rtpengine.conf
echo '# load xt_RTPENGINE module' > ${CONF_FILE}
echo 'xt_RTPENGINE' >> ${CONF_FILE}
 
# load the module and check
modprobe xt_RTPENGINE
lsmod | grep xt_RTPENGINE
 
# check files
ls -l /proc/rtpengine/control
ls -l /proc/rtpengine/list
 
TableID=0
# add forwarding table with an ID=$TableID. $TableID is a number between 0-63
echo 'add 0' > /proc/rtpengine/control
 
# Adding iptables rules to forward the incoming packets to xt_RTPENGINE module
iptables -I INPUT -p udp -j RTPENGINE --id $TableID
ip6tables -I INPUT -p udp -j RTPENGINE --id $TableID
```
特别注意点：
系统的openssl的版本必须为1.1.1以上，否则DTLS协商会出错。
spandsp必须使用freeswitch源安装，否则编译会出错.
当以上两个软件都安装完成后，就可以通过配置opensips.cfg的脚本来实现功能了，为了避免商业纠纷，以下以简化的opensips配置来举例说明：
```
#
# OpenSIPS residential configuration script
#     by OpenSIPS Solutions <team@opensips-solutions.com>
#
# Please refer to the Core CookBook at:
#      http://www.opensips.org/Resources/DocsCookbooks
# for a explanation of possible statements, functions and parameters.
#
 
 
####### Global Parameters #########
 
debug=3
log_stderror=no
log_facility=LOG_LOCAL0
 
fork=yes
children=4
auto_aliases=no
 
listen=udp:127.0.0.0:5060 # TODO: update with your local IP and port
listen=ws:127.0.0.0:8080 # TODO: update with your local IP and port
 
####### Modules Section ########
 
# set module path
mpath="/usr/local/lib/opensips/modules/"
 
#### SIGNALING module
loadmodule "signaling.so"
 
#### StateLess module
loadmodule "sl.so"
 
#### Transaction Module
loadmodule "tm.so"
modparam("tm", "fr_timeout", 5)
modparam("tm", "fr_inv_timeout", 30)
modparam("tm", "restart_fr_on_each_reply", 0)
modparam("tm", "onreply_avp_mode", 1)
 
#### Record Route Module
loadmodule "rr.so"
modparam("rr", "append_fromtag", 0)
 
#### MAX ForWarD module
loadmodule "maxfwd.so"
 
#### SIP MSG OPerationS module
loadmodule "sipmsgops.so"
 
#### FIFO Management Interface
loadmodule "mi_fifo.so"
modparam("mi_fifo", "fifo_name", "/tmp/opensips_fifo")
modparam("mi_fifo", "fifo_mode", 0666)
 
#### URI module
loadmodule "uri.so"
modparam("uri", "use_uri_table", 0)
 
#### USeR LOCation module
loadmodule "usrloc.so"
modparam("usrloc", "nat_bflag", "NAT")
modparam("usrloc", "db_mode",   0)
 
#### REGISTRAR module
loadmodule "registrar.so"
 
#### RTPengine protocol
loadmodule "rtpengine.so"
modparam("rtpengine", "rtpengine_sock", "udp:127.0.0.0:60000")
 
#### Nathelper protocol
loadmodule "nathelper.so"
modparam("registrar|nathelper", "received_avp", "$avp(rcv)")
 
#### UDP protocol
loadmodule "proto_udp.so"
 
#### WebSocket protocol
loadmodule "proto_ws.so"
 
 
####### Routing Logic ########
 
# main request routing logic
route{
	if (!mf_process_maxfwd_header("10")) {
		sl_send_reply("483","Too Many Hops");
		exit;
	}
 
	if (has_totag()) {
		# sequential requests within a dialog should
		# take the path determined by record-routing
		if (loose_route()) {
			if (is_method("INVITE")) {
				# even if in most of the cases is useless, do RR for
				# re-INVITEs alos, as some buggy clients do change route set
				# during the dialog.
				record_route();
			}
 
			# route it out to whatever destination was set by loose_route()
			# in $du (destination URI).
			route(relay);
		} else {
			if ( is_method("ACK") ) {
				if ( t_check_trans() ) {
					# non loose-route, but stateful ACK; must be an ACK after
					# a 487 or e.g. 404 from upstream server
					t_relay();
					exit;
				} else {
					# ACK without matching transaction ->
					# ignore and discard
					exit;
				}
			}
			sl_send_reply("404","Not here");
		}
		exit;
	}
 
	# CANCEL processing
	if (is_method("CANCEL")) {
		if (t_check_trans())
			t_relay();
		exit;
	}
 
	t_check_trans();
 
	if (!is_method("REGISTER")) {
		if (from_uri!=myself) {
			# if caller is not local, then called number must be local
			if (!uri==myself) {
				send_reply("403","Rely forbidden");
				exit;
			}
		}
	}
 
	# preloaded route checking
	if (loose_route()) {
		xlog("L_ERR",
		"Attempt to route with preloaded Route's [$fu/$tu/$ru/$ci]");
		if (!is_method("ACK"))
			sl_send_reply("403","Preload Route denied");
		exit;
	}
 
	# record routing
	if (!is_method("REGISTER|MESSAGE"))
		record_route();
 
	if (!uri==myself) {
		append_hf("P-hint: outbound\r\n");
		route(relay);
	}
 
	# requests for my domain
	if (is_method("PUBLISH|SUBSCRIBE")) {
		sl_send_reply("503", "Service Unavailable");
		exit;
	}
 
	# check if the clients are using WebSockets
	if (proto == WS)
		setflag(SRC_WS);
 
	# consider the client is behind NAT - always fix the contact
	fix_nated_contact();
 
	if (is_method("REGISTER")) {
 
		# indicate that the client supports DTLS
		# so we know when he is called
		if (isflagset(SRC_WS))
			setbflag(DST_WS);
 
		fix_nated_register();
		if (!save("location"))
			sl_reply_error();
 
		exit;
	}
 
	if ($rU==NULL) {
		# request with no Username in RURI
		sl_send_reply("484","Address Incomplete");
		exit;
	}
 
	# do lookup with method filtering
	if (!lookup("location","m")) {
		t_newtran();
		t_reply("404", "Not Found");
		exit;
	}
 
	route(relay);
}
 
route[relay] {
	# for INVITEs enable some additional helper routes
	if (is_method("INVITE")) {
		t_on_branch("handle_nat");
		t_on_reply("handle_nat");
	} else if (is_method("BYE|CANCEL")) {
		rtpengine_delete();
	}
 
	if (!t_relay()) {
		send_reply("500","Internal Error");
	};
	exit;
}
 
branch_route[handle_nat] {
 
	if (!is_method("INVITE") || !has_body("application/sdp"))
		return;
 
	if (isflagset(SRC_WS) && isbflagset(DST_WS))
		$var(rtpengine_flags) = "ICE=force-relay DTLS=passive";
	else if (isflagset(SRC_WS) && !isbflagset(DST_WS))
		$var(rtpengine_flags) = "RTP/AVP replace-session-connection replace-origin ICE=remove";
	else if (!isflagset(SRC_WS) && isbflagset(DST_WS))
		$var(rtpengine_flags) = "UDP/TLS/RTP/SAVPF ICE=force";
	else if (!isflagset(SRC_WS) && !isbflagset(DST_WS))
		$var(rtpengine_flags) = "RTP/AVP replace-session-connection replace-origin ICE=remove";
 
	rtpengine_offer("$var(rtpengine_flags)");
}
 
onreply_route[handle_nat] {
 
	fix_nated_contact();
	if (!has_body("application/sdp"))
		return;
 
	if (isflagset(SRC_WS) && isbflagset(DST_WS))
		$var(rtpengine_flags) = "ICE=force-relay DTLS=passive";
	else if (isflagset(SRC_WS) && !isbflagset(DST_WS))
		$var(rtpengine_flags) = "UDP/TLS/RTP/SAVPF ICE=force";
	else if (!isflagset(SRC_WS) && isbflagset(DST_WS))
		$var(rtpengine_flags) = "RTP/AVP replace-session-connection replace-origin ICE=remove";
	else if (!isflagset(SRC_WS) && !isbflagset(DST_WS))
		$var(rtpengine_flags) = "RTP/AVP replace-session-connection replace-origin ICE=remove";
 
	rtpengine_answer("$var(rtpengine_flags)");
}
```
