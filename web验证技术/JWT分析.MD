#JWT技术分析
##JWT原理
JWT是一种用于双方之间传递安全信息的简洁的、URL安全的表述性声明规范，具体协议见：
[RFC 7519](https://tools.ietf.org/html/rfc7519)

比较好的技术博文：
http://blog.leapoahead.com/2015/09/06/understanding-jwt/
http://blog.leapoahead.com/2015/09/07/user-authentication-with-jwt/

##安全性思考
因为JWT协议里面显示指定算法，思考一个问题，如果服务端秘钥固定，客户端得到数据做暴力破解
等到秘钥后，模拟签名发送处理，安全问题。算法采用RSA公私钥(私钥签名，公钥验证)，安全问题解决，如果是HS256呢？

分析下HS256下增强安全：
1、签名算法不显示指定
   增加了客户端破解难度，不知道使用的那种HASH算法
2、秘钥不固定，跟对象有关
   秘钥规则：固定长度串+Payload中字段某个值
   这样秘钥每次都会不一样，就算破解也仅仅只是单次用户的被破解，所以产生秘钥的规则很重要，不能被猜测出来
   可以加大猜测难度，比如在拼接的基础上做二次编码，这样加大猜测难度。

##目前单点系统分析

###原有单点登录设计

![原单点登录][1]
[1]: https://github.com/chengnl/thinking/tree/master/images-folder/sso.png "原单点登录交互"

原有的单点登录ticket生成思路：信息拼接后，对称加密算法加密得出ticket

验证ticket逻辑：  
业务系统调用sso验证url，验证ticket有效性，主要两方面验证，ticket是否存在，ticket能否解密以及解密后是否时效判断

目前此方式发现的不足：  
1、加密处理（对称加密方式），而且是通过能否解密验证信息完整性，用户基本信息在ticket中，基于安全考虑，密钥在SSO方，导致只能在SSO方验证解析  
2、需要调用sso验证url，增加网络请求，如果是https，负担更重。  
优点是所有的验证经过单点登录，单点登录的某个业务系统退出或者单点登录系统挂了，ticket删除或者丢失，所有的业务系统必须重新登录验证。   

###基于JWT改造后设计  

![jwt单点登录][2]
[2]: https://github.com/chengnl/thinking/tree/master/images-folder/jwt.png "jwt单点登录"

验证ticket逻辑：  
创建ticket通过sso的私钥签名，业务系统本地根据公钥验证jwt的有效性，解决两个问题票的存在性、票的有效性，sso不用存储ticket信息

另外用户基本信息是明文，业务系统可以直接得到，不用依靠sso方  

也就是优点是：
1、使用公私钥加密更加安全
2、业务系统不用每次请求都调用网络请求验证票有效性，获取用户信息也不用调用sso方获取

唯一缺点，单点登录系统挂了，业务系统还能继续使用，只能等到ticket本身时效性失效，业务系统退出可以调用sso删除浏览器cookie中的ticket


