# multiplexed协议
  多路复合协议
  原理：使用代理协议，根据不同的服务请求方法，针对性的使用服务本身的协议处理（目前仅支持二进制协议）

  服务端提供二进制接收处理，根据不同的客户端请求，分析请求头，然后分处不同的业务处理器处理

# 协议分析
因其主要是使用服务本身代理协议处理，所以其方法，字段，内容处理基本都是服务的协议，这里主要分析下它怎么做到路由服务。

## 方法调用
针对调用CALL和ONEWAY处理，写方法消息头做特殊处理，方法调用回复不用做特殊处理  

### 写方法消息头

消息头跟其他的协议区别，就是将方法名变成了：

serviceName:方法名  

然后使用服务本身使用的协议，写方法头即

WriteMessageBegin(t.serviceName+MULTIPLEXED_SEPARATOR+name, typeId, seqid)

也即时multiplexed协议支持覆写一下WriteMessageBegin，将方法名变为serviceName:方法名 

## 方法接收
因客户端发送内容的时候，将方法消息头做了处理，所以对应接收端也要对读消息头做下处理，分发给不同的业务处理器处理

multiplexed协议的Processor将业务的Processor进行了封装

服务启动时将业务的Processor都注册进multiplexed协议的Processor，然后传入server，调用逻辑：

```
-----------------------------------------------------------------------------------
|TServer >> TMultiplexedProcessor >> 获取serviceName >> service的Processor >>业务处理|
-----------------------------------------------------------------------------------

```

在server调用Processor的时候，实际调动的是multiplexed协议的Processor

这个时候Processor处理的时候先调用服务协议读取消息头，得到方法名，然后将方法名根据“serviceName:方法名”格式解析，得到对应的服务名和方法名

等到服务名后获取对应的业务处理器，然后使用对应的业务处理器处理后续内容。这个位置看golang处理的时候比较优雅：

```
func (t *TMultiplexedProcessor) Process(in, out TProtocol) (bool, TException) {
	name, typeId, seqid, err := in.ReadMessageBegin()
	if err != nil {
		return false, err
	}
	if typeId != CALL && typeId != ONEWAY {
		return false, fmt.Errorf("Unexpected message type %v", typeId)
	}
	//extract the service name
	v := strings.SplitN(name, MULTIPLEXED_SEPARATOR, 2)
	if len(v) != 2 {
		if t.DefaultProcessor != nil {
			smb := NewStoredMessageProtocol(in, name, typeId, seqid)
			return t.DefaultProcessor.Process(smb, out)
		}
		return false, fmt.Errorf("Service name not found in message name: %s.  Did you forget to use a TMultiplexProtocol in your client?", name)
	}
	actualProcessor, ok := t.serviceProcessorMap[v[0]]
	if !ok {
		return false, fmt.Errorf("Service name not found: %s.  Did you forget to call registerProcessor()?", v[0])
	}
	smb := NewStoredMessageProtocol(in, v[1], typeId, seqid)
	return actualProcessor.Process(smb, out)
}

//Protocol that use stored message for ReadMessageBegin
type storedMessageProtocol struct {
	TProtocol
	name   string
	typeId TMessageType
	seqid  int32
}

func NewStoredMessageProtocol(protocol TProtocol, name string, typeId TMessageType, seqid int32) *storedMessageProtocol {
	return &storedMessageProtocol{protocol, name, typeId, seqid}
}

func (s *storedMessageProtocol) ReadMessageBegin() (name string, typeId TMessageType, seqid int32, err error) {
	return s.name, s.typeId, s.seqid, nil
}

```

直接扩展一个NewStoredMessageProtocol协议，只是重写一下ReadMessageBegin方法，这样Process处理的时候相当于协议重头读取解析。

## 方法调用返回
因每个client端，返回的时候都是对应的连接，所以返回给对应的client的读取，直接使用服务本身协议读取内容即可





