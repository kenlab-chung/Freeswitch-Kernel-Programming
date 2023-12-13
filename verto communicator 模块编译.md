# verto communicator 模块编译
## 1 环境
debian 11，gcc 10.2.1，openssl 1.1.1n ，freeswitch 1.10.10，测试终端 windows 10 64 位 (浏览器：Microsoft edge 115.0.1901.183 64 位 ，Chrome 112.0.5615.138 64位)

## 2 安装编译工具链
安装npm
```
sudo apt install npm　
```
安装nodejs版本管理工具n 
```
sudo npm install -g n
```
运行版本管理工具n 
```
sudo n
```
在下图中选择node/18.17.0
![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/0e73efe5-00d4-458a-85c9-4b9e4ceae90c)
 切换npm国内安装源(加快安装速度)
```
#查看当前的下包镜像源
npm config get registry
#将下包的镜像源切换为淘宝镜像源
npm config set registry=https://registry.npm.taobao.org/
#检查镜像源是否下载成功
npm config get registry
```
重启ssh终端。

安装工具bower和grunt
```
npm install -g bower grunt　
```
## 3 编译verto communicator程序
 进入到verto communicator目录：
```
cd  /opt/freeswitch-1.8.7/html5/verto/verto_communicator
```
执行npm install命令，安装npm依赖库：
```
npm install
```
分别安装bower依赖库，最后在合并：
```
mv  bower.json bower.json.back 
bower --allow-root init 
bower install --allow-root moment/moment@~2.9.0 --save 
bower install --allow-root jquery@~2.1.4 --save 
bower install --allow-root js-cookie/js-cookie@~1.4.1 --save 
bower install --allow-root jquery-json@~2.5.1 --save 
bower install --allow-root angular@~1.3.15 --save 
bower install --allow-root angular-gravatar@~0.4.1 --save 
bower install --allow-root bootstrap@~3.3.4 --save 
bower install --allow-root angular-toastr@~1.4.1 --save 
bower install --allow-root angular-sanitize@~1.3.15 --save 
bower install --allow-root angular-route@~1.3.15 --save 
bower install --allow-root bower-angular@~1.2.16 --save 
bower install --allow-root angular-prompt@~1.1.1 --save 
bower install --allow-root angular-animate@~1.3.15 --save 
bower install --allow-root angular-cookies@~1.3.15 --save 
bower install --allow-root angular-directive.g-signin@~0.1.2 --save 
bower install --allow-root angular-fullscreen@~1.0.1 --save 
bower install --allow-root ngstorage@~0.3.9 --save 
bower install --allow-root humanize-duration#~3.10.0 --save 
bower install --allow-root angular-timer@~1.3.3 --save 
bower install --allow-root angular-tooltips@~0.1.21 --save 
bower install --allow-root datatables@~1.10.8 --save 
bower install --allow-root angular-bootstrap@~0.14.3 --save 
bower install --allow-root mdbootstrap/bootstrap-material-design@~0.3.0 --save 
bower install --allow-root angular-translate@~2.10.0 --save 
bower install --allow-root angular-translate-loader-static-files@~2.10.0 --save 
bower install --allow-root angular-click-outside@~2.9.2 --save 
mv  bower.json.back  bower.json
```
修改bower.js配置文件：
```
"bootstrap-material-design": "~0.3.0" 修改为 "bootstrap-material-design": "mdbootstrap/bootstrap-material-design#~0.3.0"
```
使用bower合并安装依赖库：　　
```
bower --allow-root install
```
使用grunt构建程序
```
grunt build --force
```
Error: Cannot find module 'moment'  报错处理：
```
npm install moment --save
```
## 4 测试方式1
```
grunt serve
```
浏览器打开URL：https://192.168.1.127:9001
## 5 测试
部署apache服务器，并配置证书，参考《Verto模块启用》

将verto communicator 编译后的程序链接到/var/www/html/目录下 
```
sudo ln -s /opt/freeswitch-1.8.7/html5/verto/verto_communicator/dist /var/www/html/vc
```
打开浏览器（我用的Chrome）访问https://IP/vc，呼叫3500即可测试视频会议。　　
