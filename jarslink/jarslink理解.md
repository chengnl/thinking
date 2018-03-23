##jarslink  
JarsLink (原名Titan) 是一个基于JAVA的模块化开发框架，它提供在运行时动态加载模块（一个JAR包）、卸载模块和模块间调用的API。

##初步理解

1. 提供模块化加载，动态更新模块功能，适用扩展性比较高，单功能增加频繁的业务。
2. 调用逻辑
   module注册>>调用方请求参数module和action>>执行doaction>>调用module对应的action的excute方法>>返回结果

3. 完全使用需要完善：
   - 提供模块化jar发布包的上传、发布、删除管理界面，上传jar包，发布调用moudle加载，注册，删除调用注销
   - 提供启动加载所有模块化包的初始化操作，要求模块化的名称，规范一致可识别为需要加载的模块jar包。
   - 请求统一参数追加module、action 来识别调用那个模块，那个action
   - 返回的对象最好通用，这样调用方可以统一规范解析
   

## 深入理解

### moduleload 模块加载

- ImmutableList

```
   moduleConfig.setModuleUrl(ImmutableList.of(demoModule));
```
ImmutableList  不可变list，防止修改

- classpath

```
  public static String SPRING_XML_PATTERN  = "classpath*:META-INF/spring/*.xml";
```
load path：  
classpath和classpath*区别： 
classpath：只会到你的class路径中查找找文件。
classpath*：不仅包含class路径，还包括jar文件中（class路径）进行查找。



##jarslink 地址  
[jarslink 地址](https://github.com/alibaba/jarslink/)
