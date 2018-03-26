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
   

## 代码理解
   ModuleLoaderImpl 
   1. 获取模块加载器：扩展URLClassLoader，作用过滤一些排除的包，需要加载器加载的包
   2. 构建ModuleApplicationContext，扩展：ClassPathXmlApplicationContext 支持增加模块属性，使用当前线程切换ClassLoader解决双亲委派的问题。
   3. moduleApplicationContext.refresh()实现模块bean加载
   4. 返回module：SpringModule对象，该对象提供模块的实现Action类提取，以及doaction的execute执行处理，destroy处理；不太理解doaction使用module的类加载器，获取action后直接execute不用？？？


   ModuleManagerImpl
   1. 模块的管理：模块的查找、注册、删除、销毁处理。

   ModuleServiceImpl
   1. 提供module加载和module管理操作。

   AbstractModuleRefreshScheduler
   1. 实现模块刷新处理，外部加载或者更新的与ModuleManager内部管理比对，新增、版本不同的更新、多余删除处理
  


### 代码比较优雅
- 使用大量的google类库方便操作
  1. 不可变集合：ImmutableList，集合比对：Maps.difference，集合过滤：Collections2.filter，集合转换：Maps.uniqueIndex、Maps.transformValues等等
- 继承ClassPathXmlApplicationContext实现自己的Properties加载；继承URLClassLoader实现自己类过滤
- 使用spring ApplicationContext加载，或者bean类操作 
- AbstractModuleRefreshScheduler implements InitializingBean 实现加载后动作。

- classpath

```
  public static String SPRING_XML_PATTERN  = "classpath*:META-INF/spring/*.xml";
```
load path：  
classpath和classpath*区别： 
classpath：只会到你的class路径中查找找文件。
classpath*：不仅包含class路径，还包括jar文件中（class路径）进行查找。

### 实际运用
需要将action的请求和返回结果，使用通用对象处理，因为在执行时使用了两个classloader，一个是当前线程的，一个是module的，确保请求参数和返回参数对象类型在父classloader（当前线程）中才能在后续使用。

如果仅仅在module中定义参数和返回结果类型，实际是无法使用的。需要使用父类的对象，这样action的execute才能顺利执行





##jarslink 地址  
[jarslink 地址](https://github.com/alibaba/jarslink/)
