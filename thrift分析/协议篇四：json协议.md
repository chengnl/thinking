# json协议
  分析thrift json协议格式 

  类型为字符串加"",整形不用加

# 协议分析
## 方法调用
方法调用都分为三部分写数据：消息头，参数内容   

### 一、写消息头

消息头图  

```
-------------------------------------
|协议标识 | 方法 | 方法调用类型 | 序列ID | 
-------------------------------------
```

**协议标识**：取值1

**方法**：方法名字符串

**方法调用类型**：方法调用类型 call 1

**序列ID**：方法调用的序列ID

数据格式，例如：

```
1,"HelloWorld",1,1
```

### 二、写参数内容

写完消息头之后，开始写参数内容，此部分分四部分内容：方法参数结构体开始、参数字段内容、参数结构结尾。

参数内容总体格式为：

```
---------------------------------------
|字段序号 | 字段内容 | 字段序号 | 字段内容 | 
---------------------------------------
```

数据格式，例如：

```
{1:{"str":"test"},2:{"map":["str","i32",3,{"test":10,"test1":11,"test2":12}]}}
```

####2.1参数内容

参数内容以下格式

```
-------------
| 类型 : 内容 | 
-------------
```
参数类型做字符串简写：

```
	case BOOL:
		return "tf", nil
	case BYTE:
		return "i8", nil
	case I16:
		return "i16", nil
	case I32:
		return "i32", nil
	case I64:
		return "i64", nil
	case DOUBLE:
		return "dbl", nil
	case STRING:
		return "str", nil
	case STRUCT:
		return "rec", nil
	case MAP:
		return "map", nil
	case SET:
		return "set", nil
	case LIST:
		return "lst", nil
```  
  **简单类型** 

```
-------------
| str | 内容 | 
-------------
```
样例：

```
{"str":"test"}
```

  **map类型**  

```
-----------------------------------———--------------
| map| key类型标识 | value类型标识 | mapsize | map内容 |
-----------------------------------———--------------
```
样例：

```
{"map":["str","i32",3,{"test":10,"test1":11,"test2":12}]}
```

  **set和list类型**    

```
-----------------------------------———---
| set/lst| value类型标识 | size | 集合内容 |
-----------------------------------———---
```
样例：

```
{"set":["str",3,["test","test1","test2"]]}
```

  **struct类型**   
  类似写方法参数，根据struct里面的字段顺序开始写，处理每个字段的字段内容


定义简单的方法：

```
    struct Name{
        1:string name
    }

    string HelloWorld()
    string HelloWorldForString(1:string name)
    string HelloWorldForMap(1:map<string,i32> name)
    string HelloWorldForStruct(1:Name name)

```
协议使用json协议，普通socket接收和发送

模拟发送代码如下：

```
    //模拟调用string HelloWorld()
    //s:="[1,\"HelloWorld\",1,1,{}]"
	//conn.Write([]byte(s))

	conn.Write([]byte{'['})//message begin
	conn.Write([]byte("1"))//json 协议ID
	conn.Write([]byte{','})
	conn.Write([]byte("\"HelloWorld\"")) //METHOD NAME
    conn.Write([]byte{','})
	conn.Write([]byte("1"))//THRIFT CALL
	conn.Write([]byte{','})
	conn.Write([]byte("1"))// METHOD SEQID
	conn.Write([]byte{','})
    conn.Write([]byte{'{'})//空参数
    conn.Write([]byte{'}'})
	conn.Write([]byte{']'})//message end

    //模拟string参数
	str:="[1,\"HelloWorldForString\",1,2,{1:{\"str\":\"test\"}}]"
	conn.Write([]byte(str))

    //模拟map
	mapstr:="[1,\"HelloWorldForMap\",1,3,{1:{\"map\":[\"str\",\"i32\",3,{\"test\":11,\"test1\":12,\"test2\":13}]}}]"
	conn.Write([]byte(mapstr))

	//模拟struct
	structstr:="[1,\"HelloWorldForStruct\",1,4,{1:{\"rec\":{1:{\"str\":\"test\"}}}}]"
	conn.Write([]byte(structstr))
```

完整的示例见gostudy中的thrifttest/json  
[https://github.com/chengnl/gostudy/tree/master/thrifttest/json](https://github.com/chengnl/gostudy/tree/master/thrifttest/json)

## 方法调用返回

参照方法调用，读取返回的字节。


