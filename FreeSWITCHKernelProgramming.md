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

#### 3.4.2 实现一个Dialplan
在SWITCH_MODULE_LOAD_FUNCTION中新增一个Dialplan变量声明：
```
switch_dialplan_interface_t *dp_interface;
```
在*module_interface之后，向Freeswitch核心注册一个新的Dialplan，并设置一个回调函数：
```
SWITCH_ADD_DIALPLAN(dp_interface,"bsoft_dialplan",bsoft_dialplan_hunt);
```
实现这个回调函数：
```
SWITCH_STANDARD_DIALPLAN(bsoft_dialplan_hunt)
{
    switch_caller_extension_t * extension = NULL;
    switch_channel_t *channel = switch_core_session_get_channel(session);
    if(!caller_profile)
    {
        caller_profile =switch_channel_get_caller_profile(channel);
    }
    switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(session),SWITCH_LOG_INFO,
            "Processing %s <%s>->%s in context %s\n",
            caller_profile->caller_id_name,
            caller_profile->caller_id_number,
            caller_profile->destination_number,
            caller_profile->context
            );

    extension = switch_caller_extension_new(session,"bsoft_dialplan","bsoft_dialplan");

    if(!extension) abort();

    switch_caller_extension_add_application(session,extension,"log","Info , bsoft_dialplan");

    return extension;
}
```
在控制台执行：
```
freeswitch@debian> originate user/1001 9999 bsoft_dialplan
```
![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/efdb2663-dd12-4ec3-a68d-f789dfe47804)

其中9999为Extension，bsoft_dialplan为Dialplan，Context没设置，默认为default。
 
#### 3.4.3 实现一个application
在SWITCH_MODULE_LOAD_FUNCTION中新增一个Application变量声明
```
switch_application_interface_t *app_interface;
```
在*module_interface之后，向Freeswitch核心注册一个新的Application，并设置一个回调函数：
```
 SWITCH_ADD_APP(app_interface,"bsoft_app","bsoft_app demo","bsoft_app Demo",bsoft_app_fun,"[name]",SAF_SUPPORT_NOMEDIA);
```
实现这个函数：
```
SWITCH_STANDARD_APP(bsoft_app_fun)
{

}
```
通过控制台执行show modules mod_bsoft命令可以看到刚刚实现的api 、dialplan和application:

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/58014879-c6cb-468a-8597-65599d952e93)

#### 3.4.4 事件订阅
在mod_bsoft_load()函数中添加 switch_event_bind()。
```
 switch_event_bind("mod_bsoft", SWITCH_EVENT_ALL, SWITCH_EVENT_SUBCLASS_ANY, on_channel_progress, NULL); 
```
并实现事件侦听函数：
```
static void  on_channel_progress(switch_event_t *event) 
{
     switch_log_printf(SWITCH_CHANNEL_LOG,SWITCH_LOG_INFO,"event id:%d owner:%s event name:%s\n",
　　　event->event_id,event->owner,event->subclass_name);
}
```
## 4 调试
### 4.1 设置日志级别 
mod_sofia模块接口设置sofia sip协议栈的日志级别，0关闭调试日志，9最高包括函数调用退出流程的日志打印。all会影响所有模块的日志级别。
```
sofia loglevel <all|default|tport|iptsec|nea|nta|nth_client|nth_server|nua|soa|sresolv|stun> [0-9]
```
开启sofia最高级别日志：
```
fsctl loglevel 6
sofia loglevel all 9
```
关闭sofia日志
```
sofia loglevel all 0
```
###  4.2 堆栈打印
使用系统库函数打印堆栈来查看调用流程。以通过堆栈来查看channel初始化流程为例。在FreeSWITCH呼叫业务中，我们最常见的日志就是呼叫通道的启动信息，日志如下：
```
[NOTICE] switch_channel.c:1142 New Channel sofia/internal/1001@192.168.1.2 [c30832f7-7af7-4517-9ea9-310f6c4da6a4]
```
这个日志表示一个新的会话channel初始完成。定位到src/switch_channel.c文件switch_channel_set_name()函数。在New Channel 信息打印之后执行print_stack()函数。

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/06569ee8-2046-4c56-a59c-a632fa5b1227)

