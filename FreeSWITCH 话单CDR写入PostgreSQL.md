# FreeSWITCH 话单CDR写入PostgreSQL数据库
## 1 编译mod_cdr_pg_csv模块
取消在modules.conf中取消对event_handlers/mod_cdr_pg_csv的注释
在freeswitch源码路径下执行
```
./configure --enable-core-pgsql-support --prefix=/opt/freeswitch_install/
make mod_cdr_pg_csv-install
```
## 2 手动建表
手动创建数据库cdr表，语句如下：
```
create table cdr (
    id                        serial primary key,
    local_ip_v4               inet not null,
    caller_id_name            varchar,
    caller_id_number          varchar,
    destination_number        varchar not null,
    context                   varchar not null,
    start_stamp               timestamp with time zone not null,
    answer_stamp              timestamp with time zone,
    end_stamp                 timestamp with time zone not null,
    duration                  int not null,
    billsec                   int not null,
    hangup_cause              varchar not null,
    uuid                      uuid not null,
    bleg_uuid                 uuid,
    accountcode               varchar,
    read_codec                varchar,
    write_codec               varchar,
    sip_hangup_disposition    varchar,
    ani                       varchar
);
```
## 3 修改配置
修改cdr_pg_csv.conf.xm配置文件中数据库连接信息：
```
<param name="db-info" value="host=127.0.0.1 dbname=freeswitch user=postgres password=postgres connect_timeout=10" />
```
## 4 加载模块
```
load mod_cdr_pg_csv
```
