![](/assets/spring-cloud-arch1.png)

![](/assets/spring-cloud-arch2.png)

![](/assets/import.png)![](/assets/import2.png)

eBay 架构分为清晰的四个抽象层次：

  


* **Infrastructure：** 底层基础设施，包括云计算，数据中心，计算 / 网络 / 存储，各种工具和监控等，国内公司一般把这一层称为运维层。

* **Platform Services：** 平台服务层，主要是一些框架中间件服务，包括应用和服务框架，数据访问层，表示层，消息系统，任务调度和开发者工具等等，国内公司一般把这一层称为基础框架或基础架构层。

* **Commerce Services：** 电商服务层，eBay 作为电子商务平台多年沉淀下来的核心领域服务，相当于微服务业务层，包括登录认证，分类搜索，购物车，送货和客服等等。

* **Applications：** 应用层，也称用户体验 + 渠道层，包括 eBay 主站，移动端 app，第三方接入渠道等。



![](/assets/import3.png)

