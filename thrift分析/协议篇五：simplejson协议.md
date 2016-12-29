# simplejson协议
  分析thrift simplejson协议格式 

  simplejson协议为write协议，read不支持，因其协议格式中对字段的处理，没有标识字段号，所以不能read，只分析write部分

# 协议分析
## 方法调用
方法调用都分为三部分写数据：消息头，参数内容   

### 一、写消息头

消息头图  

```
----------------------------
| 方法 | 方法调用类型 | 序列ID | 
----------------------------
```
**方法**：方法名字符串

**方法调用类型**：方法调用类型 call 1

**序列ID**：方法调用的序列ID

数据格式，例如：

```
"HelloWorld",1,1
```

### 二、写参数内容

写完消息头之后，开始写参数内容，此部分分四部分内容：方法参数结构体开始、参数字段内容、参数结构结尾。

参数内容总体格式为：

```
---------------------
| 字段名 |  字段内容 | 
---------------------
```

数据格式，例如：

```
{{"name":"test"},{"maps":[13,8,3,"test",10,"test1",11,"test2",12]}}
```

####2.1字段参数内容

参数内容以下格式

```
---------------
| 字段名 : 内容 | 
---------------
```
参数类型做字符串简写：

  **简单类型** 

```
-------------
| 名称 | 内容 | 
-------------
```
字符串样例：

字段名称name

```
{"name":"test"}
```

  **map类型**  

```
-----------------------------------———------------------
| 字段名称| key类型标识 | value类型标识 | mapsize | map内容 |
-----------------------------------———------------------
```
key类型标识，取值key类型对应的整数类型

value类型标识，取值value类型对应的整数类型

map内容：用逗号分隔

类型：bool=2、byte=3、i8=3、i16=6、i32=8、i64=10、double=4、map=13、list=15、set=14、struct=12、string=11等  

样例：

字段名称name，key为string，value为i32

```
{"name":[11,8,3,"test",10,"test1",11,"test2",12]}
```

  **set和list类型**    

```
-----------------------------------———---
| 字段名称| value类型标识 | size | 集合内容 |
-----------------------------------———---
```
key类型标识，取值key类型对应的整数类型

内容：用逗号分隔

样例：

字段名称name，key为string

```
{"name":[11,3,"test","test1","test2"]}
```

  **struct类型**   
  类似写方法参数，根据struct里面的字段顺序开始写，处理每个字段的字段内容

  样例：

  ```
  {"name":{"name":"test"}}
  ```


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
协议使用simplejson协议，普通socket接收和发送

模拟发送代码如下：

```
	// //模拟调用string HelloWorld()
	s := "[\"HelloWorld\",1,1,{}]"
	conn.Write([]byte(s))

	conn.Write([]byte{'['})              //message begin
	conn.Write([]byte("\"HelloWorld\"")) //METHOD NAME
	conn.Write([]byte{','})
	conn.Write([]byte("1")) //THRIFT CALL
	conn.Write([]byte{','})
	conn.Write([]byte("1")) // METHOD SEQID
	conn.Write([]byte{','})
	conn.Write([]byte{'{'}) //空参数
	conn.Write([]byte{'}'})
	conn.Write([]byte{']'}) //message end

	//模拟string参数
	str := "[\"HelloWorldForString\",1,2,{\"name\":\"test\"}]"
	conn.Write([]byte(str))

	//模拟map
	mapstr := "[1,\"HelloWorldForMap\",1,3,{\"name\":[11,8,3,\"test\",11,\"test1\",12,\"test2\",13]}]"
	conn.Write([]byte(mapstr))

	//模拟struct
	structstr := "[1,\"HelloWorldForStruct\",1,4,{\"name\":{\"name\":\"test\"}}]"
	conn.Write([]byte(structstr))
```

完整的示例见gostudy中的thrifttest/simplejson  
[https://github.com/chengnl/gostudy/tree/master/thrifttest/simplejson](https://github.com/chengnl/gostudy/tree/master/thrifttest/simplejson)