实现print_stack()函数：
```
#include <execinfo.h>
static void print_stack()
{
        void *stacktrace[99];
        char **symbols;
        int size,i;
        size = backtrace(stacktrace,99);
        symbols = backtrace_symbols(stacktrace,size);
        if(!symbols)
        {
                return;
        }
        for(i=0;i<size;i++)
        {
                switch_log_printf(SWITCH_CHANNEL_LOG,SWITCH_LOG_NOTICE,"STACK:%s\n",symbols[i]);
        }
        free(symbols);
}
```
重新编译和安装FreeSWITCH。启动FreeSWITCH后发起INVITE呼叫。
```
switch_channel.c:1161 New Channel sofia/internal/1001@192.168.1.2 [1f1d7158-c050-463a-893b-ce7be864d7dd]
switch_channel.c:1140 STACK:/opt/freeswitch_install/lib/libfreeswitch.so.1(+0x6268f) [0x7fdd7190468f]
switch_channel.c:1140 STACK:/opt/freeswitch_install/lib/libfreeswitch.so.1(switch_channel_set_name+0x176) [0x7fdd7190c096]
switch_channel.c:1140 STACK:./lib/freeswitch/mod/mod_sofia.so(+0x551a4) [0x7fdd6de7b1a4]
switch_channel.c:1140 STACK:./lib/freeswitch/mod/mod_sofia.so(+0x53ed2) [0x7fdd6de79ed2]
switch_channel.c:1140 STACK:/home/freeswitch-1.10.10-dev/lib/libsofia-sip-ua.so.0(+0xc7e1b) [0x7fdd710d1e1b]
switch_channel.c:1140 STACK:/home/freeswitch-1.10.10-dev/lib/libsofia-sip-ua.so.0(+0x135313) [0x7fdd7113f313]
switch_channel.c:1140 STACK:/home/freeswitch-1.10.10-dev/lib/libsofia-sip-ua.so.0(su_base_port_getmsgs+0x73) [0x7fdd7113f09d]
switch_channel.c:1140 STACK:/home/freeswitch-1.10.10-dev/lib/libsofia-sip-ua.so.0(su_base_port_step+0x165) [0x7fdd7113f655]
switch_channel.c:1140 STACK:/home/freeswitch-1.10.10-dev/lib/libsofia-sip-ua.so.0(+0x131734) [0x7fdd7113b734]
switch_channel.c:1140 STACK:/home/freeswitch-1.10.10-dev/lib/libsofia-sip-ua.so.0(su_root_step+0x6e) [0x7fdd7113c87b]
switch_channel.c:1140 STACK:./lib/freeswitch/mod/mod_sofia.so(+0x3dcb3) [0x7fdd6de63cb3]
switch_channel.c:1140 STACK:/lib/x86_64-linux-gnu/libpthread.so.0(+0x7ea7) [0x7fdd7186cea7]
switch_channel.c:1140 STACK:/lib/x86_64-linux-gnu/libc.so.6(clone+0x3f) [0x7fdd7178ca2f]
```
通过“addr2line”工具，使用模块名和偏移地址，查找函数名：
```
addr2line 0x6268f -e /opt/freeswitch_install/lib/libfreeswitch.so.1 -f
print_stack
/opt/freeswitch/src/switch_channel.c:1133

addr2line 0x551a4 -e ./lib/freeswitch/mod/mod_sofia.so -f
sofia_glue_set_name
/opt/freeswitch/src/mod/endpoints/mod_sofia/sofia_glue.c:70

addr2line 0x53ed2 -e ./lib/freeswitch/mod/mod_sofia.so -f
set_call_id
/opt/freeswitch/src/mod/endpoints/mod_sofia/sofia.c:2385

addr2line 0xc7e1b -e /home/freeswitch-1.10.10-dev/lib/libsofia-sip-ua.so.0 -f
nua_application_event
nua_stack.c:?

addr2line 0x135313 -e /home/freeswitch-1.10.10-dev/lib/libsofia-sip-ua.so.0 -f
su_base_port_execute_msgs
su_base_port.c:?

addr2line 0x131734 -e /home/freeswitch-1.10.10-dev/lib/libsofia-sip-ua.so.0 -f
su_port_step
su_root.c:?

addr2line 0x3dcb3 -e ./lib/freeswitch/mod/mod_sofia.so -f
sofia_profile_thread_run
/opt/freeswitch/src/mod/endpoints/mod_sofia/sofia.c:3472

addr2line 0x7ea7 -e /lib/x86_64-linux-gnu/libpthread.so.0 -f
start_thread
./nptl/pthread_create.c:478 (discriminator 6)
```
可以看出Invite初始channel流程如下：
```
start_thread()
=>sofia_profile_thread_run()//mod_sofia模块，启动profile端口监听
    =>su_root_step()
        =>su_port_step()
            =>su_base_port_step() //sofia_sip库，端口收到消息。
                =>su_base_port_getmsgs()
                    =>su_base_port_execute_msgs() //sofia_sip库，分发消息
                        =>nua_application_event()
                            =>set_call_id()
                                =>sofia_glue_set_name() //mod_sofia模块，设置channel名称。
                                    =>switch_channel_set_name() //freeswitch核心，设置channel名称。
                                        =>print_stack()
```
## 5 呼叫流程
在fs_cli命令发一路外呼，注意字符串写的是sofia/192.168.1.2/1001,直接指定sofia的具体profile，而不是user/1001，因为后者是个虚拟的逻辑概念。
```
originate sofia/192.168.1.2/1001 &bridge(sofia/192.168.1.2/1002)
```
fs_cli是典型的ESL客户端，所以接收端是mod_event_socketd的监听线程listener_run。
```
listener_run()
    =>parse_command()
        =>api_exec()
            =>switch_api_execute()
                =>switch_loadable_module_get_api_interface()
                    =>originate_function() //mod_dpcommand
                        =>switch_ivr_originate()
                            =>switch_core_session_outgoing_channel()
                                =>endpoint_interface->io_routines->outgoing_channel()
                                    =>sofia_outgoing_channel() //mod_sofia
                                        =>switch_core_session_request_uuid() 
                                            =>switch_channel_init()
```
最终在mod_sofia模块中创建Session对象，并调用switch_channel_init()将状态机设置为CS_INIT。
状态机驱动之后，调用mod_sofia模块的状态回调函数sofia_on_init()函数中的sofia_glue_do_invite()函数发送INVITE消息。然后，核心状态函数switch_core_standard_on_init()驱动状态迁移到CS_ROUTING状态。这时拨号方案执行列表就是我们最开始的命令：
```
&bridge(sofia/192.168.1.2/1002)
```
originate模块添加的状态回调函数originate_on_routing被调用时，驱动状态机迁移到CS_CONSUME_MEDIA状态。这时originate 挂起，等待被叫方的响应。

