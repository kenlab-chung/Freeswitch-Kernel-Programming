# FreeSWITCH内核编程
---
## 1 环境
FreeSWITCH : FreeSWITCH-1.10.11-dev

OS: debian bullseye

gcc:10.2.1

gdb:10.1.99

## 2 FreeSWITCH启动流程
### 2.1 main函数的主要工作
FreeSWITCH在main函数中除了初始化异常处理程序，解析软交换启动参数（比如：-nc -nonat 参数和 -conf -mod -log -run  -db -scripts -htdocs -base  -storage -temp -cache -recordings -grammar -certs -sounds等工作目录）之外，其核心就是调用switch_core_init_and_modload()函数初始化FreeSWITCH内核以及加载外围模块。
switch_core_init_and_modload()函数调用完毕，也就是FreeSWITCH内核和外围模块启动完毕后，main函数会调用switch_core_runtime_loop()函数进入无限循环。对于后台启动的实例，它基本上什么都没做。对于从前台启动的系统，switch_core_runtime_loop()会调用switch_console_loop()函数以启动一个控制台接收用户的键盘输入并打印系统运行信息，比如：命令行输出和日志等。
当实例退出switch_core_runtime_loop()函数后，调用switch_core_destroy()函数清理系统资源。

### 2.2 switch_console_loop函数主要工作
switch_console_loop()函数分两种情况启动，一种是使用跨平台库libedit库接收用户按键并在控制台上打印信息，一种是直接启动while循环接收用户按键以及在控制台上打印信息。不通的是使用libedit库时，函数内部启动了console_thread()线程，在这里检测用户命令合法性，并将命令存入历史记录，以备将来再执行该命令时可以使用键盘上的箭头按键翻看历史命令。同样，使用非libedit库的switch_console_loop()函数中也实现了相同的功能。
二者最终都是调用switch_console_process()函数，在switch_console_process函数内部执行switch_console_execute()函数执行命令行解析，并最终调用switch_api_execute()函数，执行命令输入，并将执行结果存放到istream中，最后会被取出并打印到控制台上。

### 2.3 switch_api_execute函数主要工作
通过switch_loadable_module_get_api_interface()查找外面模块中实现的api接口。并执行api->function回调函数。外围模块加载参考后续章节。

### 2.4 switch_core_init_and_modload函数主要工作
- 调用switch_core_init()函数初始FreeSWITCH内核。
- 调用switch_load_network_lists()函数初始化FreeSWITCH工作网络。
- 调用switch_loadable_module_init()函数外围模块。

#### 2.4.1 switch_core_init函数主要工作
初始FreeSWITCH工作目录，初始化软交换机状态机，加载软交换核心配置文件switch.conf等。

#### 2.4.2 switch_loadable_module_init函数主要工作
调用switch_xml_open_cfg和switch_loadable_module_load_module_ex函数分别加载pre_load_modules.conf，modules.conf，post_load_modules.conf三个配置文件中定义的模块。优先加载pre_load_modules.conf配置的模块，然后加载modules.conf配置的模块，最后加载post_load_modules.conf配置的模块。这些文件存放在conf/autoload_configs目录下。
其中switch_loadable_module_load_module_ex函数启动在软交换模块中SWITCH_MODULE_LOAD_FUNCTION、SWITCH_MODULE_RUNTIME_FUNCTION定义的load和runtime函数。

