# binary协议
  分析thrift二进制传输协议格式，普通socket传输
# 协议分析
## 方法调用
方法调用都分为三部分写数据：消息头，参数内容，消息结尾

其中消息结尾，没有做任何处理，主要关注消息头和参数内容部分。

目前针对写动作，分为strictWrite模式和非strictWrite模式

### 一、写消息头

1.strictWrite模式

消息头图![strictWrite模式消息头][1]
[1]:https://github.com/chengnl/thinking/blob/master/images-folder/thrift/bmheads.png "strictWrite模式消息头"


2.非strictWrite模式

### 二、写参数内容