sofia协议栈收到消息时，抛出事件，并调用sofia_event_callback回调函数。对于应答消息18X和2XX，调用链差不多：
```
sofia_event_callback()
    =>sofia_queue_message()
        =>sofia_process_dispatch_event()
            =>our_sofia_event_callback
                =>sofia_handle_sip_i_state()
```
最后18X调用switch_channel_mark_pre_answered()函数，而2XX调用switch_channel_mark_answered()函数。 被叫应答后，之前被挂起的originate激活，根据返回的结果，驱动状态机继续迁移，对于200OK，也就是answer，置状态为CS_EXECUTE。 接下来的迁移就取决于后续的处理了，本例是，执行完bridge后执行1002呼叫流程。
## 6 mod_sofia模块上报自定义头域
FreeSWITCH event事件一般都是底层定义好的，这些信息已经很完备，可以满足日常开发需求。但是在某些特殊需求下，需要进行额外的处理。比如，FreeSWITCH注册时间中，就没有X-自定义头域信息。
在定制化的SIP交换过程中，FreeSWITCH支持自定义头域，头域格式要满足“X-***”的模式。而当我们订阅“sofia::register”事件，在事件中是无法获取自定义头域信息的。
修改方式如下，在src/mod/endpionts/mod_sofia/sofia_reg.c文件中找到sofia_reg_handle_register_token()函数，并在函数中找到MY_EVENT_REGISTER "sofia::register" 事件的创建和上报流程，添加如下代码：

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/d3c32c5b-fa45-4e5f-bc09-d50e49a9619f)

从上面的代码中，我们可以看到“sofia::register”事件中包含的所有头域内容。sofia sip协议栈会解析所有的头域，并把非标准的头域都放到“sip->sip_unknown”中。增的代码中，我们把所有unknown的头域都放到register上报事件中去。
重新加载mod_sofia模块后，可以看到注册事件中看到"X-"开头的自定义头域消息。

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/de3eec6a-bea9-4bf6-96ce-944801593c8b)

## 7 Endpoints媒体交互
endpoints通讯的目的就是双方都向对方推送己方的媒体数据（包括音频数据、视频数据）。Endpoints媒体的交互在被叫发送SIP/2.0 200 OK之后便开启RTP数据包推送。其中音频和视频数据是分开进行RTP打包的。而且音频RTP包和视频RTP包分别使用各自通讯端口。比如：192.168.1.2上音频RTP数据包收发端口为19132，视频RTP数据包收发端口为16610。如下图所示：

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/8b7657cd-f8d3-4cd3-b777-eaf9fe5f7606)

### 7.1 originate 流程
#### 7.1.1 originate命令使用
originate用于发起呼叫，命令使用的基础模板：
```
originate ALEG BLEG
```
在fs_cli控制台使用的完整语法如下：
```
originate <call url> <exten>|&<application_name>(<app_args>) [<dialplan>][&lt;context>] [<cid_name>][&lt;cid_num>] [<timeout_sec>]
```
其中，
originate 为命令关键字，为必选字段，用于定义ALEG的呼叫信息，也就是通常说的呼叫字符串，可以通过通道变量定义很多参数;
|&<application_name>(<app_args>) 为必选字段，用于指定BLEG的分机号码或者用于创建BLEG的app（比如echo、bridge等）;
[][<context>] 可选参数，该参数用于指定dialplan的context，默认值：xml default ;
[<timeout_sec>] 可选参数，该参数用于指定originate超时，默认值：60