### 2.5 switch_core_destroy函数主要工作
当实例退出switch_core_runtime_loop()函数后，main()函数调用switch_core_destroy()完成内核资源释放、外围模块卸载(调用switch_loadable_module_shutdown函数)等资源清理操作。
## 3 模块结构
### 3.1 模块定义函数
FreeSWITCH已经定义好模块的框架，开发者只需根据模块的框架结构实现自身业务即可。其中，预定义的模块函数，函数指针及参数列表定义在switch_types.h头文件中。
```
#define SWITCH_MODULE_LOAD_ARGS (switch_loadable_module_interface_t **module_interface, switch_memory_pool_t *pool)
#define SWITCH_MODULE_RUNTIME_ARGS (void)
#define SWITCH_MODULE_SHUTDOWN_ARGS (void)
typedef switch_status_t (*switch_module_load_t) SWITCH_MODULE_LOAD_ARGS;
typedef switch_status_t (*switch_module_runtime_t) SWITCH_MODULE_RUNTIME_ARGS;
typedef switch_status_t (*switch_module_shutdown_t) SWITCH_MODULE_SHUTDOWN_ARGS;
#define SWITCH_MODULE_LOAD_FUNCTION(name) switch_status_t name SWITCH_MODULE_LOAD_ARGS
#define SWITCH_MODULE_RUNTIME_FUNCTION(name) switch_status_t name SWITCH_MODULE_RUNTIME_ARGS
#define SWITCH_MODULE_SHUTDOWN_FUNCTION(name) switch_status_t name SWITCH_MODULE_SHUTDOWN_ARGS
#define SWITCH_MODULE_DEFINITION_EX(name, load, shutdown, runtime, flags)                    　 \
static const char modname[] =  #name ;                                                        　\
SWITCH_MOD_DECLARE_DATA switch_loadable_module_function_table_t name##_module_interface = {     \
    SWITCH_API_VERSION,                                                                         \
    load,                                                                                       \
    shutdown,                                                                                   \
    runtime,                                                                                    \
    flags                                                                                       \
}
#define SWITCH_MODULE_DEFINITION(name, load, shutdown, runtime)                                 \
        SWITCH_MODULE_DEFINITION_EX(name, load, shutdown, runtime, SMODF_NONE)
```
- SWITCH_MODULE_LOAD_FUNCTION：模块加载函数，负责系统启动时或运行时加载模块，可以进行配置读取及资源初始化。
- SWITCH_MODULE_SHUTDOWN_FUNCTION：模块卸载函数，负载模块卸载及相关资源回收。
- SWITCH_MODULE_RUNTIME_FUNCTION：模块运行时函数，可以启动线程处理请求，监听socket等。
- SWITCH_MODULE_DEFINITION：模块定义函数，告知系统核心加载模块时调用load函数进行初始化，启动新线程执行runtime，卸载时调用shutdown函数清理资源。
### 3.2 模块加载流程
通过 switch_loadable_module_load_module() 函数加载模块，函数调用链如下：
```
switch_loadable_module_load_module 
        => switch_loadable_module_load_module_ex  
        => switch_loadable_module_load_file
        => switch_loadable_module_process
        => switch_core_launch_thread
     　　　　=>  switch_loadable_module_exec
```
在通过 switch_loadable_module_load_file()函数中通过switch_dso_data_sym()函数根据定义的 XXX_module_interface 从动态库里面获取回调函数指针，使用 switch_loadable_module_function_table_t 数据结构进行回调函数绑定。switch_loadable_module_exec 函数为独立线程中运行，模块运行时通过 module->switch_module_runtime触发。
所以整个模块加载过程如下：
```
main 
    => switch_core_init_and_modload 
        => switch_core_init
        => switch_loadable_module_init => switch_loadable_module_load_module
```
在switch_loadable_module_init()函数中首先加载系统核心模块：
```
switch_loadable_module_load_module_ex("", "CORE_SOFTTIMER_MODULE", SWITCH_FALSE, SWITCH_FALSE, &err, SWITCH_LOADABLE_MODULE_TYPE_COMMON, event_hash);
switch_loadable_module_load_module_ex("", "CORE_PCM_MODULE", SWITCH_FALSE, SWITCH_FALSE, &err, SWITCH_LOADABLE_MODULE_TYPE_COMMON, event_hash);
switch_loadable_module_load_module_ex("", "CORE_SPEEX_MODULE", SWITCH_FALSE, SWITCH_FALSE, &err, SWITCH_LOADABLE_MODULE_TYPE_COMMON, event_hash);
```
然后再通过switch_xml_open_cfg ()函数依次加载下列文件中定义的模块：
- pre_load_modules.conf
- modules.conf
- post_load_modules.conf
xml配置文件加载流程如下：
```
main => switch_core_init_and_modload 
        => switch_core_init 
            => switch_xml_init 
                => switch_xml_open_root => XML_OPEN_ROOT_FUNCTION
```
其中 SWITCH_GLOBAL_filenames 变量定义在main()函数中的 switch_core_set_globals()函数内部进行设置：
```
if (!SWITCH_GLOBAL_filenames.conf_name && (SWITCH_GLOBAL_filenames.conf_name = (char *) malloc(BUFSIZE))) {
        switch_snprintf(SWITCH_GLOBAL_filenames.conf_name, BUFSIZE, "%s", "freeswitch.xml");
    }
```
其中XML_OPEN_ROOT_FUNCTION实现如下：
```
static switch_xml_open_root_function_t XML_OPEN_ROOT_FUNCTION = (switch_xml_open_root_function_t)__switch_xml_open_root;

SWITCH_DECLARE_NONSTD(switch_xml_t) __switch_xml_open_root(uint8_t reload, const char **err, void *user_data)
{
    char path_buf[1024];
    uint8_t errcnt = 0;
    switch_xml_t new_main, r = NULL;
    if (MAIN_XML_ROOT) {
        if (!reload) {
            r = switch_xml_root();
            goto done;
        }
    }
    switch_snprintf(path_buf, sizeof(path_buf), "%s%s%s", SWITCH_GLOBAL_dirs.conf_dir, SWITCH_PATH_SEPARATOR, SWITCH_GLOBAL_filenames.conf_name);
    if ((new_main = switch_xml_parse_file(path_buf))) {
        *err = switch_xml_error(new_main);
        switch_copy_string(not_so_threadsafe_error_buffer, *err, sizeof(not_so_threadsafe_error_buffer));
        *err = not_so_threadsafe_error_buffer;
        if (!zstr(*err)) {
            switch_xml_free(new_main);
            new_main = NULL;
            errcnt++;
        } else {
            *err = "Success";
            switch_xml_set_root(new_main);
        }
    } else {
        *err = "Cannot Open log directory or XML Root!";
        errcnt++;
    }
    if (errcnt == 0) {
        r = switch_xml_root();
    }
 done:
    return r;
}
```
SWITCH_GLOBAL_filenames.conf_name即为freeswitch.xml文件，这个文件是FreeSWITCH中xml文件的总入口。并在freeswitch.xml文件中配置有加载各个模块的参数：
```
<section name="configuration" description="Various Configuration">
    <X-PRE-PROCESS cmd="include" data="autoload_configs/*.xml"/>
</section>
```
控制台动态加载流程，在fs_cli中可以使用load及reload加载模块，具体流程如下：
```
fs_cli => load ... => SWITCH_STANDARD_API(load_function) => switch_loadable_module_load_module 

fs_cli => reload ... => SWITCH_STANDARD_API(reload_function) => switch_loadable_module_unload_module 
                                                             => switch_loadable_module_load_module
```
### 3.3 关键数据结构
#### 3.3.1 switch_loadable_module_t
作用：用于定义模块信息。

