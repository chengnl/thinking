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

4字节版本号：版本号|消息类型组成的4字节内容，其中版本号为取值0x80010000,消息类型取值1，表示方法调用，该4字节内容为0x80010000|0x00000001

4字节方法名长度：获取方法名字符串长度len(methodName)

方法名字串：方法名内容字节

4字节序列ID：thrift客户端调用方法序列累加计数

2.非strictWrite模式

消息头图![非strictWrite模式消息头][1]
[1]:https://github.com/chengnl/thinking/blob/master/images-folder/thrift/bmhead.png "非strictWrite模式消息头"

4字节方法名长度：获取方法名字符串长度len(methodName)

方法名字串：方法名内容字节

1字节消息类型：消息类型取值1，表示方法调用

4字节序列ID：thrift客户端调用方法序列累加计数

### 二、写参数内容


