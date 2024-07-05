# FreeSWITCH防止SIP攻击设置

FreeSwtich作为一款开源的软交换控制系统，由于其出色的性能和稳定性，已经在全世界范围内得到了广泛的应用，包括一些大型的公司也都在使用。当FreeSwitch一旦部署在公有云上，有经验的使用者很快都会发现，有大量的非正常呼叫请求消息会对系统进行攻击，这有可能会造成系统被盗打线路的风险。其中几乎全部的攻击均来自于国外,本篇文章从系统防火墙的角度来配置如果防范来自国外的SIP攻击，文章的系统环境为Centos7，防火墙使用自带的firewalld。

- 第一步，我们配置一个名字叫china的ipset：
```
firewall-cmd --permanent --zone=public --new-ipset=china --type=hash:net
```
第二步，我们通过linux脚本向这个ipset中增加中国国内的ip地址段，国内地址段的数据来自于`http://www.ipdeny.com/ipblocks/data/countries/cn.zone`,通过bash执行脚本文件。
```
#!/bin/bash
rm -f cn.zone
wget http://www.ipdeny.com/ipblocks/data/countries/cn.zone
for i in $(cat cn.zone)
do
   firewall-cmd --permanent --ipset=china --add-entry=$i >> /dev/null 2>&1
done
```
 这步的执行比较慢，稍后介绍如果优化处理这个问题。

 - 第三步，我们将这个ipset配置给freeswitch的端口用于访问控制。
```
firewall-cmd --permanent --add-rich-rule "rule family="ipv4" source ipset="china" port protocol="udp" port="5080" accept"
firewall-cmd --permanent --add-rich-rule "rule family="ipv4" source ipset="china" port protocol="udp" port="5060" accept"
firewall-cmd --reload 
service firewalld restart
```
这样就完成了配置，国外的攻击将被防火墙挡住。

再回顾下第二步中写ipset比较慢的问题，其实研究下ipset，发现地址段是保存在如下的位置中，在这个位置中会有一个china.xml的文件：
```
/etc/firewalld/ipsets/
```
![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/6726ca08-2c70-4de6-9728-e18dbcf4335a)

我们可以将这个文件保存下来，以后重新安装或者在其他机器安装的话，只需在完成第一步后，用这个文件替换系统生成的china.xml即可，但是需要执行如下命令来重启下防火墙，然后再执行第三步就可以了。

最后重启防火墙：
```
firewall-cmd –reload 
service firewalld restart
```
