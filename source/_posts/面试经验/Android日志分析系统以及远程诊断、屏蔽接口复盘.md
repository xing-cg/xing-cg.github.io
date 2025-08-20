---
title: Android日志分析系统复盘
categories:
  - - 面试经验
tags: 
date: 2025/8/18
updated: 2025/8/18
comments: 
published:
---
搭建一个HTTP后台服务，（网络库，线程安全）
根据提供的URL，下载日志压缩包，（下载服务，wget）
然后进行压缩包的解压操作。（解压服务，tar、7z）
解压之后，编写脚本程序统计日志文件数目、分类。（脚本编程，Shell）
使用检索工具，解析日志的关键内容。（检索，rg）

# 目前支持解析的日志类型
1. anr、tombstone的数目
2. logdump、kernel log、tombstone的具体事件
    1. 均已支持检索正则表达式及显示其原句
3. dropbox panic字眼

# 问题解决 - 报文拆分、超频重发
响应的消息要发到企业微信API。
每次的消息都有长度限制为5120个字符，如果超过了限制就需要做一个报文拆分处理

虽然企业微信api发送成功率很高，但是服务器也有必要确认客户端收到了消息，直到收到回发的ok才放心。

按目前项目要处理的内容的体量来说不那么紧要，但是对于可能扩大规模很有必要，
企业微信接口每分钟只能发20条消息，如果收到超频率应答，需要阻塞1分钟重发。总的来说，需要完善一下系统的确认应答机制。
## 三种异常处理：分包、循环+重传递归
```cpp
    // 查项目目前支持的errcode表
    if(WorkWXErrCodeErrMsg.find(receivedJson["errcode"]) != WorkWXErrCodeErrMsg.end())
    {
        // 接收到ok报文，即客户端确认收到
        if(receivedJson["errcode"] == 0){
            return true;
        }
        else // 异常处理
        {
            LOG_ERROR << "WorkWX API Return Json ErrMsg: " << string(receivedJson["errmsg"]);
            // 超长，分包处理
            if(receivedJson["errcode"] == WorkWXErrCode::MSG_OUT_OF_LENGTH)
            {
                vector<json> splitContentJsonVec;
                SplitJsonMsgPack(splitContentJsonVec, contentJson, fileName);
                for(const auto & pack : splitContentJsonVec)
                {
                    if(!SendJsonToWX(webhook, fileName, pack, level + 1))
                    {
                        LOG_ERROR << "Failed to send split message pack, contentJson was log to the file.";
                        LOG_ERROR << contentJson.dump();
                        return false;
                    }
                }
                return true;
            }// 企业微信api规定1分钟只能发20条消息。超频，阻塞1min处理
            else if(receivedJson["errcode"] == WorkWXErrCode::FREQ_OUT_OF_LIMIT){
                sleep(60);
                return SendJsonToWX(webhook, fileName, contentJson, level + 1);
            }// 微信系统繁忙，超时，阻塞level min重发
            else if(receivedJson["errcode"] == WorkWXErrCode::BUSY){
                sleep(level * 60);
                return SendJsonToWX(webhook, fileName, contentJson, level + 1);
            }
        }
    }
```

```cpp
void SplitJsonMsgPack(vector<json> & jsonVec, const json &j, const string &fileName)
{
    string msgtype = j["msgtype"];
    string content = j[msgtype]["content"];
    //每个包加 文件名+count(页数)  因为不能超过4096个所以此处抽出72个字节存放包名
    for(int count = 1, pos = 0; pos <= content.size(); pos += 4024, ++count)
    {
        string head = fileName + " :" + to_string(count) + "\n";
        string splitMsg = head + content.substr(pos, 4024);
        json contentJson = {{"content", splitMsg.c_str()}};
        contentJson = {{"msgtype", "markdown"}, {"markdown", contentJson}};
        jsonVec.emplace_back(contentJson);
    }
}
```
# 问题解决 - 本地下载问题
wget -N 参数：如果本地文件​**​不存在​**​，它会直接下载该文件；如果本地文件​**​已经存在​**​，Wget 会向服务器发出一个 ​**​`HEAD`请求​**​（而非完整的 `GET`请求）来获取该资源的​**​最后修改时间​**​ (`Last-Modified`头部)。

如果一个终端下载了此文件，然后在本地删除掉此文件，再尝试下载时，发现有时不会下载，即误认为存在一样的文件。

需要和后台那边进行沟通，让其提供一个校验码接口，以便可以和本地对比。
## 先尝试解压，后下载
优化了一下处理逻辑，先尝试解压再进行下载，对于已经下载好的日志包，效果很明显，直接略去了不必要的下载。
## 和后台联调 - 压缩包检验码问题
问了后台那边网址包含不包含压缩包校验码，
如果不包含，那就把异常情况放在解压缩的环节上，在解压缩脚本上进行异常处理
# C调用Shell
popen、system、fork+exec三者的区别、和异常处理，

其中system可以让父进程自动阻塞，避免了下载文件、解压缩的进程安全问题，省去了好多进程控制的代码。但是返回值挺多的，得分好多情况，

popen是封装了fork、pipe、exec，自动关闭了不需要的读写端，由于封装了三个操作，可能代码上很方便，但是可能内部会出现一些异常，需要去避开坑点，

而fork+exec虽然很原始，但是让代码逻辑很明了，主进程可以在等待期间做一些自定义的事。

所以把代码中需要调用脚本的地方各取所需，分情况调用。这也给了一个启发，需要在调用时多看手册、多Google，明确细节和坑点。

# 日志分析系统总体流程

# 远程诊断新的需求
学习守护平台远程诊断日志上报模块有新的需求，昨天开发的是扫描局域网设备数的代码，需了解安卓多线程和异步编程的知识，学习service有关内容来更好的进行开发远程诊断模块。

前期主要做的是设备端的诊断代码实现，熟悉一下IoT的接口规范，和后台那边对接一下数据格式，熟悉一下传感器的知识，如何获取数据。

熟悉远程诊断现有的代码框架，在此基础上进行扩展，完成获取时间的代码，熟悉麦克风检测和获取传感器的代码。

代码完成后，和后台沟通数据模型，和后台进行联调。

完成网卡信息的获取代码，完成JSON的整合。自测之后，和后台那边进行联调，目前链路是通的，格式正确，但是有些值获取的有点小问题，比如子网掩码。

整理远程诊断的代码，提交到仓库。

调试了一下TC02和DT15的IoT远程诊断功能，解决了一些小问题，提交最新的代码等待合并。

