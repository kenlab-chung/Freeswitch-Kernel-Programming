# Docker 构建FreeSWITCH 操作指南
## 构建PostgreSQL数据库
创建文件Dockerfile.postgres
```
# 指定基础镜像
FROM postgres:latest
 
# 设置数据库密码（如果需要）
ENV POSTGRES_PASSWORD=postgres
 
# 将本地目录与容器内部的/var/lib/postgresql/data目录关联起来
#VOLUME /var/lib/postgresql/data
 
# 复制自定义配置文件到容器内部
# COPY postgresql.conf /etc/postgresql/postgresql.conf
 
# 运行postgres命令初始化数据库
CMD ["postgres"]
```
执行一下命令构建镜像：
```
sudo docker build -f ./Dockerfile.postgres  -t bsoft-postgresql:ori .
```
构建完成后可以查看已生成的镜像列表：
```
docker images
```
## 启动docker 镜像
编写docker-compose文件
```
version: '3'  # Docker Compose file version

services:
  db:
    image: bsoft-postgresql:v1.0.0
    #restart: always
    container_name: bsoft-db
    volumes:
      - /opt/postgresql-16/data:/var/lib/postgresql/data
    ports:
      - "5433:5432"

```
执行如下指令启动docker
```
sudo docker-compose -f docker-compose.yml up //sudo docker-compose -f docker-compose.yml up -d 表示后台启动
```
修改容器内容后，需要保存为镜像，可以导出到其它地方使用：
```
sudo docker commit 67890320b1ed bsoft-pg:v1.0.0
sudo docker save -o /opt/bsoft-pg-v1.0.0.tar bsoft-pg:v1.0.0
```
在其它地方导入镜像：
```
docker load -i /home/bsoft-pg-v1.0.0.tar
```
使用docker-compose 启动docker容器
```
sudo docker-compose -f docker-compose.yml up -d
```

## 构建FreeSWITCH软交换

创建文件Dockerfile.postgres
```
# 指定基础镜像
FROM debian:bullseye

COPY ./lib/ /usr/local/lib/
COPY ./bsoft-switch/bin/ /usr/local/freeswitch/bin/
COPY ./bsoft-switch/conf/ /usr/local/freeswitch/conf/
COPY ./bsoft-switch/fonts/ /usr/local/freeswitch/fonts/
COPY ./bsoft-switch/grammar/ /usr/local/freeswitch/grammar/
COPY ./bsoft-switch/htdocs/ /usr/local/freeswitch/htdocs/
COPY ./bsoft-switch/images/ /usr/local/freeswitch/images/
COPY ./bsoft-switch/lib/ /usr/local/freeswitch/lib/
COPY ./bsoft-switch/mod/ /usr/local/freeswitch/mod/
COPY ./bsoft-switch/scripts/ /usr/local/freeswitch/scripts/
COPY ./bsoft-switch/sounds/ /usr/local/freeswitch/sounds/
COPY ./bsoft-switch/include/ /usr/local/freeswitch/include/
COPY ./bsoft-switch/storage/ /usr/local/freeswitch/storage/


CMD ["/usr/local/freeswitch/bin/freeswitch","-c"]
```
## 启动FreeSWITCH 镜像
编写docker-compose文件
```
version: '3'  # Docker Compose file version

services:
  bsoft-switch:
    image: bsoft-switch:ori
    #restart: always
    container_name: bsoft-switch
    network_mode: host #host 模式启动时，不需要映射端口
    volumes:
      - /opt/bsoft-switch/conf:/usr/local/freeswitch/conf
      - /opt/bsoft-switch/recordings:/usr/local/freeswitch/recordings
      - /opt/bsoft-switch/scripts:/usr/local/freeswitch/scripts
      - /opt/bsoft-switch/certs:/usr/local/freeswitch/certs
      - /opt/bsoft-switch/log:/usr/local/freeswitch/log

        #ports: 
        #- "5060:5060"
        #- "5080:5080"

```
非host 模式启动
```version: '3'  # Docker Compose file version

services:
  bsoft-switch:
    image: bsoft-switch:v1.0.1
    #restart: always
    container_name: bsoft-switch
    #network_mode: host
    volumes:
      - /opt/bsoft-switch/conf:/usr/local/freeswitch/conf
      - /opt/bsoft-switch/recordings:/usr/local/freeswitch/recordings
      - /opt/bsoft-switch/scripts:/usr/local/freeswitch/scripts
      - /opt/bsoft-switch/certs:/usr/local/freeswitch/certs
      - /opt/bsoft-switch/log:/usr/local/freeswitch/log

    ports:
      - "5060:5060/udp"
      - "5080:5080/udp"
      - "5060:5060"
      - "5080:5080"
      - "16384-16584:16384-16584/udp"
      - "8021:8021"


```
## docker-compose安装
```
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```
## 构建ZLMediaKit镜像
Dockerfile
```
# 指定基础镜像
FROM debian:bullseye

COPY ./bsoft-media/lib/ /usr/local/lib/
COPY ./bsoft-media/MediaServer /opt/BsoftMedia/
COPY ./bsoft-media/config.ini /opt/BsoftMedia/conf/
COPY ./bsoft-media/supervisord.media.conf /etc/supervisor/conf.d/supervisord.conf
RUN ldconfig
RUN apt-get update && apt-get install -y supervisor
RUN apt-get autoremove
RUN chmod a+x /opt/BsoftMedia/MediaServer
```
supervisord.conf
```
[supervisord]
nodaemon=true

[program:first_app]
command=/opt/BsoftMedia/MediaServer -c /opt/BsoftMedia/conf/config.ini
```
docker-compose.yml
```
version: '3'  # Docker Compose file version

services:
  bsoft-media:
    image: bsoft-media:ori
    #restart: always
    container_name: bsoft-media
    volumes:
      - /opt/bsoft-media/conf/:/opt/BsoftMedia/conf/
      - /opt/bsoft-media/logs/media/:/opt/BsoftMedia/log/

    ports:
      - "554:554"
      - "332:332"
      - "1935:1935"
      - "19350:19350"
      - "80:80"
      - "443:443"
      - "9000:9000"
      - "10000:10000"
      - "9000:9000/udp"
      - "10000:10000/udp"
      - "50000-50300:50000-50300/udp"
```
## 构建GB28181镜像
Dockerfile
```
# 指定基础镜像
FROM debian:bullseye

WORKDIR /opt/BsoftMedia/

RUN apt-get update && apt-get install -y openjdk-11-jdk

COPY ./bsoft-media/wvp-pro-2.6.9-08180706.jar .
COPY ./bsoft-media/application-local.yml ./conf/

#COPY ./bsoft-media/supervisord.gb.conf /etc/supervisor/conf.d/supervisord.conf

RUN chmod a+x /opt/BsoftMedia/wvp-pro-2.6.9-08180706.jar
RUN apt-get autoremove

CMD ["java","-jar","wvp-pro-2.6.9-08180706.jar"]

```
docker-compose.yml
```
version: '3'  # Docker Compose file version

services:
  bsoft-media-gb28181:
    image: bsoft-media-gb28181:v1.0.0
    #restart: always
    container_name: bsoft-media-gb28181-v1.0.0
    command: ["java", "-jar", "wvp-pro-2.6.9-08180706.jar","--spring.config.location=./conf/application-local.yml"]
    volumes:
      - /opt/bsoft-media/conf/:/opt/BsoftMedia/conf/
      - /opt/bsoft-media/logs/gb28181/:/logs/
    ports:
      - "18080:18080"
      - "5090:5090"
      - "5090:5090/udp"

```