结构体定义：
```
struct switch_loadable_module {
    char *key;                                                    //模块文件名
    char *filename;                                               //模块文件路径(动态库路径)
    int perm;                                                     //定义模块是否允许被卸载
    switch_loadable_module_interface_t *module_interface;         //模块接口(由switch_module_load函数赋值)
    switch_dso_lib_t lib;                                         //动态库句柄(dlopen函数返回)
    switch_module_load_t switch_module_load;                      //模块加载函数
    switch_module_runtime_t switch_module_runtime;                //模块运行时函数
    switch_module_shutdown_t switch_module_shutdown;              //模块关闭（卸载）函数
    switch_memory_pool_t *pool;                                   //模块内存池
    switch_status_t status;                                       //switch_module_shutdown 函数的返回值
    switch_thread_t *thread;
    switch_bool_t shutting_down;                                  //模块是否关闭
    switch_loadable_module_type_t type;
};typedef struct switch_loadable_module switch_loadable_module_t;
```
#### 3.3.2 switch_loadable_module_interface_t
作用：模块接口（入口）。

结构体定义：
```
/*! \brief The abstraction of a loadable module 模块接口（入口） */
    struct switch_loadable_module_interface {
    /*! the name of the module 模块的名称 */
    const char *module_name;
    /*! the table of endpoints the module has implemented 模块endpoint的具体实现 */
    switch_endpoint_interface_t *endpoint_interface;
    /*! the table of timers the module has implemented  模块timer的具体实现 */
    switch_timer_interface_t *timer_interface;
    /*! the table of dialplans the module has implemented 模块dialplan的具体实现 */
    switch_dialplan_interface_t *dialplan_interface;
    /*! the table of codecs the module has implemented 模块编解码的具体实现 */
    switch_codec_interface_t *codec_interface;
    /*! the table of applications the module has implemented 模块提供的app工具的具体实现 */
    switch_application_interface_t *application_interface;
    /*! the table of chat applications the module has implemented 模块提供的文本聊天app工具的具体实现 */
    switch_chat_application_interface_t *chat_application_interface;
    /*! the table of api functions the module has implemented 模块提供的api具体实现 */
    switch_api_interface_t *api_interface;
    /*! the table of json api functions the module has implemented 模块提供的json格式api的具体实现 */
    switch_json_api_interface_t *json_api_interface;
    /*! the table of file formats the module has implemented 模块支持的文件格式的具体实现（比如mp4、mkv等文件格式） */
    switch_file_interface_t *file_interface;
    /*! the table of speech interfaces the module has implemented  模块使用的speech接口实现 */
    switch_speech_interface_t *speech_interface;
    /*! the table of directory interfaces the module has implemented  模块使用的directory接口实现 */
    switch_directory_interface_t *directory_interface;
    /*! the table of chat interfaces the module has implemented 模块使用的chat接口实现 */
    switch_chat_interface_t *chat_interface;
    /*! the table of say interfaces the module has implemented 模块使用的say接口实现 */
    switch_say_interface_t *say_interface;
    /*! the table of asr interfaces the module has implemented  模块使用的asr接口实现 */
    switch_asr_interface_t *asr_interface;
    /*! the table of management interfaces the module has implemented 模块使用的管理接口实现 */
    switch_management_interface_t *management_interface;
    /*! the table of limit interfaces the module has implemented 模块使用的limit接口实现 */
    switch_limit_interface_t *limit_interface;
    /*! the table of database interfaces the module has implemented  模块使用的limit接口实现*/
    switch_database_interface_t *database_interface;
    switch_thread_rwlock_t *rwlock; //模块使用的锁
    int refs; //模块锁的计数器
    switch_memory_pool_t *pool;//模块内存池
};

typedef struct switch_loadable_module_interface switch_loadable_module_interface_t;
```
使用 switch_loadable_module_create_module_interface()函数 来创建 switch_loadable_module_interface_t 实例
```
SWITCH_DECLARE(switch_loadable_module_interface_t *) switch_loadable_module_create_module_interface(switch_memory_pool_t *pool,const char *name)
{
    switch_loadable_module_interface_t *mod;

    mod = switch_core_alloc(pool, sizeof(switch_loadable_module_interface_t));
    switch_assert(mod != NULL);

    mod->pool = pool;

    mod->module_name = switch_core_strdup(mod->pool, name);
    switch_thread_rwlock_create(&mod->rwlock, mod->pool);
    return mod;
}
```
使用 switch_loadable_module_create_interface 来创建模块里面的子接口，示例如下：
```
*module_interface = switch_loadable_module_create_module_interface(pool, modname);

rtc_endpoint_interface = switch_loadable_module_create_interface(*module_interface, SWITCH_ENDPOINT_INTERFACE);
rtc_endpoint_interface->interface_name = "rtc";
rtc_endpoint_interface->io_routines = &rtc_io_routines;
rtc_endpoint_interface->state_handler = &rtc_event_handlers;
rtc_endpoint_interface->recover_callback = rtc_recover_callback;
```
具体实现如下：
```
SWITCH_DECLARE(void *) switch_loadable_module_create_interface(switch_loadable_module_interface_t *mod,　switch_module_interface_name_t iname)
{
    switch (iname) {
    case SWITCH_ENDPOINT_INTERFACE:
        ALLOC_INTERFACE(endpoint)
    case SWITCH_TIMER_INTERFACE:
        ALLOC_INTERFACE(timer)
    case SWITCH_DIALPLAN_INTERFACE:
        ALLOC_INTERFACE(dialplan)
    case SWITCH_CODEC_INTERFACE:
        ALLOC_INTERFACE(codec)
    case SWITCH_APPLICATION_INTERFACE:
        ALLOC_INTERFACE(application)
    case SWITCH_CHAT_APPLICATION_INTERFACE:
        ALLOC_INTERFACE(chat_application)
    case SWITCH_API_INTERFACE:
        ALLOC_INTERFACE(api)
    case SWITCH_JSON_API_INTERFACE:
        ALLOC_INTERFACE(json_api)
    case SWITCH_FILE_INTERFACE:
        ALLOC_INTERFACE(file)
    case SWITCH_SPEECH_INTERFACE:
        ALLOC_INTERFACE(speech)
    case SWITCH_DIRECTORY_INTERFACE:
        ALLOC_INTERFACE(directory)
    case SWITCH_CHAT_INTERFACE:
        ALLOC_INTERFACE(chat)
    case SWITCH_SAY_INTERFACE:
        ALLOC_INTERFACE(say)
    case SWITCH_ASR_INTERFACE:
        ALLOC_INTERFACE(asr)
    case SWITCH_MANAGEMENT_INTERFACE:
        ALLOC_INTERFACE(management)
    case SWITCH_LIMIT_INTERFACE:
        ALLOC_INTERFACE(limit)
    case SWITCH_DATABASE_INTERFACE:
        ALLOC_INTERFACE(database)
    default:
        switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_WARNING, "Invalid Module Type!\n");
        return NULL;
    }
}
```
#### 3.3.3 switch_endpoint_interface_t
作用：Endpoints入口。

