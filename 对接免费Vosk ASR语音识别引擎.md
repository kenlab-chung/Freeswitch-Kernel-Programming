# FreeSWITCH对接Vosk 免费ASR语音识别引擎
## 1 Vosk简介
Vosk是一个离线开源语音识别工具，它可以识别16中语言，包含中文。总体效果还是不错的，因为我们要对接到呼叫中心，因此我们需要实时的流式传输语音数据，目前主流方案就是采用websocket协议传输语音，Vosk直接提供了Websocket的Server程序。而且程序已经打包成docker发布，所以启动起来特别简单：
```
docker run -d -p 2700:2700 alphacep/kaldi-cn:latest
```
## 在FreeSWITCH中实现与Vosk对接
本文使用FreeSWITCH插件模式实现ASR对接。FreeSWITCH中已经定义好了asr识别接口，所以只要根据接口做实现即可。主要要实现的接口如下：
```
	asr_interface = switch_loadable_module_create_interface(*module_interface, SWITCH_ASR_INTERFACE);
	asr_interface->interface_name = "vosk";
	asr_interface->asr_open = vosk_asr_open;
	asr_interface->asr_close = vosk_asr_close;
	asr_interface->asr_load_grammar = vosk_asr_load_grammar;
	asr_interface->asr_unload_grammar = vosk_asr_unload_grammar;
	asr_interface->asr_resume = vosk_asr_resume;
	asr_interface->asr_pause = vosk_asr_pause;
	asr_interface->asr_feed = vosk_asr_feed;
	asr_interface->asr_check_results = vosk_asr_check_results;
	asr_interface->asr_get_results = vosk_asr_get_results;
	asr_interface->asr_start_input_timers = vosk_asr_start_input_timers;
```
实现源码如下
```
# filename mod_vosk.c
#define __PRETTY_FUNCTION__ __func__

#include <switch.h>
#include <netinet/tcp.h>
#include "libks/ks.h"


#define AUDIO_BLOCK_SIZE 3200

SWITCH_MODULE_LOAD_FUNCTION(mod_vosk_load);
SWITCH_MODULE_SHUTDOWN_FUNCTION(mod_vosk_shutdown);
SWITCH_MODULE_DEFINITION(mod_vosk, mod_vosk_load, mod_vosk_shutdown, NULL);

static switch_mutex_t *MUTEX = NULL;
static switch_event_node_t *NODE = NULL;

static struct {
	char *server_url;
	int return_json;

	int auto_reload;
	switch_memory_pool_t *pool;
	ks_pool_t *ks_pool;
} globals;


typedef struct {
	kws_t *ws;
	char *result;
	switch_mutex_t *mutex;
	switch_buffer_t *audio_buffer;
} vosk_t;

/*! function to open the asr interface */
static switch_status_t vosk_asr_open(switch_asr_handle_t *ah, const char *codec, int rate, const char *dest, switch_asr_flag_t *flags)
{
	vosk_t *vosk;
	ks_json_t *req = ks_json_create_object();
	ks_json_add_string_to_object(req, "url", (dest ? dest : globals.server_url));

	if (!(vosk = (vosk_t *) switch_core_alloc(ah->memory_pool, sizeof(*vosk)))) {
		return SWITCH_STATUS_MEMERR;
	}
	ah->private_info = vosk;
	switch_mutex_init(&vosk->mutex, SWITCH_MUTEX_NESTED, ah->memory_pool);

	if (switch_buffer_create_dynamic(&vosk->audio_buffer, AUDIO_BLOCK_SIZE, AUDIO_BLOCK_SIZE, 0) != SWITCH_STATUS_SUCCESS) {
		switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_INFO, "Buffer create failed\n");
		return SWITCH_STATUS_MEMERR;
	}

	codec = "L16";
	ah->codec = switch_core_strdup(ah->memory_pool, codec);

	if (kws_connect_ex(&vosk->ws, req, KWS_BLOCK | KWS_CLOSE_SOCK, globals.ks_pool, NULL, 30000) != KS_STATUS_SUCCESS) {
		ks_json_delete(&req);
		switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_INFO, "Websocket connect to %s failed\n", globals.server_url);
		return SWITCH_STATUS_GENERR;
	}
	ks_json_delete(&req);
	switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_INFO, "ASR open\n");

	return SWITCH_STATUS_SUCCESS;
}

/*! function to close the asr interface */
static switch_status_t vosk_asr_close(switch_asr_handle_t *ah, switch_asr_flag_t *flags)
{
	vosk_t *vosk = (vosk_t *) ah->private_info;

	switch_mutex_lock(vosk->mutex);
	switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_INFO, "ASR closed\n");

	/** FIXME: websockets server still expects us to read the close confirmation and only then close
	    libks library doens't implement it yet. */
	kws_close(vosk->ws, KWS_CLOSE_SOCK);
	kws_destroy(&vosk->ws);

	switch_set_flag(ah, SWITCH_ASR_FLAG_CLOSED);
	switch_buffer_destroy(&vosk->audio_buffer);
	switch_safe_free(vosk->result);
	switch_mutex_unlock(vosk->mutex);

	return SWITCH_STATUS_SUCCESS;
}

/*! function to feed audio to the ASR */
static switch_status_t vosk_asr_feed(switch_asr_handle_t *ah, void *data, unsigned int len, switch_asr_flag_t *flags)
{
	int poll_result;
	kws_opcode_t oc;
	uint8_t *rdata;
	int rlen;
	vosk_t *vosk = (vosk_t *) ah->private_info;

	if (switch_test_flag(ah, SWITCH_ASR_FLAG_CLOSED))
		return SWITCH_STATUS_BREAK;

	switch_mutex_lock(vosk->mutex);

	switch_buffer_write(vosk->audio_buffer, data, len);
	if (switch_buffer_inuse(vosk->audio_buffer) > AUDIO_BLOCK_SIZE) {
		char buf[AUDIO_BLOCK_SIZE];
		int rlen;

		rlen = switch_buffer_read(vosk->audio_buffer, buf, AUDIO_BLOCK_SIZE);
		switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_DEBUG, "Sending data %d\n", rlen);
		if (kws_write_frame(vosk->ws, WSOC_BINARY, buf, rlen) < 0) {
			switch_mutex_unlock(vosk->mutex);
			return SWITCH_STATUS_BREAK;
		}
	}

	poll_result = kws_wait_sock(vosk->ws, 0, KS_POLL_READ | KS_POLL_ERROR);

	if (poll_result != KS_POLL_READ) {
		//switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_DEBUG, "Received Poll Failed\n");
		switch_mutex_unlock(vosk->mutex);
		return SWITCH_STATUS_SUCCESS;
	}
	rlen = kws_read_frame(vosk->ws, &oc, &rdata);
	if (rlen < 0) {
		switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_DEBUG, "Received Read Failed\n");
		switch_mutex_unlock(vosk->mutex);
		return SWITCH_STATUS_BREAK;
	}
	if (oc == WSOC_PING) {
		switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_DEBUG, "Received ping\n");
		kws_write_frame(vosk->ws, WSOC_PONG, rdata, rlen);
		switch_mutex_unlock(vosk->mutex);
		return SWITCH_STATUS_SUCCESS;
	}

	switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_DEBUG, "Recieved %d bytes:\n%s\n", rlen, rdata);
	switch_safe_free(vosk->result);
	vosk->result = switch_safe_strdup((const char *)rdata);
	switch_mutex_unlock(vosk->mutex);

	return SWITCH_STATUS_SUCCESS;
}

/*! function to pause recognizer */
static switch_status_t vosk_asr_pause(switch_asr_handle_t *ah)
{
	return SWITCH_STATUS_SUCCESS;
}

/*! function to resume recognizer */
static switch_status_t vosk_asr_resume(switch_asr_handle_t *ah)
{
	return SWITCH_STATUS_SUCCESS;
}

/*! Process asr_load_grammar request from FreeSWITCH. */
static switch_status_t vosk_asr_load_grammar(switch_asr_handle_t *ah, const char *grammar, const char *name)
{
	return SWITCH_STATUS_SUCCESS;
}

/*! Process asr_unload_grammar request from FreeSWITCH. */
static switch_status_t vosk_asr_unload_grammar(switch_asr_handle_t *ah, const char *name)
{
	return SWITCH_STATUS_SUCCESS;
}


/*! function to read results from the ASR*/
static switch_status_t vosk_asr_check_results(switch_asr_handle_t *ah, switch_asr_flag_t *flags)
{
	vosk_t *vosk = (vosk_t *) ah->private_info;
	return (vosk->result && (strstr(vosk->result, "\"\"") == NULL)) ? SWITCH_STATUS_SUCCESS : SWITCH_STATUS_FALSE;
}

/*! function to read results from the ASR */
static switch_status_t vosk_asr_get_results(switch_asr_handle_t *ah, char **xmlstr, switch_asr_flag_t *flags)
{
	vosk_t *vosk = (vosk_t *) ah->private_info;
	switch_status_t ret;


	switch_mutex_lock(vosk->mutex);
	if (globals.return_json) {
		if  (strstr(vosk->result, "\"partial\"") == NULL) {
			*xmlstr = switch_safe_strdup(vosk->result);
			ret = SWITCH_STATUS_SUCCESS;
		} else {
			*xmlstr = switch_safe_strdup(vosk->result);
			ret = SWITCH_STATUS_MORE_DATA;
		}
	} else {
		cJSON *result = cJSON_Parse(vosk->result);

		if (cJSON_HasObjectItem(result, "text")) {
			*xmlstr = switch_safe_strdup(cJSON_GetObjectCstr(result, "text"));
			ret = SWITCH_STATUS_SUCCESS;
		} else if (cJSON_HasObjectItem(result, "partial")) {
			*xmlstr = switch_safe_strdup(cJSON_GetObjectCstr(result, "partial"));
			ret = SWITCH_STATUS_MORE_DATA;
		} else {
			ret = SWITCH_STATUS_GENERR;
		}
		cJSON_Delete(result);
	}

	switch_safe_free(vosk->result);
	vosk->result = NULL;
	switch_mutex_unlock(vosk->mutex);

	return ret;
}

/*! function to start input timeouts */
static switch_status_t vosk_asr_start_input_timers(switch_asr_handle_t *ah)
{
	return SWITCH_STATUS_SUCCESS;
}

static switch_status_t load_config(void)
{
	char *cf = "vosk.conf";
	switch_xml_t cfg, xml = NULL, param, settings;
	switch_status_t status = SWITCH_STATUS_SUCCESS;

	if (!(xml = switch_xml_open_cfg(cf, &cfg, NULL))) {
		switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_ERROR, "Open of %s failed\n", cf);
		status = SWITCH_STATUS_FALSE;
		goto done;
	}


	if ((settings = switch_xml_child(cfg, "settings"))) {
		for (param = switch_xml_child(settings, "param"); param; param = param->next) {
			char *var = (char *) switch_xml_attr_soft(param, "name");
			char *val = (char *) switch_xml_attr_soft(param, "value");
			if (!strcasecmp(var, "server-url")) {
				globals.server_url = switch_core_strdup(globals.pool, val);
			}
			if (!strcasecmp(var, "return-json")) {
				globals.return_json = atoi(val);
			}
		}
	}

  done:
	if (!globals.server_url) {
		globals.server_url = switch_core_strdup(globals.pool, "ws://127.0.0.1:2700");
	}
	if (xml) {
		switch_xml_free(xml);
	}

	return status;
}

static void do_load(void)
{
	switch_mutex_lock(MUTEX);
	load_config();
	switch_mutex_unlock(MUTEX);
}

static void event_handler(switch_event_t *event)
{
	if (globals.auto_reload) {
		do_load();
		switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_INFO, "Vosk Reloaded\n");
	}
}

SWITCH_MODULE_LOAD_FUNCTION(mod_vosk_load)
{
	switch_asr_interface_t *asr_interface;

	switch_mutex_init(&MUTEX, SWITCH_MUTEX_NESTED, pool);

	globals.pool = pool;

	ks_init();

	ks_pool_open(&globals.ks_pool);
	ks_global_set_default_logger(7);

	if ((switch_event_bind_removable(modname, SWITCH_EVENT_RELOADXML, NULL, event_handler, NULL, &NODE) != SWITCH_STATUS_SUCCESS)) {
		switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_ERROR, "Couldn't bind!\n");
	}

	do_load();

	/* connect my internal structure to the blank pointer passed to me */
	*module_interface = switch_loadable_module_create_module_interface(pool, modname);

	asr_interface = switch_loadable_module_create_interface(*module_interface, SWITCH_ASR_INTERFACE);
	asr_interface->interface_name = "vosk";
	asr_interface->asr_open = vosk_asr_open;
	asr_interface->asr_close = vosk_asr_close;
	asr_interface->asr_load_grammar = vosk_asr_load_grammar;
	asr_interface->asr_unload_grammar = vosk_asr_unload_grammar;
	asr_interface->asr_resume = vosk_asr_resume;
	asr_interface->asr_pause = vosk_asr_pause;
	asr_interface->asr_feed = vosk_asr_feed;
	asr_interface->asr_check_results = vosk_asr_check_results;
	asr_interface->asr_get_results = vosk_asr_get_results;
	asr_interface->asr_start_input_timers = vosk_asr_start_input_timers;

	/* indicate that the module should continue to be loaded */
	return SWITCH_STATUS_SUCCESS;
}

SWITCH_MODULE_SHUTDOWN_FUNCTION(mod_vosk_shutdown)
{
	ks_pool_close(&globals.ks_pool);
	ks_shutdown();

	switch_event_unbind(&NODE);
	return SWITCH_STATUS_UNLOAD;
}

```
配置文件：vosk.conf.xml
```
<configuration name="vosk.conf" description="Vosk ASR Configuration">
  <settings>
    <param name="server-url" value="ws://localhost:2700"/>
    <param name="return-json" value="0"/>
  </settings>
</configuration>
```
注册一个分机测试，配置一个Dialplan如下
```
    <extension name="vosk_asr">
      <condition field="destination_number" expression="^123456$">
		<action application="answer"/>
		<action application="set" data="fire_asr_events=true"/>
		<action application="detect_speech" data="vosk default default"/>
		<action application="park" data=""/>
      </condition>
    </extension>
```
后续参考文件：
配置文件：conf/default.xml
```
<include>
  <context name="default">
    <extension name="asr_demo">
        <condition field="destination_number" expression="^.*$">
          <action application="answer"/>
          <action application="play_and_detect_speech" data="ivr/ivr-welcome.wav detect:vosk default"/>
          <action application="speak" data="tts_commandline|espeak|You said ${detect_speech_result}!"/>
        </condition>
    </extension>
  </context>
</include>
```
效果图：
![image](https://github.com/kenlab-chung/Freeswitch-Kernel-Programming/assets/59462735/e03dcb2b-4ef8-4130-9700-0c3cd167e618)

配置文件：conf/default.esl.xml
```
<include>
  <context name="default">
    <extension name="asr_demo">
        <condition field="destination_number" expression="^.*$">
          <action application="answer"/>
          <action application="set" data="fire_asr_events=true"/>
          <action application="detect_speech" data="vosk default default"/>
          <action application="sleep" data="10000000"/>
        </condition>
    </extension>
  </context>
</include>
```