#### 7.1.2 originate功能入口函数
入口函数为originate_function，在 mod_commands_load 中绑定：
```
SWITCH_ADD_API(commands_api_interface, "originate", "Originate a call", originate_function, ORIGINATE_SYNTAX);
```
具体实现如下：
```
SWITCH_STANDARD_API(originate_function)
{
    switch_channel_t *caller_channel;
    switch_core_session_t *caller_session = NULL;
    char *mycmd = NULL, *argv[10] = { 0 };
    int i = 0, x, argc = 0;
    char *aleg, *exten, *dp, *context, *cid_name, *cid_num;
    uint32_t timeout = 60;
    switch_call_cause_t cause = SWITCH_CAUSE_NORMAL_CLEARING;
    switch_status_t status = SWITCH_STATUS_SUCCESS;

    if (zstr(cmd)) {
        stream->write_function(stream, "-USAGE: %s\n", ORIGINATE_SYNTAX);
        return SWITCH_STATUS_SUCCESS;
    }

    /* log warning if part of ongoing session, as we'll block the session */
    if (session){
        switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(session), 　　　　　　SWITCH_LOG_NOTICE, 　　　　　　"Originate can take 60 seconds to complete, and blocks the existing session. Do not confuse with a lockup.\n");
    }

    mycmd = strdup(cmd);
    switch_assert(mycmd);
    argc = switch_separate_string(mycmd, ' ', argv, (sizeof(argv) / sizeof(argv[0])));

    if (argc < 2 || argc > 7) {
        stream->write_function(stream, "-USAGE: %s\n", ORIGINATE_SYNTAX);
        goto done;
    }

    for (x = 0; x < argc && argv[x]; x++) {
        if (!strcasecmp(argv[x], "undef")) {
            argv[x] = NULL;
        }
    }

    aleg = argv[i++];
    exten = argv[i++];
    dp = argv[i++];
    context = argv[i++];
    cid_name = argv[i++];
    cid_num = argv[i++];

    switch_assert(exten);

    if (!dp) {
        dp = "XML";
    }

    if (!context) {
        context = "default";
    }

    if (argv[6]) {
        timeout = atoi(argv[6]);
    }

    if (switch_ivr_originate(NULL, &caller_session, &cause, aleg, timeout, NULL, cid_name, cid_num, NULL, NULL, SOF_NONE, NULL, NULL) 　　　　 != SWITCH_STATUS_SUCCESS
        || !caller_session) {
            stream->write_function(stream, "-ERR %s\n", switch_channel_cause2str(cause));
        goto done;
    }

    caller_channel = switch_core_session_get_channel(caller_session);

    if (*exten == '&' && *(exten + 1)) {
        switch_caller_extension_t *extension = NULL;
        char *app_name = switch_core_session_strdup(caller_session, (exten + 1));
        char *arg = NULL, *e;

        if ((e = strchr(app_name, ')'))) {
            *e = '\0';
        }

        if ((arg = strchr(app_name, '('))) {
            *arg++ = '\0';
        }

        if ((extension = switch_caller_extension_new(caller_session, app_name, arg)) == 0) {
            switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(session), SWITCH_LOG_CRIT, "Memory Error!\n");
            abort();
        }
        switch_caller_extension_add_application(caller_session, extension, app_name, arg);
        switch_channel_set_caller_extension(caller_channel, extension);
        switch_channel_set_state(caller_channel, CS_EXECUTE);
    } else {
        switch_ivr_session_transfer(caller_session, exten, dp, context);
    }

    stream->write_function(stream, "+OK %s\n", switch_core_session_get_uuid(caller_session));

    switch_core_session_rwunlock(caller_session);

  done:
    switch_safe_free(mycmd);
    return status;
}
```
调用流程如下：
```
originate_function 
    => switch_ivr_originate 
        => switch_core_session_outgoing_channel 
            => endpoint_interface->io_routines->outgoing_channel
 　　　　=> switch_core_session_thread_launch
```
#### 7.1.3 switch_ivr_originate函数
该函数用于发起具体的呼叫。switch_ivr_originate函数定义：
```
SWITCH_DECLARE(switch_status_t) switch_ivr_originate(
    switch_core_session_t *session,                          //发起originate的channel，即 caller_channel , aleg
    switch_core_session_t **bleg,                            //originate所在的leg，会在该函数赋值
    switch_call_cause_t *cause,                                //失败原因，会在该函数赋值
    const char *bridgeto,                                    //bleg的呼叫字符串，只读
    uint32_t timelimit_sec,                                    //originate超时时间
    const switch_state_handler_table_t *table,                //bleg的状态机回调函数
    const char *cid_name_override,                            //origination_caller_id_name，用于设置主叫名称
    const char *cid_num_override,                            //origination_caller_id_number，用于设置主叫号码
    switch_caller_profile_t *caller_profile_override,        //主叫的profile
    switch_event_t *ovars, switch_originate_flag_t flags,    //originate导出的通道变量（从aleg）
    switch_call_cause_t *cancel_cause,                        // originate取消原因
    switch_dial_handle_t *dh)                                // dial handle,功能类似呼叫字符串，可以设置多条leg同时originate
```
如果outgoing_channel执行成功，会在函数switch_core_session_outgoing_channel()中发送SWITCH_EVENT_CHANNEL_OUTGOING事件；并且该channel会设置上CF_ORIGINATING标识位。
```
if (switch_event_create(&event, SWITCH_EVENT_CHANNEL_OUTGOING) == SWITCH_STATUS_SUCCESS) {
    switch_channel_event_set_data(peer_channel, event);
    switch_event_fire(&event);
}
```
使用switch_core_session_thread_launch()启动线程创建session ：
```
if (!switch_core_session_running(oglobals.originate_status[i].peer_session)) {
    if (oglobals.originate_status[i].per_channel_delay_start) {
        switch_channel_set_flag(oglobals.originate_status[i].peer_channel, CF_BLOCK_STATE);
    }
    switch_core_session_thread_launch(oglobals.originate_status[i].peer_session);
}
```
### 7.2 bridge流程
#### 7.2.1 流程入口
bridge app入口(mod_dptools.c)：

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/ab4705b1-040d-4b0d-9504-972f1fd87434)