结构体定义：
```
struct switch_endpoint_interface {
    /*! the interface's name */                　　 // endpoint名称，比如："rtc"
    const char *interface_name; 
    /*! channel abstraction methods */             // endpoint对应的io操作回调函数
    switch_io_routines_t *io_routines; 
    /*! state machine methods */                   // endpoint对应的事件处理回调函数
    switch_state_handler_table_t *state_handler;
    /*! private information */                    //endpoint私有参数配置（比如编码格式、采样率等）
    void *private_info;
    switch_thread_rwlock_t *rwlock;               //endpoint锁
    int refs;                                     //endpoint锁的引用次数
    switch_mutex_t *reflock;                      //endpoint引用锁
    /* parent */
    switch_loadable_module_interface_t *parent;   //endpoint所属模块
    /* to facilitate linking */
    struct switch_endpoint_interface *next;        //next指针
    switch_core_recover_callback_t recover_callback; // endpoint对应的recover回调函数
};
typedef struct switch_endpoint_interface switch_endpoint_interface_t;
```
#### 3.3.4 switch_io_routines
作用：存储io操作的回调函数。

结构体定义：
```
struct switch_io_routines {
    /*! creates an outgoing session from given session, caller profile */
    switch_io_outgoing_channel_t outgoing_channel;         //创建外呼channel的回调函数
    /*! read a frame from a session */
    switch_io_read_frame_t read_frame;                    //读session音频数据的回调函数
    /*! write a frame to a session */
    switch_io_write_frame_t write_frame;                // 写session音频数据的回调函数
    /*! send a kill signal to the session's channel */
    switch_io_kill_channel_t kill_channel;                // kill信号处理函数，用于处理channel接收的kill信号
    /*! send a string of DTMF digits to a session's channel */
    switch_io_send_dtmf_t send_dtmf;                    //send dtmf操作的回调函数，用于处理channel接收的DTMF字符串
    /*! receive a message from another session */
    switch_io_receive_message_t receive_message;        // 处理channel消息的回调函数，用于处理其它channel发来的消息
    /*! queue a message for another session */
    switch_io_receive_event_t receive_event;            //发送channel消息的回调函数，用于向目标session发送自定义事件（比如rtc session、rtmp session等
    /*! change a sessions channel state */
    switch_io_state_change_t state_change;                //channel状态修改的回调函数
    /*! read a video frame from a session */
    switch_io_read_video_frame_t read_video_frame;        //读session视频数据的回调函数
    /*! write a video frame to a session */
    switch_io_write_video_frame_t write_video_frame;    //写session视频数据的回调函数
    /*! read a video frame from a session */
    switch_io_read_text_frame_t read_text_frame;        //读session文本数据的回调函数
    /*! write a video frame to a session */
    switch_io_write_text_frame_t write_text_frame;        //写session文本数据的回调函数
    /*! change a sessions channel run state */
    switch_io_state_run_t state_run;                    // 改变session的运行状态,保留
    /*! get sessions jitterbuffer */
    switch_io_get_jb_t get_jb;                            //获取session的jitter_buffer
    void *padding[10];
};

typedef struct switch_io_routines switch_io_routines_t;
```
#### 3.3.5 switch_state_handler_table_t
作用：存储状态机的回调函数

