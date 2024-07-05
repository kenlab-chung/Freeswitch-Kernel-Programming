# FreeSWITCH对接Vosk 免费ASR语音识别引擎
## 1 Vosk简介
Vosk是一个离线开源语音识别工具，它可以识别16中语言，包含中文。总体效果还是不错的，因为我们要对接到呼叫中心，因此我们需要实时的流式传输语音数据，目前主流方案就是采用websocket协议传输语音，Vosk直接提供了Websocket的Server程序。而且程序已经打包成docker发布，所以启动起来特别简单：
```
docker run -d -p 2700:2700 alphacep/kaldi-cn:latest
```