函数调用链：

```
audio_bridge_function 
    => switch_ivr_signal_bridge
or  => switch_ivr_multi_threaded_bridge 
```
uuid_bridge api入口（mod_commands.c）：

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/324b64ba-d390-40b6-bfa7-aa81ae4f68b1)

函数调用链：
```
uuid_bridge_function => switch_ivr_uuid_bridge
```
#### 7.2.2 bridge机制
注册回调函数：
```
static const switch_state_handler_table_t audio_bridge_peer_state_handlers = {
    /*.on_init */ NULL,
    /*.on_routing */ audio_bridge_on_routing,
    /*.on_execute */ NULL,
    /*.on_hangup */ NULL,
    /*.on_exchange_media */ audio_bridge_on_exchange_media,
    /*.on_soft_execute */ NULL,
    /*.on_consume_media */ audio_bridge_on_consume_media,
};
```
 状态机里面进行回调, 当channel进入CS_EXCHANGE_MEDIA状态后，回调 audio_bridge_on_exchange_media 函数，触发audio_bridge_thread线程。
### 7.3 媒体交互流程
#### 7.3.1 注册编码类型
在函数mod_spandsp_codecs_load()中调用switch_core_codec_add_implementation()函数注册音频编解码。

例如添加PCMU编码
```
switch_core_codec_add_implementation(pool, codec_interface, SWITCH_CODEC_TYPE_AUDIO,    /* enumeration defining the type of the codec */
                                             0,    /* the IANA code number */
                                             "PCMU",    /* the IANA code name */
                                             NULL,    /* default fmtp to send (can be overridden by the init function) */
                                             8000,    /* samples transferred per second */
                                             8000,    /* actual samples transferred per second */
                                             64000,    /* bits transferred per second */
                                             mpf * count,    /* number of microseconds per frame */
                                             spf * count,    /* number of samples per frame */
                                             bpf * count,    /* number of bytes per frame decompressed */
                                             ebpf * count,    /* number of bytes per frame compressed */
                                             1,    /* number of channels represented */
                                             spf * count,    /* number of frames per network packet */
                                             switch_g711u_init,    /* function to initialize a codec handle using this implementation */
                                             switch_g711u_encode,    /* function to encode raw data into encoded data */
                                             switch_g711u_decode,    /* function to decode encoded data into raw data */
                                             switch_g711u_destroy);    /* deinitalize a codec handle using this implementation */
```
#### 7.3.2 RTP数据交互及转码 
函数调用链：
```
audio_bridge_on_exchange_media => audio_bridge_thread
```
收发音频数据：
```
audio_bridge_thread 
    => switch_core_session_read_frame
         => need_codec
         => switch_core_codec_decode （调用implement的encode进行转码操作，比如 switch_g711a_decode)
     => session->endpoint_interface->io_routines->read_frame 即： sofia_read_frame
         => switch_core_media_read_frame
       　　 => switch_rtp_zerocopy_read_frame
            => rtp_common_read
            　　=> read_rtp_packet
                  => switch_socket_recvfrom


audio_bridge_thread 
    => switch_core_session_write_frame
         => switch_core_session_start_audio_write_thread (ptime不一致时启动线程，有500长度的队列)
          => switch_core_codec_encode （调用implement的encode进行转码操作，比如 switch_g711u_encode)
     => perform_write
        => session->endpoint_interface->io_routines->write_frame 比如： sofia_write_frame
        => switch_core_media_write_frame
            => switch_rtp_write_frame
            　　=>switch_socket_sendto
                =>rtp_common_write
```
## 8 FreeSWITCH内核与Sofia模块协同工作流程分析
FreeSWITCH 调用SIP协议栈实现SIP协议交换层级如下：

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/6a349cb3-bc6c-4b6a-9741-274ad44cba4e)
### 8.1 FreeSWITCH Core
#### 8.1.1 模块加载过程
freeswitch主程序初始化时会从modules.conf.xml文件中读取配置，如果配置中如下内容生效，FreeSWITCH启动时则执行加载sofia模块操作。
```
<load module="mod_sofia"/>
```
具体过程如下：
```
main()
    =>switch_core_init_and_modload()
        =>switch_loadable_module_init()
            =>switch_loadable_module_load_module_ex()// 读取xml文件并加载模块
```
#### 8.1.2 状态机
##### 8.1.2.1 状态机初始化
在如下函数中实现状态机初始化
```
switch_core_session_run()
```
##### 8.1.2.2  改变状态机状态
通过调用switch_channel_set_state来实现状态机的状态改变。