结构体定义：
```
struct switch_state_handler_table {
    /*! executed when the state changes to init //channel进入 CS_INIT 状态的回调函数 */
    switch_state_handler_t on_init;
    /*! executed when the state changes to routing  //  channel进入 CS_ROUTING 状态的回调函数*/
    switch_state_handler_t on_routing;
    /*! executed when the state changes to execute // channel进入 CS_EXECUTE 状态的回调函数，用于执行操作*/
    switch_state_handler_t on_execute;
    /*! executed when the state changes to hangup  channel进入 CS_HANGUP 状态的回调函数//  */
    switch_state_handler_t on_hangup;
    /*! executed when the state changes to exchange_media // channel进入 CS_EXCHANGE_MEDIA 状态的回调函数*/
    switch_state_handler_t on_exchange_media;
    /*! executed when the state changes to soft_execute  // channel进入 CS_SOFT_EXECUTE 状态的回调函数，用于从其它channel接收或发送数据*/
    switch_state_handler_t on_soft_execute;
    /*! executed when the state changes to consume_media // channel进入 CS_CONSUME_MEDIA 状态的回调函数*/
    switch_state_handler_t on_consume_media;
    /*! executed when the state changes to hibernate //channel进入 CS_HIBERNATE 状态的回调函数，sleep操作*/
    switch_state_handler_t on_hibernate;
    /*! executed when the state changes to reset //channel进入 CS_RESET 状态的回调函数 */
    switch_state_handler_t on_reset;
    /*! executed when the state changes to park  //channel进入 CS_PARK 状态的回调函数 */
    switch_state_handler_t on_park;
    /*! executed when the state changes to reporting  //channel进入 CS_REPORTING 状态的回调函数*/
    switch_state_handler_t on_reporting;
    /*! executed when the state changes to destroy  //channel进入 CS_DESTROY 状态的回调函数*/
    switch_state_handler_t on_destroy;
    int flags;
    void *padding[10];
};

typedef struct switch_state_handler_table switch_state_handler_table_t;
```
`switch_core_state_machine.c文件switch_core_session_run()`函数中使用 STATE_MACRO 触发，部分触发代码如下：
```
case CS_ROUTING:    /* Look for a dialplan and find something to do */
    STATE_MACRO(routing, "ROUTING");
    break;
case CS_RESET:        /* Reset */
    STATE_MACRO(reset, "RESET");
    break;
    /* These other states are intended for prolonged durations so we do not signal lock for them */
case CS_EXECUTE:    /* Execute an Operation */
    STATE_MACRO(execute, "EXECUTE");
    break;
case CS_EXCHANGE_MEDIA:    /* loop all data back to source */
    STATE_MACRO(exchange_media, "EXCHANGE_MEDIA");
    break;
case CS_SOFT_EXECUTE:    /* send/recieve data to/from another channel */
    STATE_MACRO(soft_execute, "SOFT_EXECUTE");
    break;
case CS_PARK:        /* wait in limbo */
    STATE_MACRO(park, "PARK");
    break;
case CS_CONSUME_MEDIA:    /* wait in limbo */
    STATE_MACRO(consume_media, "CONSUME_MEDIA");
    break;
case CS_HIBERNATE:    /* sleep */
    STATE_MACRO(hibernate, "HIBERNATE");
    break;
```

