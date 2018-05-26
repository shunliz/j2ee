###### 从零开始学CAS单点登录——实例Demo

2017年04月05日 16:40:35

阅读数：6443

首先说明的是，本文提供的是一个快速上手的实例，并不打算详细阐述单点登录的概念和CAS的工作原理。这也是本人平时学习的习惯：不管什么技术，先把环境搭起来，从最简单的Hello World逐步到其背后的Why-What-How。

---

### **一、准备工作** {#一准备工作}

单点，至少我们得跨域吧，所以我这里部署了三个系统到三个Tomcat下：

| 应用 | 域名 | Tomcat端口 | 作用 | 来源 |
| :--- | :--- | :--- | :--- | :--- |
| CAS Server | cas-server.com | 38080 | 中央认证服务 | 官网下载 |
| web-one | cas-app1.com | 18080 | 接入系统1 | 自定义 |
| web-one | cas-app2.com | 28080 | 接入系统2 | 自定义 |

实现不同域名的模拟，只需配置hosts文件：  
**\System32\drivers\etc\hosts**

```
127.0
.0
.1
   cas-app1
.com
127.0
.0
.1
   cas-app2
.com
127.0
.0
.1
   cas-server
.com

```

---

### **二、CAS Server部署** {#二cas-server部署}

**1 下载CAS Server**  
下载地址：[https://www.apereo.org/projects/cas/download-cas](https://www.apereo.org/projects/cas/download-cas)  
当前使用的是3.5.0版本

**2 获取CAS Server部署包**  
使用cas server的方法有两种：  
1. 直接将解压后文件夹下的modules文件夹中的cas-server-webapp-3.5.0.war到Tomcat中；  
2. Eclipse中导入cas-server-3.5.0多模块maven项目，在Eclipse中启动cas-server-webapp这个web项目

> 不管用哪种方式，都是一样的，只是cas帮我们打好了包而已，第2种导入Eclipse便于我们调试和修改代码。

**3 修改配置**  
1 WEB-INF/cas.properties  
只需修改cas server部署的Tomcat的访问路径即可：

```
server
.name=http:
//cas-server:38080

```

2 WEB-INF/deployerConfigContext.xml  
为了简单起见，我们不启用HTTPS，添加requireSecure为false：

```
<
bean 
class
=
"org.jasig.cas.authentication.handler.support.HttpBasedServiceCredentialsAuthenticationHandler"

                    p:httpClient-
ref
=
"httpClient"
 p:requireSecure=
"false"
 /
>

```

3 ticketGrantingTicketCookieGenerator.xml  
同样添加cookieSecure为false：

```
<
bean 
id
=
"ticketGrantingTicketCookieGenerator"
class
=
"org.jasig.cas.web.support.CookieRetrievingCookieGenerator"

        p:cookieSecure=
"false"

        p:cookieMaxAge=
"-1"

        p:cookieName=
"CASTGC"

        p:cookiePath=
"/cas"
 /
>

```

4 warnCookieGenerator.xml  
同样添加cookieSecure为false：

```
<
bean 
id
=
"warnCookieGenerator"
class
=
"org.jasig.cas.web.support.CookieRetrievingCookieGenerator"

        p:cookieSecure=
"false"

        p:cookieMaxAge=
"-1"

        p:cookieName=
"CASPRIVACY"

        p:cookiePath=
"/cas"
 /
>

```

**4 启动Tomcat**  
修改配置后，我们就可以启动部署了CAS Server的Tomcat了。  
前面配置了hosts文件，直接访问`http://cas-server.com:38080/cas/`，不出意外我们会看到默认的登录页面：  
![](https://img-blog.csdn.net/20170405165942511?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcG9zdG51bGw=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")  
因为cas server默认使用的给我们演示用的认证处理器，我们只需输入账号密码都是相同的字符串，即表示登录凭据是正确的，如我们输入”test/test”，就能看到登录成功的页面：  
![](https://img-blog.csdn.net/20170405170536107?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcG9zdG51bGw=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")

到此为止，我们完成了CAS Server的部署，也就是认证中心已经部署好了。

---

### **三、业务系统部署** {#三业务系统部署}

**1 新建项目**  
用Eclipse快速新建两个maven的webapp项目

**2 配置项目**  
1 pom.xml  
整合cas，肯定得用到cas的代码吧，所以在pom中引入cas客户端：

```
<
dependency
>
<
groupId
>
org.jasig.cas.client
<
/
groupId
>
<
artifactId
>
cas-client-core
<
/
artifactId
>
<
version
>
3.2.1
<
/
version
>
<
/
dependency
>
<
dependency
>
<
groupId
>
log4j
<
/
groupId
>
<
artifactId
>
log4j
<
/
artifactId
>
<
version
>
1.2.16
<
/
version
>
<
/
dependency
>

```

2 web.xml  
要登录拦截，肯定得靠过滤器，所以在web.xml中配置一系列的Filter。具体哪些Filter可以暂时不关心，只需配置几个URL即可，serverName就是我们当前的接入系统，casServer就是CAS Server认证中心：

```
<
!-- ======================== start ======================== --
>
<
listener
>
<
listener-class
>
org.jasig.cas.client.session.SingleSignOutHttpSessionListener
<
/
listener-class
>
<
/
listener
>
<
filter
>
<
filter-name
>
CAS Single Sign Out Filter
<
/
filter-name
>
<
filter-class
>
org.jasig.cas.client.session.SingleSignOutFilter
<
/
filter-class
>
<
/
filter
>
<
filter-mapping
>
<
filter-name
>
CAS Single Sign Out Filter
<
/
filter-name
>
<
url-pattern
>
/*
<
/
url-pattern
>
<
/
filter-mapping
>
<
filter
>
<
filter-name
>
CAS Filter
<
/
filter-name
>
<
filter-class
>
org.jasig.cas.client.authentication.AuthenticationFilter
<
/
filter-class
>
<
init-param
>
<
param-name
>
casServerLoginUrl
<
/
param-name
>
<
param-value
>
http://cas-server.com:38080/cas/login
<
/
param-value
>
<
/
init-param
>
<
init-param
>
<
param-name
>
serverName
<
/
param-name
>
<
param-value
>
http://cas-app1.com:18080/
<
/
param-value
>
<
/
init-param
>
<
/
filter
>
<
filter-mapping
>
<
filter-name
>
CAS Filter
<
/
filter-name
>
<
url-pattern
>
/*
<
/
url-pattern
>
<
/
filter-mapping
>
<
filter
>
<
filter-name
>
CAS Validation Filter
<
/
filter-name
>
<
filter-class
>

        org.jasig.cas.client.validation.Cas20ProxyReceivingTicketValidationFilter
    
<
/
filter-class
>
<
init-param
>
<
param-name
>
casServerUrlPrefix
<
/
param-name
>
<
param-value
>
http://cas-server.com:38080/cas/
<
/
param-value
>
<
/
init-param
>
<
init-param
>
<
param-name
>
serverName
<
/
param-name
>
<
param-value
>
http://cas-app1.com:18080
<
/
param-value
>
<
/
init-param
>
<
/
filter
>
<
filter-mapping
>
<
filter-name
>
CAS Validation Filter
<
/
filter-name
>
<
url-pattern
>
/*
<
/
url-pattern
>
<
/
filter-mapping
>
<
filter
>
<
filter-name
>
CAS HttpServletRequest Wrapper Filter
<
/
filter-name
>
<
filter-class
>

        org.jasig.cas.client.util.HttpServletRequestWrapperFilter
    
<
/
filter-class
>
<
/
filter
>
<
filter-mapping
>
<
filter-name
>
CAS HttpServletRequest Wrapper Filter
<
/
filter-name
>
<
url-pattern
>
/*
<
/
url-pattern
>
<
/
filter-mapping
>
<
filter
>
<
filter-name
>
CAS Assertion Thread Local Filter
<
/
filter-name
>
<
filter-class
>
org.jasig.cas.client.util.AssertionThreadLocalFilter
<
/
filter-class
>
<
/
filter
>
<
filter-mapping
>
<
filter-name
>
CAS Assertion Thread Local Filter
<
/
filter-name
>
<
url-pattern
>
/*
<
/
url-pattern
>
<
/
filter-mapping
>
<
!-- ======================== end ======================== --
>

```

**3 启动Tomcat**  
访问`http://cas-app1.com:18080/web-one/`,不出所料的话，应该会自动重定向到cas登录页面：  
![](https://img-blog.csdn.net/20170405173203918?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcG9zdG51bGw=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")

> 尤其注意这里的URL。

随便输入账号密码相同做登录，即可认证通过成功访问web1的页面：  
![](https://img-blog.csdn.net/20170405173454009?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcG9zdG51bGw=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")

然后访问`http://cas-app2.com:28080/web-two/`，不会重定向到登录页面，直接到web2的页面：  
![](https://img-blog.csdn.net/20170405173800451?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcG9zdG51bGw=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")

访问cas认证中心`cas-server.com:38080/cas/`，也是直接到登录成功的页面：  
![](https://img-blog.csdn.net/20170405173928093?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcG9zdG51bGw=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")

如果要单点注销登录，直接访问`http://cas-server.com:38080/cas/logout`：  
![](https://img-blog.csdn.net/20170405180049762?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcG9zdG51bGw=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "这里写图片描述")

**到此为止，三个系统的最最基本的单点登录功能整合完成了。**

---

### **四、还有什么问题** {#四还有什么问题}

1.没有使用https  
2.登录页面的风格是cas，如何自定义  
3.账号密码不是从数据库中读取的  
4.这里只有登录凭据认证，权限该如何处理？