##### 8.1.2.3 处理状态变化
当状态发生变化后，通过switch_channel_set_running_state函数来改变running_state，并执行相关的回调来通知其状态已经发生改变：
```
endpoint_interface->io_routines->state_run()
```
**主叫变化**
- CS_NEW

switch_core_session_run初始状态为CS_NEW
- CS_INIT

sofia_handle_sip_i_state

case nua_callstate_received (收到invite请求) 修改状态机的状态 ：CS_NEW ==> CS_INIT switch_channel_set_state(channel, CS_INIT);

switch_core_session_run

状态机处理状态变化 STATE_MACRO(init, "INIT"); on_init 即 ： sofia_on_init

switch_core_standard_on_init
- CS_ROUTING

sofia_on_init

修改状态机的状态 ： CS_INIT == > CS_ROUTING

switch_channel_set_state(channel, CS_ROUTING);

switch_core_session_run

状态机处理状态变化 STATE_MACRO(routing, "ROUTING"); on_routing 即 ： sofia_on_routing

switch_core_standard_on_routing

- CS_EXECUTE

witch_core_standard_on_routing 修改状态机的状态 ： CS_ROUTING == > CS_EXECUTE switch_channel_set_state(session->channel, CS_EXECUTE);

switch_core_session_run

状态机处理状态变化 STATE_MACRO(execute, "EXECUTE"); on_execute 即 ： sofia_on_execute

switch_core_standard_on_execute

- CS_HANGUP

sofia_handle_sip_i_bye switch_channel_hangup ： 即 switch_channel_perform_hangup 修改状态机的状态 channel->state = CS_HANGUP;
```
状态机处理状态变化
    switch_core_session_hangup_state            
        STATE_MACRO(hangup, "HANGUP");
```

**被叫变化**

- CS_NEW

switch_core_session_run初始状态为CS_NEW

- CS_INIT

sofia_outgoing_channel 修改状态机的状态 ：CS_NEW ==> CS_INIT switch_channel_set_state(nchannel, CS_INIT);

状态机处理逻辑参考主叫。

- CS_ROUTING

sofia_on_init

修改状态机的状态 ： CS_INIT == > CS_ROUTING switch_channel_set_state(channel, CS_ROUTING);

状态机处理逻辑参考主叫。

- CS_CONSUME_MEDIA

originate_on_routing 修改状态机的状态：CS_ROUTING -> CS_CONSUME_MEDIA switch_channel_set_state(channel, CS_CONSUME_MEDIA);

switch_core_session_run

状态机处理状态变化 STATE_MACRO(consume_media, "CONSUME_MEDIA"); on_ consume_media 即 ： switch_core_standard_on_consume_media

- CS_EXCHANGE_MEDIA

switch_core_session_run

状态机处理状态变化 STATE_MACRO(exchange_media, "EXCHANGE_MEDIA"); on_exchange_media 即 ： sofia_on_exchange_media

switch_core_standard_on_exchange_media

- CS_HANGUP

audio_bridge_on_exchange_media

switch_channel_hangup ： 即 switch_channel_perform_hangup 修改状态机的状态 channel->state = CS_HANGUP;

```
状态机处理状态变化
    switch_core_session_hangup_state
        STATE_MACRO(hangup, "HANGUP");
```
### 8.2 Sofia应用层
模块结构：

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/55abc5fc-c99d-4bfc-9f05-d9a2fb41e938)

#### 8.2.1 模块加载过程
启动事件处理线程池 ：
```
SWITCH_MODULE_LOAD_FUNCTION(mod_sofia_load)
    => sofia_msg_thread_start(0);
        => sofia_msg_thread_run
            => sofia_process_dispatch_event
                => our_sofia_event_callback : 处理消息
```
启动服务器监听 ：
```
SWITCH_MODULE_LOAD_FUNCTION(mod_sofia_load)
    => config_sofia(0, NULL)
        => launch_sofia_profile_thread
            => sofia_profile_thread_run                
                => nua_create
                    => su_home_new
                    => nua_stack_init                    
                        => nta_agent_create
                            => nta_agent_add_tport                            
                                => tport_tbind
                                    => tport_bind_server
                                        => tport_listen ： 监听客户端发来的数据
```
#### 8.2.2 数据库表格
freeswitch core 相关：
- aliases
- calls
- channels
- complete
- interfaces
- nat
- recovery
- registrations
- tasks