### 3.4 模块开发实例
#### 3.4.1 实现一个api
以模块mod_bsoft为例。实现一个hello_bsoft api接口。
在src/mod/endpoints目录下创建mod_bsoft目录。
把模块名加入到modules.conf中，make时根据此文件选择编译哪些模块，并生成相应的makefile文件。

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/61d53f62-da03-4f52-8876-1272e49e8e31)

在configure.ac中加入mod_bsoft的Makefile配置：

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/d2dea185-25d0-4cf6-aa48-6d1e5ec80f90)

在mod_bsoft目录中创建mod_bsoft.c文件。代码如下：
```
/*************************************************************************
  > File Name: mod_bsoft.c
  > Author: zhongqf
  > Mail: zhongqf.cn@gmail.com
  > Created Time: 2023-11-22 15:42:03
 ************************************************************************/

#include<switch.h>
SWITCH_MODULE_LOAD_FUNCTION(mod_bsoft_load);
SWITCH_MODULE_RUNTIME_FUNCTION(mod_bsoft_runtime);
SWITCH_MODULE_SHUTDOWN_FUNCTION(mod_bsoft_shutdown);
SWITCH_MODULE_DEFINITION(mod_bsoft,mod_bsoft_load,mod_bsoft_shutdown,mod_bsoft_runtime);

SWITCH_STANDARD_API(hello_bsoft)
{
    switch_log_printf(SWITCH_CHANNEL_LOG,SWITCH_LOG_NOTICE,"hello bsoft!\n");
    return SWITCH_STATUS_SUCCESS;
}

SWITCH_MODULE_LOAD_FUNCTION(mod_bsoft_load)
{
    switch_api_interface_t *api_interface;
    *module_interface = switch_loadable_module_create_module_interface(pool,modname);
    SWITCH_ADD_API(api_interface,"hello_bsoft","hello bosft API",hello_bsoft,"syntax");
    return SWITCH_STATUS_SUCCESS;
}
SWITCH_MODULE_RUNTIME_FUNCTION(mod_bsoft_runtime)
{
    int i=0;
    while(i<=10)
    {
        switch_log_printf(SWITCH_CHANNEL_LOG,SWITCH_LOG_NOTICE,"hello from bsoft!\n");
        switch_yield(100000);
        i++;
    }
    return SWITCH_STATUS_TERM;
}
SWITCH_MODULE_SHUTDOWN_FUNCTION(mod_bsoft_shutdown)
{
    switch_log_printf(SWITCH_CHANNEL_LOG,SWITCH_LOG_NOTICE,"mod_bsoft shutdown!\n");
    return SWITCH_STATUS_SUCCESS;

}
```
编写makefile.am文件： 
```
include $(top_srcdir)/build/modmake.rulesam
MODNAME=mod_bsoft

mod_LTLIBRARIES = mod_bsoft.la
mod_bsoft_la_SOURCES  = mod_bsoft.c
mod_bsoft_la_CFLAGS   = $(AM_CFLAGS)
mod_bsoft_la_LIBADD   = $(switch_builddir)/libfreeswitch.la
mod_bsoft_la_LDFLAGS  = -avoid-version -module -no-undefined -shared
```
执行freeswitch源码目录下执行
```
./bootstrap.sh && ./configure --prefix=/opt/freeswitch_install/
```
可以看到mod_bsoft目录下产生了一个Makefile文件。此时可以执行在此目录下编译单独编译模块，并部署到安装目录下。
```
make && make install
```
也可以在freeswitch元目录下执行：
```
make mod_bsoft && make mod_bsoft-install
```
在freeswitch安装目录下可以看到mod_bsoft模块相关文件：

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/1c274c3e-0ef7-4ffe-a745-cacf4dc2d0ee)

在控制台中加载mod_bsoft模块，可以看到模块可以正常加载，并打印runtime中的日志。

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/935a35f7-a763-4388-823d-a777ec76b36e)

可以调用模块中定义的hello_bsoft接口

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/630e2f43-e5e7-4851-b449-8f49585a71a0)

可以正常卸载：

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/bdd76918-d75d-4a84-8aaf-2e806b710555)

### 3.4.2 实现一个Dialplan


