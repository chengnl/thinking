对常用的rpc技术分析，目前只分析了下thrift和58的Gaea

初步看了下源码，先简单说下里面的不同点，后续写文档详细说明下情况：

thrift优点：  
1、IDL文件，各端类定义统一  
2、协议和序列化结合，利用自动生成类转化为基本类型序列化，语言扩展性强  
3、字段扩展，optional，require  

thrift缺点：  
1、客户端自己实现（动态代理、负载均衡、错误重试、节点发现）  
2、生成类文件多  
3、内容加密处理困难  
4、服务端（监控、权限、过滤器自己实现，通过代理实现类）  


Gaea优点：  
1、自己客户端，提供（动态代理、负载均衡、错误重试、节点发现）  
2、自定义协议和序列化（对象名hashcode标注，维护序列化对象map，内容，变长为加长度，内容），数据加密，代码少   
3、自己服务端，监控、权限、过滤器  
4、链接权限验证  

Gaea缺点：  
1、无IDL文件，各端没有统一生成类代码，后续维护  
2、序列化复杂对象，非基本类型（对象名hashcode标注，内容，变长为加长度，内容），依赖反射，语言扩展难  
3、类型后直接内容，字段无序号，扩展增加字段困难  


结合一下：  
使用Gaea的客户端和服务端（包括网络层），将协议和序列化抽出来，使用thrift的协议和序列化，或者使用protobuf，准备抽时间写一个  

留个待办事项TODO：  
抽时间实现自己的RPC调用框架  
实现客户端（动态代理、负载均衡、错误重试、节点发现）  
只利用thrift协议和序列化，协议可扩展，能使用protobuf  
实现服务端（监控、权限、过滤器）  