sofia相关：
- sip_authentication
- sip_dialogs
- sip_presence
- sip_registrations
- sip_shared_appearance_dialogs
- sip_shared_appearance_subscriptions
- sip_subscriptions

limit相关：
- db_data
- group_data
- limit_data

fifo相关：
- fifo_bridge
- fifo_callers
- fifo_outbound

语音信箱相关：
- voicemail_msgs
- voicemail_prefs

#### 8.2.3 呼叫流程涉及内容
- 收到A的nua_i_invite，返回407，具体如下:
```
sofia_handle_sip_i_invite => sofia_reg_handle_register => sofia_reg_auth_challenge => 407
```
- 收到A的nua_i_invite，返回180，具体如下:
```
sofia_handle_sip_i_invite => sofia_reg_handle_register => sofia_reg_parse_auth
```
处理nua_i_state消息
sofia.c : sofia_handle_sip_i_state

mod_dialplan_xml.c : dialplan_hunt (ring_ready)
解析拨号方案，执行lua脚本，设置通道变量。

发送180事件

涉及数据表：

sip_registrations,ip_dialogs

- 发送invite给B分机，具体如下：
处理nua_r_invite消息

sofia_handle_sip_r_invite => originate_on_routing

涉及数据表：

sip_dialogs

- B应答，双方通话，具体如下

```
sofia_receive_message : SWITCH_MESSAGE_INDICATE_ANSWER               
    => sofia_answer_channel
        => sofia_glue_tech_choose_port
        => sofia_glue_set_local_sdp
        => sofia_glue_activate_rtp
```

#### 8.2.4 编码协商过程
- 先匹配freeswitch默认的codec，从加载的模块中查找支持类型的具体码率信息
```
sofia_handle_sip_i_invite
=> sofia_glue_tech_prepare_codecs
=> switch_loadable_module_get_codecs_sorted
```
- 协商发给被叫的sdp
```
sofia_glue_do_invite
=> sofia_glue_tech_prepare_codecs
=> switch_loadable_module_get_codecs_sorted
```
```
 ocodec + codec_string
```
- 产生M头，并发送包含sdp的invite
```
sofia_glue_do_invite
=> sofia_glue_set_local_sdp
=> generate_m
=> nua_invite
```
- 协商200 OK的sdp
```
sofia_answer_channel
=> sofia_glue_tech_prepare_codecs
=> switch_loadable_module_get_codecs_sorted
```
#### 8.2.5 媒体交互
- RTP数据交互
```
audio_bridge_on_exchange_media
=> audio_bridge_thread
```
收发音频数据
```
=> switch_core_session_read_frame
=> session->endpoint_interface->io_routines->read_frame 即： sofia_read_frame

=> switch_core_session_write_frame
=> perform_write
=> session->endpoint_interface->io_routines->write_frame 即： sofia_write_frame
```
收发视频数据

1、启动线程
```
=> launch_video
=> video_bridge_thread
```

2、收发数据
```
=> switch_core_session_read_video_frame
=>session->endpoint_interface->io_routines->read_video_frame
即：sofia_read_video_frame

=> switch_core_session_write_video_frame
=> session->endpoint_interface->io_routines->write_video_frame
即：sofia_write_video_frame
```
- 终止RTP交互
```
audio_bridge_thread
=> switch_core_session_kill_channel(session_b, SWITCH_SIG_BREAK);
即 ：switch_core_session_perform_kill_channel
=> session->endpoint_interface->io_routines->kill_channel
即 : sofia_kill_channel
=> switch_rtp_break
```
### 8.3 sofia协议栈
#### 8.3.1 协议栈结构图

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/6512c1b7-488b-4731-8df8-23de148387eb)

NUA : 基本的SIP用户代理的功能，其功能包括呼叫管理，消息和事件检索。

NTA : sofia SIP事务API（NTA）提供了简单的接口的SIP事务，传输和信息处理。

Tport : 该模块包含一个由SIP，RTSP 和HTTP协议使用的通用传输接口，这是一个协议栈和传输协议实现之间的抽象层。

#### 8.3.2 信令流程

![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/5def601f-50c6-4575-a29c-25f8826ca6e3)

 场景描述： A 呼叫 B ，B接通后通话一段时间后，挂断电话。
 
