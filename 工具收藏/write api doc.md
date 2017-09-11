编写api文档工具：API-Blueprint

先使用 API Blueprint Language（类似markdown）语法写好api文档。
https://github.com/apiaryio/api-blueprint

https://apiblueprint.org/documentation/


使用：aglio 可以生成html并发布浏览

安装：
```
 npm install -g aglio

```

aglio 是一个可以根据 api-blueprint 的文档生成静态 HTML 页面的工具。
https://github.com/danielgtaylor/aglio

例如：
aglio -i ./hello.md -s &


使用：drakov 可以mock测试
安装：
```
npm install -g drakov
```
提供模拟接口测试
例如：
drakov -f ./hello.md -p 3000

