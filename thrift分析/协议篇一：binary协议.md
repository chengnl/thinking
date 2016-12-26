# binary协议
  分析thrift二进制传输协议格式，普通socket传输
  完整的示例见gostudy中的thrifttest/binary[https://github.com/chengnl/gostudy/tree/master/thrifttest/binary](https://github.com/chengnl/gostudy/tree/master/thrifttest/binary)
# 协议分析
## 方法调用
方法调用都分为三部分写数据：消息头，参数内容，消息结尾  
其中消息结尾，没有做任何处理，主要关注---消息头|参数内容部分。  
目前针对写动作，分为strictWrite模式和非strictWrite模式

### 一、写消息头

1.strictWrite模式

消息头图![strictWrite模式消息头][1]
[1]:https://github.com/chengnl/thinking/blob/master/images-folder/thrift/bmheads.png "strictWrite模式消息头"

**4字节版本号**：版本号|消息类型组成的4字节内容，其中版本号为取值0x80010000,消息类型取值1，表示方法调用，该4字节内容为0x80010000|0x00000001  

**4字节方法名长度**：获取方法名字符串长度len(methodName)  

**方法名字串**：方法名内容字节  

**4字节序列ID**：thrift客户端调用方法序列累加计数  

2.非strictWrite模式

消息头图![非strictWrite模式消息头][2]
[2]:https://github.com/chengnl/thinking/blob/master/images-folder/thrift/bmhead.png "非strictWrite模式消息头"

**4字节方法名长度**：获取方法名字符串长度len(methodName)  

**方法名字串**：方法名内容字节  

**1字节消息类型**：消息类型取值1，表示方法调用  

**4字节序列ID**：thrift客户端调用方法序列累加计数  

### 二、写参数内容

写完消息头之后，开始写参数内容，此部分分四部分内容：方法参数结构体开始、参数字段内容、写字段结束、参数结构结尾。其中参数结构体开始和结尾部分，不做任何写处理，可忽略，
主要关注字段和字段结束，所以参数内容，主要是是将各字段的内容写入，最后加写字段结束符 

####2.1参数字段

写字段，根据参数顺序编号写  
字段处理图![字段处理图][3]
[3]:https://github.com/chengnl/thinking/blob/master/images-folder/thrift/field.png "字段处理图"

**1字节参数类型**：参数类型值，分bool=2、byte=3、i8=3、i16=6、i32=8、i64=10、double=4、map=13、list=15、set=14、struct=12等  

**2字节字段顺序号**：字段在方法参数上的标记序号

**参数内容**：参数内容根据不同的参数类型处理，具体见参数类型部分

参数多个情况，根据上面消息格式依次处理每个字段参数

#####2.1.1参数类型

不同参数类型，在参数内容处理上有所不同，分下面的两种情况：  

1.固定长度类型参数  
  此部分类型有：bool=2、byte=3、i8=3、i16=6、i32=8、i64=10、double=4  
  此类型的字段，参数内容即写类型对应的固定长度的内容值即可。 

2.变长类型参数  

  变成类型主要有：string、map、set、list、struct  

  **string类型**  
  string类型![string类型][4]
[4]:https://github.com/chengnl/thinking/blob/master/images-folder/thrift/stringfield.png "string类型"  
  4字节参数内容长度、参数内容字节 两部分组成

  **map类型**  
  map类型![map类型][5]
[5]:https://github.com/chengnl/thinking/blob/master/images-folder/thrift/mapfield.png "map类型"  
  1字节key类型：key类型  
  1字节value类型：value类型  
  4字节map的size：map大小  
  map内容：循环写key，value内容，根据key、value的类型做不同的写处理 

  **set和list类型**    
  set和list类型![set和list类型][6]
[6]:https://github.com/chengnl/thinking/blob/master/images-folder/thrift/slfield.png "set和list类型"  
  1字节value类型：类型字节  
  4字节集合size：集合大小  
  写value内容：循环写value内容，根据value类型做不同的写处理  

  **struct类型**   
  类似写方法参数，根据struct里面的字段顺序开始写，处理每个字段的字段内容，字段结束标识

####2.2字段结束
所有字段写完之后，写入1个字节结束符，该字节值为0


### 三、代码示例
定义简单的方法：

```
 string HelloWorld()

```
协议使用二进制协议，普通socket接收和发送

模拟发送代码如下：

```
func sendMethod() {
	//模拟调用string HelloWorld()
	methodName := "HelloWorld"
	var seqNum int32 = 1

	//写消息头
	//4字节----version|方法调用1
	v := uint32(0x80010000) | uint32(1)
	writeI32(v)
	//4字节----方法名长度
	len := len(methodName)
	writeI32(uint32(len))
	//方法内容字节----方法名
	conn.Write([]byte(methodName))
	//4字节----方法调用序列号
	writeI32(uint32(seqNum))
	//写参 无参数，忽略

	//1字节----字段结束符  STOP 0
	writeByte(0)
}

func writeI32(val uint32) {
	v := buf[0:4]
	_ = v[3]
	v[0] = byte(val >> 24)
	v[1] = byte(val >> 16)
	v[2] = byte(val >> 8)
	v[3] = byte(val)
	conn.Write(v)
}
func writeByte(val byte) {
	v := buf[0:1]
	_ = v[0]
	v[0] = val
	conn.Write(v)
}

```


## 方法调用返回
类似方法调用分为三部分写数据：消息头，结果内容，消息结尾  
其中消息结尾，没有做任何处理，主要关注---消息头|结果内容部分。  
目前针对写动作，也分为strictWrite模式和非strictWrite模式

特别：  

如果是void方法，写消息头后，直接返回字段结束符，不用写结果内容，相当于无返回字段

如果是oneway方式，不用写消息，相当于不关心方法结果。


### 一、写消息头

1.strictWrite模式

消息头图![strictWrite模式消息头][7]
[7]:https://github.com/chengnl/thinking/blob/master/images-folder/thrift/bmheadsback.png "strictWrite模式消息头"

**4字节版本号**：版本号|消息类型组成的4字节内容，其中版本号为取值0x80010000,消息类型取值2，表示方法调用返回，该4字节内容为0x80010000|0x00000010  

**4字节方法名长度**：获取方法名字符串长度len(methodName)  

**方法名字串**：方法名内容字节  

**4字节序列ID**：thrift客户端调用方法序列累加计数  

2.非strictWrite模式

消息头图![非strictWrite模式消息头][8]
[8]:https://github.com/chengnl/thinking/blob/master/images-folder/thrift/bmheadback.png "非strictWrite模式消息头"

**4字节方法名长度**：获取方法名字符串长度len(methodName)  

**方法名字串**：方法名内容字节  

**1字节消息类型**：消息类型取值2，表示方法调用返回 

**4字节序列ID**：thrift客户端调用方法序列累加计数  

### 二、写结果内容

处理方式与调用处理参数类似，也是处理1字节参数类型，2字节顺序号，结果内容

所有字段内容写完之后，写入1个字节结束符，该字节值为0

### 三、代码示例
定义简单的方法：

```

 string HelloWorld()

```
协议使用二进制协议，普通socket接收和发送

模拟接收代码如下：

```
func receiveResult() {
	//读回消息头
	//4字节----version|方法回复2
	v := readI32()
	fmt.Printf("VERSION_1=:%d，type=:%d\n", uint32(v)&0xffff0000, v&0x0000ffff)
	//4字节----方法名长度
	len := readI32()
	fmt.Printf("method len=:%d\n", len)
	//方法名内容字节
	method := make([]byte, len)
	conn.Read(method)
	fmt.Printf("method name=:%s\n", string(method))
	//4字节----方法调用序列号
	seq := readI32()
	fmt.Printf("method seq=:%d\n", seq)

	//读取返回字段
	for {
		//1字节----字段类型
		fieldVal := readByte()
		fmt.Printf("result field type=:%d\n", fieldVal)
		if fieldVal == 0 { //读字段结标识符
			fmt.Printf("result field read end\n")
			break
		}
		//2字节----字段顺序号
		fieldOrderVal := readI16()
		fmt.Printf("result field order=:%d\n", fieldOrderVal)
		switch fieldVal {
		case 11: //字符串
			//读4个字节，字符串长度
			l := readI32()
			fmt.Printf("result len=:%d\n", l)
			//读字符串内容
			content := make([]byte, l)
			conn.Read(content)
			fmt.Printf("result content=:%s\n", string(content))
		default: //其他类型，暂时不模拟
			return
		}
	}
}

func readI32() int32 {
	b := make([]byte, 4)
	conn.Read(b)
	v := uint32(b[0])<<24 | uint32(b[1])<<16 | uint32(b[2])<<8 | uint32(b[3])
	return int32(v)
}
func readI16() int16 {
	b := make([]byte, 2)
	conn.Read(b)
	v := uint16(uint16(b[0])<<8 | uint16(b[1]))
	return int16(v)
}
func readByte() byte {
	b := make([]byte, 1)
	conn.Read(b)
	return byte(b[0])
}

```