- A 发送 INVITE 请求
```
tport.c : tport_recv_data
nta.c : agent_recv_request
```
- FS回应100给A分机
```
nua_server.c : nua_stack_process_request
nua_server.c : SR_STATUS1(sr, SIP_100_TRYING)
nta.c : nta_incoming_treply
nta.c : incoming_reply
tport.c: tport_tsend
```
- FS发送认证请求给A分机
```
nua_application_event

处理nua_i_invite 消息

sofia.c : sofia_process_dispatch_event
sofia.c : our_sofia_event_callback
sofia.c : sofia_handle_sip_i_invite
sofia.c : sofia_reg_handle_register
sofia_reg.c : sofia_reg_auth_challenge

回复407

nua.c : nua_respond
nua_stack.c : nua_signal
nua_stack.c : nua_stack_signal
nua_server.c : nua_stack_respond
nua_server.c : nua_server_respond

发送407给A分机

nua_invite_server_respond
nua_base_server_respond
nta_incoming_mreply
incoming_reply
tport_tsend
tport_resolve
tport_by_addrinfo
tport_prepare_and_send
tport_send_msg
tport_vsend
sent 407 Proxy Authentication Required for INVITE (1)

call state(A) : init -> received
call state(A) : received -> terminated
```
- A分机发送ACK
```
nta.c : agent_recv_message
nta.c : agent_recv_request
```
- A分机重新发送 INVITE 请求，附加认证信息
```
tport.c : tport_recv_data
nta.c : agent_recv_request
```
- FS回应100给A分机
```
nua_server.c : nua_stack_process_request
nua_server.c : SR_STATUS1(sr, SIP_100_TRYING)
nta.c : nta_incoming_treply
nta.c : incoming_reply
tport.c: tport_tsend
```
- FS发送 INVITE 给 B分机
```
switch_ivr_originate.c : switch_ivr_originate

sofia_on_init
sofia_glue_do_invite
nua_invite : nua_r_invite 消息
nua_signal

sofia.c : sofia_process_dispatch_event
sofia.c : our_sofia_event_callback
sofia.c : sofia_handle_sip_r_invite

sofia_on_init
sofia_on_routing
switch_ivr_originate.c ：originate_on_routing

nua_stack_signal
nua_stack_invite

nta_leg_tcreate


执行发送invite请求
nta.c : outgoing_send
tport_tsend

call state(B) : init -> calling
```
- B分机回应180给FS
```
tport_recv_event
agent_recv_response

处理 nua_r_invite 消息
sofia.c : sofia_process_dispatch_event
sofia.c : our_sofia_event_callback
sofia.c : sofia_handle_sip_r_invite

call state(B) : calling -> proceeding
```
- FS发送180给A分机
```
nua_session.c : signal_call_state_change
nua_stack_tevent : nua_i_state
nua_application_event

处理nua_i_invite 消息

sofia.c : sofia_process_dispatch_event
sofia.c : our_sofia_event_callback
sofia.c : sofia_handle_sip_i_invite
sofia_reg.c : sofia_reg_handle_register
sofia_reg.c : sofia_reg_parse_auth

处理nua_i_state消息
sofia.c : sofia_handle_sip_i_state


mod_dialplan_xml.c : dialplan_hunt (ring_ready)
解析拨号方案，执行lua脚本，设置通道变量。

发送180
switch_channel_mark_ring_ready

nua_invite_server_respond
nua_base_server_respond
nta_incoming_mreply
incoming_reply
tport_tsend
tport_resolve
tport_by_addrinfo
tport_prepare_and_send
tport_send_msg
tport_vsend

call state(A) : init -> received
```
- B发送200 ok
```
tport_recv_event
agent_recv_response

处理 nua_r_invite 消息

sofia.c : sofia_process_dispatch_event
sofia.c : our_sofia_event_callback
sofia.c : sofia_handle_sip_r_invite

call state(B) : proceeding -> completing
```
- FS发送ACK给B
```
nua_ack
nua_signal : nua_r_ack
outgoing_send
tport_tsend
call state(A) : received -> early
call state(B) : completing -> ready
```
- FS发送200 OK给A

```
sofia_glue_tech_set_codec
switch_rtp_create
sofia_glue_negotiate_sdp
sofia_glue_activate_rtp

switch_channel_perform_answer : SWITCH_MESSAGE_INDICATE_ANSWER
switch_channel_perform_mark_answered

auto_record.lua ： 执行录音操作B
auto_record.lua ： 执行录音操作A

sofia_receive_message (SWITCH_MESSAGE_INDICATE_ANSWER)

发送200 OK
nua_respond : 200
nua_signal : nua_r_respond
nua_stack_signal
nua_stack_respond
nua_server_respond
nta_incoming_treply
nta_incoming_mreply
incoming_reply
tport_tsend

call state(A) : early -> completed
```
- A回应ACK
```
nta.c : agent_recv_message
nta.c : agent_recv_request
call state(A) : completed -> ready
```
- 通话一段时间后，A主动发送BYE
```
tport_recv_event
agent_recv_message
agent_recv_request
sofia_on_hangup
```
- FS发送BYE给B
```
nua_bye : nua_r_bye
nua_stack_signal
outgoing_send
tport_tsend
```
- FS发送200 OK给A
```
nta ： sent 200 OK for BYE
tport_tsend

call state(A) : ready -> terminating
call state(A) : terminated
```
- B发送200 OK给FS，回应挂断请求
```
nta: received 200 OK for BYE
状态变化：
call state(B) : ready -> terminating
call state(B) : terminating -> terminated
```

