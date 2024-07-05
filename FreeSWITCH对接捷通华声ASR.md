# FreeSWITCH对接捷通华声ASR
## 1 开启freeswitch的unimrcp模块
`vim /usr/local/freeswitch/conf/autoload_configs/modules.conf.xml `将<load module="mod_enum"/>注释掉
```
<!--load module="mod_enum"/>-->
在<modules>子标签中增加：
 <load module="mod_unimrcp"/>
```
## 2 修改freeswitch允许java远程Esl访问
`vim /usr/local/freeswitch/conf/autoload_configs/event_socket.conf.xml`
```
<configuration name="event_socket.conf" description="Socket Client">?
    <settings>
        <param name="nat-map" value="false"/>
        <param name="listen-ip" value="0.0.0.0"/>
        <param name="listen-port" value="8021"/>
        <param name="password" value="ClueCon"/>
        <param name="apply-inbound-acl" value="lan"/>
    </settings>
</configuration>
```
## 3 修改freeswitch中unimrcp连接asr和tts配置文件unimrcp.conf.xml
`vim /usr/local/freeswitch/conf/autoload_configs/unimrcp.conf.xml`
```
<configuration name="unimrcp.conf" description="UniMRCP Client">
    <settings>
        <param name="default-tts-profile" value="jtts"/>
        <param name="default-asr-profile" value="jasr"/>
        <param name="log-level" value="DEBUG"/>
        <param name="enable-profile-events" value="false"/>
        <param name="max-connection-count" value="100"/>
        <param name="offer-new-connection" value="1"/>
        <param name="request-timeout" value="3000"/>
    </settings>
    <profiles>
        <X-PRE-PROCESS cmd="include" data="../mrcp_profiles/*.xml"/>
    </profiles>
</configuration>
```
## 4 增加语音识别对接配置文件jasr.xml
`vim /usr/local/freeswitch/conf/mrcp_profiles/jasr.xml`
```
<include>
<profile name="jasr" version="2">
<!--ASR的ip --> 
<param name="server-ip" value="10.163.0.11"/> 
<!--ASR的Mrcp配置文件的sip端口 -->
<param name="server-port" value="5064"/>
<!--freeswitch的ip --> 
<param name="client-ip" value="10.163.0.11"/>
<!--freeswitch的连接ASR的端口 --> 
<param name="client-port" value="5055" />
<param name="sip-transport" value="udp"/>
<!--freeswitch的ip --> 
<param name="rtp-ip" value="10.163.0.11"/>
<param name="rtp-port-min" value="4000"/>
<param name="rtp-port-max" value="5000"/>
<param name="playout-delay" value="50"/>
<param name="min-playout-delay" value="30" />
<param name="max-playout-delay" value="200"/>
<param name="ptime" value="20"/>
<param name="codecs" value="PCMU PCMA L16/96/8000"/>
<param name="rtcp" value="0" />
<!-- Add any default MRCP params for SPEAK requests here -->
<synthparams>
</synthparams>
<recogparams>
</recogparams>
</profile>
</include>
```
## 5 增加语音合成对接配置文件jtts.xml
```
vim /usr/local/freeswitch/conf/mrcp_profiles/jtts.xml
```
```
<include>
<profile name="jtts" version="2">
<!--TTS的ip --> 
<param name="server-ip" value="10.163.0.11"/>
<!--TTS的Mrcp配置文件的sip端口 -->
<param name="server-port" value="5062"/>
<!--freeswitch的ip --> 
<param name="client-ip" value="10.163.0.11"/>
<!--freeswitch的连接TTS的端口 --> 
<param name="client-port" value="5092"/>
<param name="sip-transport" value="udp"/>
<!--freeswitch的ip --> 
<param name="rtp-ip" value="10.163.0.11"/>
<param name="rtp-port-min" value="4000"/>
<param name="rtp-port-max" value="5000"/>
<param name="rtcp" value="1"/>
<param name="rtcp-bye" value="2"/>
<param name="rtcp-tx-interval" value="5000"/>
<param name="rtcp-rx-resolution" value="1000"/>
<param name="codecs" value="PCMU/0/8000"/>
<!-- Add any default MRCP params for SPEAK requests here -->
<synthparams>
</synthparams>
<!-- Add any default MRCP params for RECOGNIZE requests here -->
<recogparams>
</recogparams>
</profile>
</include>
```
## 6 grammar.lua
```
local brokerAddress = "http://10.0.1.10:8080/CSRBroker"
local robothashcode = "robot"
local channelId = "1"
local appKey = "ac5d5452"
local talkerId = "8000"
local receiverId = "8185"
local myUserId = "123456"
local times = os.time()

function initGrammarXML() 
    times = os.time()
    grammar = "<?xml version=\"1.0\" encoding=\"utf-8\"?>"
    grammar = grammar .. "<grammar xmlns=\"http://www.w3.org/2001/06/grammar\" xml:lang=\"en-US\" version=\"1.0\" root=\"service\">"
    grammar = grammar .. "<rule id=\"service\">"
    grammar = grammar .. "<one-of>"
    grammar = grammar .. "<item><ruleref uri=\"#voice-guide\"/></item>"
    grammar = grammar .. "</one-of>"
    grammar = grammar .. "</rule>"
    grammar = grammar .. "<rule id=\"domain\">"
    grammar = grammar .. "<one-of>"
    grammar = grammar .. "<item><ruleref uri=\"#common\"/></item>"
    grammar = grammar .. "</one-of>"
    grammar = grammar .. "</rule>"
    grammar = grammar .. "<rule id=\"need-qa\">"
    grammar = grammar .. "<one-of>"
    grammar = grammar .. "<item>{\"regular\":{\"qa\":{\"address\":\""..brokerAddress.."/queryAction\",\"robot\":\"" .. robothashcode .. "\",\"channel\":\""..channelId.."\",\"appKey\":\""..appKey.."\", \"protocolId\": 5, \"talkerId\": \""..talkerId.."\",\"receiverId\": \""..receiverId.."\",\"type\": \"voice\",\"isNeedClearHistory\": 0, \"isQuestionQuery\": 0, \"userId\":\"" .. myUserId .. "\",\"time\":"..times..",\"msgId\":\"0\"},\"result\":\"singleNode.answerMsg,singleNode.cmd,answerTypeId\"}}</item>"
    grammar = grammar .. "</one-of>"
    grammar = grammar .. "</rule>"
    grammar = grammar .. "</grammar>\r\n\r\n"
    return grammar
end
```
## 7 hello.gram
```
<?xml version="1.0" encoding="utf-8"?>
<grammar xmlns="http://www.w3.org/2001/06/grammar" xml:lang="en-US" version="1.0" root="service">
    <rule id="service">
        <one-of>
            <item>
                <ruleref uri="#voice-guide" />
            </item>
        </one-of>
    </rule>
    <rule id="domain">
        <one-of>
            <item>
                <ruleref uri="#common" />
            </item>
        </one-of>
    </rule>
    <rule id="need-qa">
        <one-of>
            <item>{"regular":{"qa":{"address":"http://172.16.11.2:8080/CSRBroker/queryAction","robot":"lianxinzhicheng","channel":"1","appKey":"ac5d5452", "protocolId": 5, "talkerId": "10021","receiverId": "8185","type": "voice","isNeedClearHistory": 0, "isQuestionQuery": 0, "userId":"4722518476062.7","time":123456789,"msgId":"0"},"result":"singleNode.cmd,singleNode.answerMsg"}}</item>
        </one-of>
    </rule>
</grammar>
```
## 8 hainanliantong.lua
```
package.path = "/usr/local/share/lua/5.2/?.lua"
package.cpath = "/usr/local/lib/lua/5.2/?.so;"

http=require("socket.http")
ltn12=require("ltn12")

--灵云能力平台appkey
local appKey = "ac5d5452";
--机器人问答渠道id
local channelId = "1";
--broker问答地址
local brokerAddress = "http://10.0.1.224:8080/CSRBroker"
--主叫号码
local call_num = session:getVariable("caller_id_number") 
--被叫号码
local talkerId = session:getVariable("destination_number")
local robothashcode = "hainanliantong"
local receiverId = "8185"


math.randomseed(os.time())
--通话的唯一标识
local myUserId = math.random() * 100000000000000
myUserId = "hnlt"..call_num.."_"..math.floor (myUserId)


function parseargs_xml(s)
   local arg = {}
   string.gsub(s, "(%w+)=([\"'])(.-)%2", function (w, _, a)
                        arg[w] = a
                     end)
   return arg
end
 
function parse_xml(s)
   local stack = {};
   local top = {};
   table.insert(stack, top);
   local ni,c,label,xarg, empty;
   local i, j = 1, 1;
   while true do
      ni,j,c,label,xarg, empty = string.find(s, "<(%/?)(%w+)(.-)(%/?)>", i);
      if not ni then
     break
      end
      local text = string.sub(s, i, ni-1);
      if not string.find(text, "^%s*$") then
     table.insert(top, text);
      end
      if empty == "/" then
     table.insert(top, {label=label, xarg=parseargs_xml(xarg), empty=1});
      elseif c == "" then
     top = {label=label, xarg=parseargs_xml(xarg)};
     table.insert(stack, top);
      else
     local toclose = table.remove(stack);
     top = stack[#stack];
     if #stack < 1 then
        error("nothing to close with "..label);
     end
     if toclose.label ~= label then
        error("trying to close "..toclose.label.." with "..label);
     end
     table.insert(top, toclose);
      end
      i = j+1;
   end
   local text = string.sub(s, i);
   if not string.find(text, "^%s*$") then
      table.insert(stack[stack.n], text);
   end
   if #stack > 1 then
      error("unclosed "..stack[stack.n].label);
   end
   return stack[1];
end

-- Used to parse the XML results.
 function getResults(s) 
    local xml = parse_xml(s);
    local stack = {}
    local top = {}
    table.insert(stack, top)
    top = {grammar=xml[1].xarg.grammar, score=xml[1].xarg.score, text=xml[1][1][1]}
    table.insert(stack, top)
    return top;
 end
 
 
local function json2true(str,from,to)
    return true, from+3
end

local function json2false(str,from,to)
    return false, from+4
end

local function json2null(str, from, to)
    return nil, from+3
end

local function json2nan(str, from, to)
    return nul, from+2
end

local numberchars = {
    ['-'] = true,
    ['+'] = true,
    ['.'] = true,
    ['0'] = true,
    ['1'] = true,
    ['2'] = true,
    ['3'] = true,
    ['4'] = true,
    ['5'] = true,
    ['6'] = true,
    ['7'] = true,
    ['8'] = true,
    ['9'] = true,
}

local function json2number(str,from,to)
    local i = from+1
    while(i<=to) do
        local char = string.sub(str, i, i)
        if not numberchars[char] then
            break
        end
        i = i + 1
    end
    local num = tonumber(string.sub(str, from, i-1))
    if not num then
        error(_format('json格式错误，不正确的数字, 错误位置:{from}', from))
    end
    return num, i-1
end

local function json2string(str,from,to)
    local ignor = false
    for i = from+1, to do
        local char = string.sub(str, i, i)
        if not ignor then
            if char == '\"' then
                return string.sub(str, from+1, i-1), i
            elseif char == '\\' then
                ignor = true
            end
        else
            ignor = false
        end
    end
    error(_format('json格式错误，字符串没有找到结尾, 错误位置:{from}', from))
end

local function json2array(str,from,to)
    local result = {}
    from = from or 1
    local pos = from+1
    local to = to or string.len(str)
    while(pos<=to) do
        local char = string.sub(str, pos, pos)
        if char == '\"' then

            result[#result+1], pos = json2string(str,pos,to)
--[[    elseif char == ' ' then

        elseif char == ':' then

        elseif char == ',' then]]
        elseif char == '[' then
            result[#result+1], pos = json2array(str,pos,to)
        elseif char == '{' then
            result[#result+1], pos = json2table(str,pos,to)
        elseif char == ']' then
            return result, pos
        elseif (char=='f' or char=='F') then
            result[#result+1], pos = json2false(str,pos,to)
        elseif (char=='t' or char=='T') then
            result[#result+1], pos = json2true(str,pos,to)
        elseif (char=='n') then
            result[#result+1], pos = json2null(str,pos,to)
        elseif (char=='N') then
            result[#result+1], pos = json2nan(str,pos,to)
        elseif numberchars[char] then
            result[#result+1], pos = json2number(str,pos,to)
        end
        pos = pos + 1
    end
    error(_format('json格式错误，表没有找到结尾, 错误位置:{from}', from))
end

function _G.json2table(str,from,to)
    local result = {}
    from = from or 1
    local pos = from+1
    local to = to or string.len(str)
    local key
    while(pos<=to) do
        local char = string.sub(str, pos, pos)
        if char == '\"' then
            if not key then
                key, pos = json2string(str,pos,to)
            else
                result[key], pos = json2string(str,pos,to)
                key = nil
            end
--[[    elseif char == ' ' then

        elseif char == ':' then

        elseif char == ',' then]]
        elseif char == '[' then
            if not key then
                key, pos = json2array(str,pos,to)
            else
                result[key], pos = json2array(str,pos,to)
                key = nil
            end
        elseif char == '{' then
            if not key then
                key, pos = json2table(str,pos,to)
            else
                result[key], pos = json2table(str,pos,to)
                key = nil
            end
        elseif char == '}' then
            return result, pos
        elseif (char=='f' or char=='F') then
            result[key], pos = json2false(str,pos,to)
            key = nil
        elseif (char=='t' or char=='T') then
            result[key], pos = json2true(str,pos,to)
            key = nil
        elseif (char=='n') then
            result[key], pos = json2null(str,pos,to)
            key = nil
        elseif (char=='N') then
            result[key], pos = json2nan(str,pos,to)
            key = nil
        elseif numberchars[char] then
            if not key then
                key, pos = json2number(str,pos,to)
            else
                result[key], pos = json2number(str,pos,to)
                key = nil
            end
        end
        pos = pos + 1
    end
    error(_format('json格式错误，表没有找到结尾, 错误位置:{from}', from))
end

--json格式中表示字符串不能使用单引号
local jsonfuncs={
    ['\"']=json2string,
    ['[']=json2array,
    ['{']=json2table,
    ['f']=json2false,
    ['F']=json2false,
    ['t']=json2true,
    ['T']=json2true,
}

function _G.json2lua(str)
    local char = string.sub(str, 1, 1)
    local func=jsonfuncs[char]
    if func then
        return func(str, 1, string.len(str))
    end
    if numberchars[char] then
        return json2number(str, 1, string.len(str))
    end
end


--将mrcp返回的结果转化成lua可用的jsonlua play_and_detect_speech
function makeXml2Json(str)
    str = string.sub(str,21,#str-1)
    resultJson = json2lua(str)
    return resultJson
end
function printLog(text)
    if(text) then
        freeswitch.consoleLog("warning","***********************LOG START******************************\n")
        freeswitch.consoleLog("warning",text .. "\n")
        freeswitch.consoleLog("warning","***********************LOG OVER******************************\n")
    else
        freeswitch.consoleLog("warning","***********************LOG START******************************\n")
        freeswitch.consoleLog("warning","object is nil")
        freeswitch.consoleLog("warning","***********************LOG OVER******************************\n")
    end
end

function initGrammarXML() 
    times = os.time()
    grammar = "<?xml version=\"1.0\" encoding=\"utf-8\"?>"
    grammar = grammar .. "<grammar xmlns=\"http://www.w3.org/2001/06/grammar\" xml:lang=\"en-US\" version=\"1.0\" root=\"service\">"
    grammar = grammar .. "<rule id=\"service\">"
    grammar = grammar .. "<one-of>"
    grammar = grammar .. "<item><ruleref uri=\"#voice-guide\"/></item>"
    grammar = grammar .. "</one-of>"
    grammar = grammar .. "</rule>"
    grammar = grammar .. "<rule id=\"domain\">"
    grammar = grammar .. "<one-of>"
    grammar = grammar .. "<item><ruleref uri=\"#common\"/></item>"
    grammar = grammar .. "</one-of>"
    grammar = grammar .. "</rule>"
    grammar = grammar .. "<rule id=\"need-qa\">"
    grammar = grammar .. "<one-of>"
    grammar = grammar .. "<item>{\"regular\":{\"qa\":{\"address\":\""..brokerAddress.."/queryAction\",\"robot\":\"" .. robothashcode .. "\",\"channel\":\""..channelId.."\",\"appKey\":\""..appKey.."\", \"protocolId\": 5, \"talkerId\": \""..talkerId.."\",\"receiverId\": \""..receiverId.."\",\"type\": \"voice\",\"isNeedClearHistory\": 0, \"isQuestionQuery\": 0, \"userId\":\"" .. myUserId .. "\",\"time\":"..times..",\"msgId\":\"0\"},\"result\":\"singleNode.answerMsg,aiResult,singleNode.cmd,answerTypeId\"}}</item>"
    grammar = grammar .. "</one-of>"
    grammar = grammar .. "</rule>"
    grammar = grammar .. "</grammar>\r\n\r\n"
    return grammar
end
function replaceChar(word)
    word = word.gsub(word, """, "\"");
    word = word.gsub(word, "&#x0A;", "\r\n");
    return word
end

function sendMessage2NLU(message)
        local response_body = {}
        times = os.time()
        local reqbody = 'query=' .. message .. '&protocolId=5&userId='..myUserId..'&receiverId=' .. receiverId .. '&talkerId='..talkerId..'&isQuestionQuery=0&msgId=0&type=voice&platformConnType='..channelId..'&appKey='..appKey..'&isNeedClearHistory=0&robotHashCode=' ..robothashcode.."&sendTime="..times;
        res, code = http.request{
          url = "http://10.0.1.224:8080/CSRBroker/queryAction", 
          method = "POST",
          headers =
          {
                ["Content-Type"] = "application/x-www-form-urlencoded;charset=utf-8",
                ["Content-Length"] = string.len(reqbody)
          },
          source = ltn12.source.string(reqbody),
          sink = ltn12.sink.table(response_body)
        }
        return response_body[1];
end

--对话
local prompt = ""
local answerTypeId = ""
local textToNlu = ""

local isOver = false
local isKeyInput = false

local overFlag = "over"
local keyInputFlag = "INPUT"
local ttsName="XuMengJuan"

--返回结果的次数，规定值返回一次
local updateTime = 0
local noAnswerText = "播报抱歉，机器人没有听清，请您再说一遍"
local noAnswerCount = 0 
local useTTS = false
--发送文字给nlu
function textToNluHandle(msg)
    resultBack = sendMessage2NLU(msg)
    luaResult = json2table(resultBack)
    answerTypeId = luaResult.answerTypeId
    prompt = luaResult.singleNode.answerMsg
    isKeyInput = false
    --处理不是场景，也即无法理解的情况
    if(answerTypeId ~= 4) then
        isOver = true
    end
    if(answerTypeId == 4) then
        if(luaResult.singleNode.cmd ~= nil) then
            printLog("-----gxzq---textToNluHandle#cmd-----"..luaResult.singleNode.cmd)
            if(string.len(luaResult.singleNode.cmd)>1) then
                endCode = string.sub(luaResult.singleNode.cmd,1,1)
                actionCode = string.sub(luaResult.singleNode.cmd,2,-1)
                printLog("---0720---textToNluHandel---"..actionCode.."---")
                if(endCode == "1") then
                    isOver = true
                else
                    isOver = false
                end
            else
                endCode = string.sub(luaResult.singleNode.cmd,1,1)
                if(endCode == "1") then
                    isOver = true
                else
                    isOver = false
                end
            end
        end
    end
end

--播报
function toSpeak()
    matchWav()
    if useTTS == true then
            session:playAndGetDigits(1,1,1,1,'',"say:unimrcp:"..ttsName..":"..prompt,"",'[0123456789*#]')
    else
            session:playAndGetDigits(1,1,1,1,'',prompt,"",'[0123456789*#]')
    end
end

--强制挂断流程 比如用户已经挂电话
function forceEndHandle()
    session:sleep(1000)
    session:hangup()
end

--按键输入
function keyInputMethod()
    printLog("nkj#mrcpInputMethod#method#begin-----")
    matchWav()
    if useTTS == true then
               confirmDigits=session:playAndGetDigits(1,11,2,5000,"#","say:unimrcp:"..ttsName..":" .. prompt,"","[0123456789*#]")
    else
                confirmDigits=session:playAndGetDigits(1,11,2,5000,"#","" .. prompt,"","[0123456789*#]")
    end
    
    if(confirmDigits ~= nil and string.len(confirmDigits)>0) then
        textToNlu = confirmDigits
    else
        textToNlu = "nokeyinput"
    end
    printLog("nkj#mrcpInputMethod#method#end-----")
    printLog("nkj#mrcpInputMethod#nlu#begin-----")
    textToNluHandle(textToNlu)
    printLog("nkj#mrcpInputMethod#nlu#end-----")
end

--发送语音流mrcp 调用nlu
--适用与场景交互 知识问答还需添加answertype的判断
function mrcpInputMethod()
    --session:execute("detect_speech", "stop")
    local grammar = initGrammarXML()
        matchWav()
        if useTTS == true then
            session:execute("play_and_detect_speech","say:unimrcp:"..ttsName..":"..prompt.."detect:unimrcp: {start-input-timers=false,no-input-timeout=10000,recognition-timeout=15000,speech-complete-timeout=400,sensitivity-level=0.13} inline:" .. grammar)
            --session:execute("play_and_detect_speech","say:unimrcp:"..ttsName..":,,,, detect:unimrcp: {start-input-timers=true,no-input-timeout=10000,recognition-timeout=15000,speech-complete-timeout=400,sensitivity-level=0.13} inline:" .. grammar)
        else
            session:execute("play_and_detect_speech",prompt.." detect:unimrcp: {start-input-timers=true,no-input-timeout=10000,recognition-timeout=15000,speech-complete-timeout=400,sensitivity-level=0.13} inline:" .. grammar)
               --session:execute("play_and_detect_speech",prompt.." detect:unimrcp: {start-input-timers=true,no-input-timeout=10000,recognition-timeout=15000,speech-complete-timeout=400,sensitivity-level=0.13} inline:" .. grammar)
        end
    xml = session:getVariable('detect_speech_result')
    if(xml~=nil) then
        xml = replaceChar(xml)
    end
    if(xml ~= nil and string.len(xml)>50) then --识别有返回
        noAnswerCount = 0
        nluResult = parse_xml(xml)[2][1][1][1][1]
        jsonResult = json2table(nluResult)
        results = jsonResult.results
        aiResult = results[1].aiResult
        isKeyInput = false
        if(results[1].answerMsg ~= nil) then
            prompt = results[1].answerMsg;
            answerTypeId = results[1].answerTypeId
            if(results[1].cmd ~= nil) then
                if(string.len(results[1].cmd)>1) then
                        endCode = string.sub(results[1].cmd,1,1)
                        actionCode = string.sub(results[1].cmd,2,-1)
                                printLog("---0720---"..endCode.."---"..actionCode.."---")
                        if(endCode == "1") then
                            isOver = true
                        else
                            isOver = false
                        end
                    else
                        endCode = string.sub(results[1].cmd,1,1)
                        if(endCode == "1") then
                            isOver = true
                        else
                            isOver = false
                        end
                end
            end
        else
            --没有回答
            prompt = noAnswerText
            noAnswerCount = noAnswerCount + 1
        end    
    else  --识别没返回
        prompt = noAnswerText
        noAnswerCount = noAnswerCount + 1
        textToNluHandle("silent")
    end
    --连续两次没有结果 需要设置需要挂断
    if (noAnswerCount >= 2) then
        isOver = true
        printLog("nkj#mrcpInputMethod#noAnswerCount#Over------")
    end
end

--流程主方法
function QAChat()
    --进入场景
    textToNluHandle("开始")
    if(session:ready() ~= true)then
                forceEndHandle()
        end
    --多轮对话
    while(session:ready() == true) do
                --判断流程是否挂断
                if isOver then
                        toSpeak()
                        printLog("nkj-------挂断---------")
                        forceEndHandle()
                        break;
                end
              
        --判断按键输入还是语音输入
        if isKeyInput then
            keyInputMethod()
        else
            --调用播报方法
                --    toSpeak()
            mrcpInputMethod()
            
        end
        
         --判断session是否还在
        if(session:ready() ~= true)then
            forceEndHandle()
            break;
        end
    end
        if(session:ready() ~= true)then
                forceEndHandle()
        end
end

local msgKeyValue = {}
msgKeyValue["1"]="您好！我是海南联通客服代表小王，工号0001，感谢您使用联通宽带，来电主要是对您近期新装的宽带情况进行回访，可以占用您一到两分钟时间吗？"
msgKeyValue["2"]="请问我司工作人员是否已为您完成宽带安装呢？"
msgKeyValue["3"]="不好意思，只打搅您1到2分钟时间，好吗？"
msgKeyValue["4"]="请问您是否有申请安装联通宽带呢？"
msgKeyValue["5"]="好的，那请问工作人员上门装机时有穿工装和带工作牌吗？"
msgKeyValue["6"]="很抱歉，我们马上将您的问题反馈到相关部门，安排装维人员与您联系，尽快完成安装。那请问工作人员上门装机时有穿工装和带工作牌吗？"
msgKeyValue["7"]="很抱歉，我们马上将您的问题反馈到相关部门，安排装维人员与您联系，尽快完成安装。非常感谢您接受本次的回访，祝您生活愉快，再见！"
msgKeyValue["8"]="很抱歉，我们马上将您的问题反馈到相关部门，安排装维人员与您联系。非常感谢您接受本次的回访，祝您生活愉快，再见！"
msgKeyValue["9"]="那打扰您了，感谢您的接听，祝您生活愉快，再见！"
msgKeyValue["10"]="好的，那请问安装完后工作人员为您清理现场了吗？"
msgKeyValue["11"]="感谢您反馈的情况，我们已经记录下来了，并会及时改进要求工作人员以专业形象面对客户。那请问安装完后工作人员为您清理现场了吗？"
msgKeyValue["12"]="不好意思，马上结束了，可以再耽误你一下吗？"
msgKeyValue["13"]="那您对安装过程满意吗？"
msgKeyValue["14"]="工作人员未及时清理给您带来不便，我们在此至于诚挚歉意，此问题我们已经记录并会及时改进，希望能给您带来更好的服务。您对安装过程满意吗？"
msgKeyValue["15"]="那1-10分您打几分？10分为最高分。"
msgKeyValue["16"]="非常感谢您接受我本次的回访，祝您生活愉快，再见！"
msgKeyValue["17"]="我们希望给客户提供更满意的服务，让客户使用更舒心。您觉得我们哪方面做的不好，需要改进的呢？"
msgKeyValue["18"]="非常感谢您给我们提供的宝贵意见，祝您生活愉快，再见！"
msgKeyValue["19"]="那是没有安装好呢还是没有安装？"
msgKeyValue["20"]="我是联通客服代表小王，来电主要是对您近期新装的宽带情况进行回访，可以占用您一到两分钟时间吗？"
msgKeyValue["21"]="不好意思，我这边实在听不到您的声音，本次来电是想对您近期新装的宽带情况进行回访，感谢您的接听，再见！"
msgKeyValue["22"]="不好意思，我这边没有听清，您能再说一遍吗？"
msgKeyValue["23"]="那请问工作人员上门装机时有穿工装和带工作牌吗？"


function matchWav()
    useTTS = true
    for key, value in pairs(msgKeyValue) do
        if value == prompt then
            useTTS = false
            prompt = "/usr/local/freeswitch/dzsounds/hainanliantong/"..key..".wav"
            break;
        end
    end
end

session:set_tts_params("unimrcp:jtts", ttsName)
QAChat()
```
