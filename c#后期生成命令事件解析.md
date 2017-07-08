# c# 后期生成命令事件解析

@(综合)


示例代码如下：

```
REM Copy .config files
xcopy /R /Y /C "$(TargetDir)Configs\*.config" "$(ProjectDir)Configs"
xcopy /R /Y /C "$(TargetDir)Configs\entlib\*.config" "$(ProjectDir)Configs\entlib"

REM Copy BM Common Dlls
xcopy /R /Y /C "$(SolutionDir)bin\$(ConfigurationName)\*.dll" "$(TargetDir)"
xcopy /R /Y /C "$(SolutionDir)bin\$(ConfigurationName)\SH3H.*.pdb" "$(TargetDir)"
xcopy /R /Y /C "$(SolutionDir)bin\$(ConfigurationName)\SH3H*.xml" "$(TargetDir)"
```

**目的**：编译期间，将某些文件夹下config配置文件、dll文件copy到指定目录下





示例代码如下：
```REM Copy .config filesxcopy /R /Y /C &quot;(TargetDir)Configs\*.config&quot; &quot;(ProjectDir)Configs&quot;xcopy /R /Y /C &quot;$(TargetDir)Configs\entlib\*.config&quot; &quot;$(ProjectDir)Configs\entlib&quot;
REM Copy BM Common Dllsxcopy /R /Y /C "$(SolutionDir)bin\$(ConfigurationName)\*.dll" "$(TargetDir)"xcopy /R /Y /C "$(SolutionDir)bin\$(ConfigurationName)\SH3H.*.pdb" "$(TargetDir)"xcopy /R /Y /C "$(SolutionDir)bin\$(ConfigurationName)\SH3H*.xml" "$(TargetDir)"```
**目的**：编译期间，将某些文件夹下config配置文件、dll文件copy到指定目录下
```

### 单点登录原理与简单实现

@(综合)

1. http无状态协议，服务器浏览器共同维护的状态：会话机制

2. 会话机制：浏览器第一次请求服务

   ​

   ​

   ​

   ​

   ​

3. 登录状态，当输入user和pws后，服务器拿到后会去数据库比对，正确的话，应该会将这个会话标记为‘已登录’等之类的状态，既然是会话的状态，自然要保存在会话对象中,例如`HttpSession session=request.getSession;session.setAttribute('isLogin',true)`;用户在此访问时，服务器端会在会话对象中查看登录状态。
   ![Alt text](./1480571789025.png)

每次请求受保护资源时，都会检查会话对象中的登录状态，只有isLogin=true的会话才能访问，登录机制因此而实现。


### 多系统的复杂性

web系统由单系统发展成多系统组成的应用群，复杂性应该由系统内部承担，而不是用户。

`单系统登录解决方案的核心是cookie,`cookie携带会话id在浏览器与服务器之间维护会话状态。但cookie是由限制的，这个限制就是cookie的域，浏览器发送http请求时会自动携带与该域匹配的cookie,而不是所有的cookie
![Alt text](./1480572164729.png)

既然这样，为什么不将web应用群中所有子系统的域名统一在一个顶级域名下，例如“*.baidu.com”，然后将它们的cookie域设置为“baidu.com”，这种做法理论上是可以的，甚至早期很多多系统登录就采用这种同域名共享cookie的方式。

然而，可行并不代表好，**共享cookie的方式存在众多局限**。首先，应用群域名得统一；其次，`应用群各系统使用的技术（至少是web服务器）要相同`，不然cookie的key值（tomcat为JSESSIONID）不同，无法维持会话，共享cookie的方式是无法实现跨语言技术平台登录的，比如java、php、.net系统之间；第三，`cookie本身不安全`。

因此，我们需要一种全新的登录方式来实现多系统应用群的登录，这就是`单点登录`:SSO:是指在多系统应用群中登录一个系统，便可在其他所有系统中得到授权而无需再次登录

1. 登录

相比于单系统登录，sso需要一个独立的认证中心，只有认证中西能接受用户的用户名密码等安全信息，其他系统不提供登录入口，只接受认证中心的间接授权。间接授权通过令牌实现，sso认证中心验证用户的用户名密码没问题，创建授权令牌，在接下来的跳转过程中，授权令牌作为参数发送各子系统，子系统拿到令牌，即得到了授权，可以借此创建局部会话，局部会话登录方式与单系统的登录方式相同。这个过程，也就是单点登录的原理。
![Alt text](./1480572788475.png)

**下面对上图简要描述，对单点登录很清晰的理解**
用户访问系统1的受保护资源，系统1发现用户未登录，跳转至sso认证中心，并将自己的地址作为参数 sso认证中心发现用户未登录，将用户引导至登录页面 用户输入用户名密码提交登录申请 sso认证中心校验用户信息，创建用户与sso认证中心之间的会话，称为全局会话，同时创建授权令牌 sso认证中心带着令牌跳转会最初的请求地址（系统1） 系统1拿到令牌，去sso认证中心校验令牌是否有效 sso认证中心校验令牌，返回有效，注册系统1 系统1使用该令牌创建与用户的会话，称为局部会话，返回受保护资源 用户访问系统2的受保护资源 系统2发现用户未登录，跳转至sso认证中心，并将自己的地址作为参数 sso认证中心发现用户已登录，跳转回系统2的地址，并附上令牌 系统2拿到令牌，去sso认证中心校验令牌是否有效 sso认证中心校验令牌，返回有效，注册系统2 系统2使用该令牌创建与用户的局部会话，返回受保护资源
用户登录成功之后，会与sso认证中心及各个子系统建立会话，用户与sso认证中心建立的会话称为全局会话，用户与各个子系统建立的会话称为局部会话，局部会话建立之后，用户访问子系统受保护资源将不再通过sso认证中心，全局会话与局部会话有如下约束关系
局部会话存在，全局会话一定存在 全局会话存在，局部会话不一定存在 全局会话销毁，局部会话必须销毁
你可以通过博客园、百度、csdn、淘宝等网站的登录过程加深对单点登录的理解，注意观察登录过程中的跳转url与参数

2. 注销

单点登录自然也要单点注销，在一个子系统中注销，所有子系统的会话都将被销毁，用下面的图来说明

![Alt text](./1480573165915.png)

sso认证中心一直监听全局会话的状态，一旦全局会话销毁，监听器将通知所有注册系统执行注销操作

### 四，部署图
单点登录涉及sso认证中心与众子系统，子系统与sso认证中心需要通信以交换令牌、校验令牌及发起注销请求，因而子系统必须集成sso的客户端，sso认证中心则是sso服务端，整个单点登录过程实质是sso客户端与服务端通信的过程，用下图描述
![Alt text](./1480573343889.png)







​	



