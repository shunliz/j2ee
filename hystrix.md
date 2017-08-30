一、认识Hystrix  


Hystrix是Netflix开源的一款容错框架，包含常用的容错方法：线程池隔离、信号量隔离、熔断、降级回退。在高并发访问下，系统所依赖的服务的稳定性对系统的影响非常大，依赖有很多不可控的因素，比如网络连接变慢，资源突然繁忙，暂时不可用，服务脱机等。我们要构建稳定、可靠的分布式系统，就必须要有这样一套容错方法。  
本文将逐一分析线程池隔离、信号量隔离、熔断、降级回退这四种技术的原理与实践。

二、线程隔离

2.1为什么要做线程隔离

比如我们现在有3个业务调用分别是查询订单、查询商品、查询用户，且这三个业务请求都是依赖第三方服务-订单服务、商品服务、用户服务。三个服务均是通过RPC调用。当查询订单服务，假如线程阻塞了，这个时候后续有大量的查询订单请求过来，那么容器中的线程数量则会持续增加直致CPU资源耗尽到100%，整个服务对外不可用，集群环境下就是雪崩。如下图

  


![](http://mmbiz.qpic.cn/mmbiz_png/3xsFRgx4kHol84FkG4pPTnP58fUjXNc6MJ8lkzibgTtzML5hDoc1nyVice1DF25CFIHXMcRuUZ0KGL98VAK47mag/0.png?tp=webp&wxfrom=5&wx_lazy=1)

订单服务不可用.png

：

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

整个tomcat容器不可用.png

  


2.2、线程隔离-线程池

2.2.1、Hystrix是如何通过线程池实现线程隔离的

Hystrix通过命令模式，将每个类型的业务请求封装成对应的命令请求，比如查询订单-&gt;订单Command，查询商品-&gt;商品Command，查询用户-&gt;用户Command。每个类型的Command对应一个线程池。创建好的线程池是被放入到ConcurrentHashMap中，比如查询订单：

final static ConcurrentHashMap&lt;String, HystrixThreadPool&gt; threadPools = new ConcurrentHashMap&lt;String, HystrixThreadPool&gt;\(\);

threadPools.put\(“hystrix-order”, new HystrixThreadPoolDefault\(threadPoolKey, propertiesBuilder\)\);

当第二次查询订单请求过来的时候，则可以直接从Map中获取该线程池。具体流程如下图：

![](http://mmbiz.qpic.cn/mmbiz_png/3xsFRgx4kHol84FkG4pPTnP58fUjXNc6BaqQZSWh6dZ9PQo5T16u4Tc09BKUMMHFicgdLQMaPhJaibTE5QANp78g/0.png?tp=webp&wxfrom=5&wx_lazy=1)

hystrix线程执行过程和异步化.png

创建线程池中的线程的方法，查看源代码如下：

![](http://mmbiz.qpic.cn/mmbiz_png/3xsFRgx4kHol84FkG4pPTnP58fUjXNc6umBylZn1Mib3olJB1ichTo3VcGM04DSyODR1f6wgWO84moK8EFkxcWow/0.png?tp=webp&wxfrom=5&wx_lazy=1)

创建线程池中的线程.png

  


执行Command的方式一共四种，直接看官方文档\(https://github.com/Netflix/Hystrix/wiki/How-it-Works\)，具体区别如下：

•execute\(\)：以同步堵塞方式执行run\(\)。调用execute\(\)后，hystrix先创建一个新线程运行run\(\)，接着调用程序要在execute\(\)调用处一直堵塞着，直到run\(\)运行完成。

•queue\(\)：以异步非堵塞方式执行run\(\)。调用queue\(\)就直接返回一个Future对象，同时hystrix创建一个新线程运行run\(\)，调用程序通过Future.get\(\)拿到run\(\)的返回结果，而Future.get\(\)是堵塞执行的。

• observe\(\)：事件注册前执行run\(\)/construct\(\)。第一步是事件注册前，先调用observe\(\)自动触发执行run\(\)/construct\(\)（如果继承的是HystrixCommand，hystrix将创建新线程非堵塞执行run\(\)；如果继承的是HystrixObservableCommand，将以调用程序线程堵塞执行construct\(\)），第二步是从observe\(\)返回后调用程序调用subscribe\(\)完成事件注册，如果run\(\)/construct\(\)执行成功则触发onNext\(\)和onCompleted\(\)，如果执行异常则触发onError\(\)。

•toObservable\(\)：事件注册后执行run\(\)/construct\(\)。第一步是事件注册前，调用toObservable\(\)就直接返回一个Observable&lt;String&gt;对象，第二步调用subscribe\(\)完成事件注册后自动触发执行run\(\)/construct\(\)（如果继承的是HystrixCommand，hystrix将创建新线程非堵塞执行run\(\)，调用程序不必等待run\(\)；如果继承的是HystrixObservableCommand，将以调用程序线程堵塞执行construct\(\)，调用程序等待construct\(\)执行完才能继续往下走），如果run\(\)/construct\(\)执行成功则触发onNext\(\)和onCompleted\(\)，如果执行异常则触发onError\(\)  
注：  
execute\(\)和queue\(\)是HystrixCommand中的方法，observe\(\)和toObservable\(\)是HystrixObservableCommand 中的方法。从底层实现来讲，HystrixCommand其实也是利用Observable实现的（如果我们看Hystrix的源码的话，可以发现里面大量使用了RxJava），虽然HystrixCommand只返回单个的结果，但HystrixCommand的queue方法实际上是调用了toObservable\(\).toBlocking\(\).toFuture\(\)，而execute方法实际上是调用了queue\(\).get\(\)。

2.2.2、如何应用到实际代码中

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

线程池实际代码使用.png

  


2.2.3、线程隔离-线程池小结

执行依赖代码的线程与请求线程\(比如Tomcat线程\)分离，请求线程可以自由控制离开的时间，这也是我们通常说的异步编程，Hystrix是结合RxJava来实现的异步编程。通过设置线程池大小来控制并发访问量，当线程饱和的时候可以拒绝服务，防止依赖问题扩散。

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

线程隔离.png

  


线程池隔离的优点:  
\[1\]:应用程序会被完全保护起来，即使依赖的一个服务的线程池满了，也不会影响到应用程序的其他部分。  
\[2\]:我们给应用程序引入一个新的风险较低的客户端lib的时候，如果发生问题，也是在本lib中，并不会影响到其他内容，因此我们可以大胆的引入新lib库。  
\[3\]:当依赖的一个失败的服务恢复正常时，应用程序会立即恢复正常的性能。  
\[4\]:如果我们的应用程序一些参数配置错误了，线程池的运行状况将会很快显示出来，比如延迟、超时、拒绝等。同时可以通过动态属性实时执行来处理纠正错误的参数配置。  
\[5\]:如果服务的性能有变化，从而需要调整，比如增加或者减少超时时间，更改重试次数，就可以通过线程池指标动态属性修改，而且不会影响到其他调用请求。  
\[6\]:除了隔离优势外，hystrix拥有专门的线程池可提供内置的并发功能，使得可以在同步调用之上构建异步的外观模式，这样就可以很方便的做异步编程（Hystrix引入了Rxjava异步框架）。

尽管线程池提供了线程隔离，我们的客户端底层代码也必须要有超时设置，不能无限制的阻塞以致线程池一直饱和。

线程池隔离的缺点:  
\[1\]:线程池的主要缺点就是它增加了计算的开销，每个业务请求（被包装成命令）在执行的时候，会涉及到请求排队，调度和上下文切换。不过Netflix公司内部认为线程隔离开销足够小，不会产生重大的成本或性能的影响。

The Netflix API processes 10+ billion Hystrix Command executions per day using thread isolation. Each API instance has 40+ thread-pools with 5–20 threads in each \(most are set to 10\).  
Netflix API每天使用线程隔离处理10亿次Hystrix Command执行。 每个API实例都有40多个线程池，每个线程池中有5-20个线程（大多数设置为10个）。

对于不依赖网络访问的服务，比如只依赖内存缓存这种情况下，就不适合用线程池隔离技术，而是采用信号量隔离。

2.3、线程隔离-信号量

2.3.1、线程池和信号量的区别

上面谈到了线程池的缺点，当我们依赖的服务是极低延迟的，比如访问内存缓存，就没有必要使用线程池的方式，那样的话开销得不偿失，而是推荐使用信号量这种方式。下面这张图说明了线程池隔离和信号量隔离的主要区别：线程池方式下业务请求线程和执行依赖的服务的线程不是同一个线程；信号量方式下业务请求线程和执行依赖服务的线程是同一个线程

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

信号量和线程池的区别.png

  


2.3.2、如何使用信号量来隔离线程

将属性execution.isolation.strategy设置为SEMAPHORE ，象这样 ExecutionIsolationStrategy.SEMAPHORE，则Hystrix使用信号量而不是默认的线程池来做隔离。

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

信号量使用.png

  


2.3.4、线程隔离-信号量小结

信号量隔离的方式是限制了总的并发数，每一次请求过来，请求线程和调用依赖服务的线程是同一个线程，那么如果不涉及远程RPC调用（没有网络开销）则使用信号量来隔离，更为轻量，开销更小。

三、熔断

3.1、熔断器\(Circuit Breaker\)介绍

熔断器，现实生活中有一个很好的类比，就是家庭电路中都会安装一个保险盒，当电流过大的时候保险盒里面的保险丝会自动断掉，来保护家里的各种电器及电路。Hystrix中的熔断器\(Circuit Breaker\)也是起到这样的作用，Hystrix在运行过程中会向每个commandKey对应的熔断器报告成功、失败、超时和拒绝的状态，熔断器维护计算统计的数据，根据这些统计的信息来确定熔断器是否打开。如果打开，后续的请求都会被截断。然后会隔一段时间默认是5s，尝试半开，放入一部分流量请求进来，相当于对依赖服务进行一次健康检查，如果恢复，熔断器关闭，随后完全恢复调用。如下图：

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

熔断器开关图.png

  


说明，上面说的commandKey，就是在初始化的时候设置的andCommandKey\(HystrixCommandKey.Factory.asKey\("testCommandKey"\)\)

再来看下熔断器在整个Hystrix流程图中的位置，从步骤4开始，如下图：

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

Hystrix流程图.png

  


Hystrix会检查Circuit Breaker的状态。如果Circuit Breaker的状态为开启状态，Hystrix将不会执行对应指令，而是直接进入失败处理状态（图中8 Fallback）。如果Circuit Breaker的状态为关闭状态，Hystrix会继续进行线程池、任务队列、信号量的检查（图中5）

3.2、如何使用熔断器\(Circuit Breaker\)

由于Hystrix是一个容错框架，因此我们在使用的时候，要达到熔断的目的只需配置一些参数就可以了。但我们要达到真正的效果，就必须要了解这些参数。Circuit Breaker一共包括如下6个参数。  
1、circuitBreaker.enabled  
是否启用熔断器，默认是TURE。  
2、circuitBreaker.forceOpen  
熔断器强制打开，始终保持打开状态。默认值FLASE。  
3、circuitBreaker.forceClosed  
熔断器强制关闭，始终保持关闭状态。默认值FLASE。  
4、circuitBreaker.errorThresholdPercentage  
设定错误百分比，默认值50%，例如一段时间（10s）内有100个请求，其中有55个超时或者异常返回了，那么这段时间内的错误百分比是55%，大于了默认值50%，这种情况下触发熔断器-打开。  
5、circuitBreaker.requestVolumeThreshold  
默认值20.意思是至少有20个请求才进行errorThresholdPercentage错误百分比计算。比如一段时间（10s）内有19个请求全部失败了。错误百分比是100%，但熔断器不会打开，因为requestVolumeThreshold的值是20. 这个参数非常重要，熔断器是否打开首先要满足这个条件，源代码如下

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

熔断器打开先后条件判断.png

  


6、circuitBreaker.sleepWindowInMilliseconds  
半开试探休眠时间，默认值5000ms。当熔断器开启一段时间之后比如5000ms，会尝试放过去一部分流量进行试探，确定依赖服务是否恢复。

测试代码（模拟10次调用，错误百分比为5%的情况下，打开熔断器开关。）：

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

熔断器实际使用代码1.png

  


![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

熔断器实际使用代码2.png

  


测试结果：

call times:1 result:fallback: isCircuitBreakerOpen: false  
call times:2 result:running: isCircuitBreakerOpen: false  
call times:3 result:running: isCircuitBreakerOpen: false  
call times:4 result:fallback: isCircuitBreakerOpen: false  
call times:5 result:running: isCircuitBreakerOpen: false  
call times:6 result:fallback: isCircuitBreakerOpen: false  
call times:7 result:fallback: isCircuitBreakerOpen: false  
call times:8 result:fallback: isCircuitBreakerOpen: false  
call times:9 result:fallback: isCircuitBreakerOpen: false  
call times:10 result:fallback: isCircuitBreakerOpen: false  
熔断器打开  
call times:11 result:fallback: isCircuitBreakerOpen: true  
call times:12 result:fallback: isCircuitBreakerOpen: true  
call times:13 result:fallback: isCircuitBreakerOpen: true  
call times:14 result:fallback: isCircuitBreakerOpen: true  
call times:15 result:fallback: isCircuitBreakerOpen: true  
call times:16 result:fallback: isCircuitBreakerOpen: true  
call times:17 result:fallback: isCircuitBreakerOpen: true  
call times:18 result:fallback: isCircuitBreakerOpen: true  
call times:19 result:fallback: isCircuitBreakerOpen: true  
call times:20 result:fallback: isCircuitBreakerOpen: true  
5s后熔断器关闭  
call times:21 result:running: isCircuitBreakerOpen: false  
call times:22 result:running: isCircuitBreakerOpen: false  
call times:23 result:fallback: isCircuitBreakerOpen: false  
call times:24 result:running: isCircuitBreakerOpen: false  
call times:25 result:running: isCircuitBreakerOpen: false

  


3.3、熔断器\(Circuit Breaker\)源代码HystrixCircuitBreaker.java分析

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

HystrixCircuitBreaker.java.png

  


Factory 是一个工厂类，提供HystrixCircuitBreaker实例

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

Factory源码解析.png

  


HystrixCircuitBreakerImpl是HystrixCircuitBreaker的实现，allowRequest\(\)、isOpen\(\)、markSuccess\(\)都会在HystrixCircuitBreakerImpl有默认的实现。

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

HystrixCircuitBreakerImpl-1.png

  


![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

HystrixCircuitBreakerImpl-allowSingleTest\(\).png  


  


![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

HystrixCircuitBreakerImpl-isOpen\(\).png

  


3.4、熔断器小结

每个熔断器默认维护10个bucket,每秒一个bucket,每个blucket记录成功,失败,超时,拒绝的状态，默认错误超过50%且10秒内超过20个请求进行中断拦截。下图显示HystrixCommand或HystrixObservableCommand如何与HystrixCircuitBreaker及其逻辑和决策流程进行交互，包括计数器在断路器中的行为。

四、回退降级

4.1、降级

所谓降级，就是指在在Hystrix执行非核心链路功能失败的情况下，我们如何处理，比如我们返回默认值等。如果我们要回退或者降级处理，代码上需要实现HystrixCommand.getFallback\(\)方法或者是HystrixObservableCommand. HystrixObservableCommand\(\)。

![](http://mmbiz.qpic.cn/mmbiz_png/3xsFRgx4kHol84FkG4pPTnP58fUjXNc6uiapiaxh4TpbnogKoqZMCLGxEz2rEGxgjgCymFN0RNsThSLc8Iy7Hwmg/0.png?tp=webp&wxfrom=5&wx_lazy=1)

CommandHelloFailure-降级.png

4.2、Hystrix的降级回退方式

Hystrix一共有如下几种降级回退模式：

4.2.1、Fail Fast 快速失败

@Override

   protected String run\(\) {

       if \(throwException\) {

           throw new RuntimeException\("failure from CommandThatFailsFast"\);

       } else {

           return "success";

       }

   }

如果我们实现的是HystrixObservableCommand.java则 重写 resumeWithFallback方法

@Override

   protected Observable&lt;String&gt; resumeWithFallback\(\) {

       if \(throwException\) {

           return Observable.error\(new Throwable\("failure from CommandThatFailsFast"\)\);

       } else {

           return Observable.just\("success"\);

       }

   }

4.2.2、Fail Silent 无声失败

返回null，空Map，空List

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

fail silent.png

@Override

   protected String getFallback\(\) {

       return null;

   }

@Override

   protected List&lt;String&gt; getFallback\(\) {

       return Collections.emptyList\(\);

   }

@Override

   protected Observable&lt;String&gt; resumeWithFallback\(\) {

       return Observable.empty\(\);

   }

4.2.3、Fallback: Static 返回默认值

回退的时候返回静态嵌入代码中的默认值，这样就不会导致功能以Fail Silent的方式被清楚，也就是用户看不到任何功能了。而是按照一个默认的方式显示。

@Override

   protected Boolean getFallback\(\) {

       return true;

   }

@Override

   protected Observable&lt;Boolean&gt; resumeWithFallback\(\) {

       return Observable.just\( true \);

   }

4.2.4、Fallback: Stubbed 自己组装一个值返回

当我们执行返回的结果是一个包含多个字段的对象时，则会以Stubbed 的方式回退。Stubbed 值我们建议在实例化Command的时候就设置好一个值。以countryCodeFromGeoLookup为例，countryCodeFromGeoLookup的值，是在我们调用的时候就注册进来初始化好的。CommandWithStubbedFallback command = new CommandWithStubbedFallback\(1234, "china"\);主要代码如下：

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

CommandWithStubbedFallback.png

4.2.5、Fallback: Cache via Network 利用远程缓存

通过远程缓存的方式。在失败的情况下再发起一次remote请求，不过这次请求的是一个缓存比如redis。由于是又发起一起远程调用，所以会重新封装一次Command，这个时候要注意，执行fallback的线程一定要跟主线程区分开，也就是重新命名一个ThreadPoolKey。

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

Cache via Network.png

  


![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

Cache via Network.png

  


4.2.6、Primary + Secondary with Fallback 主次方式回退（主要和次要）

这个有点类似我们日常开发中需要上线一个新功能，但为了防止新功能上线失败可以回退到老的代码，我们会做一个开关比如使用zookeeper做一个配置开关，可以动态切换到老代码功能。那么Hystrix它是使用通过一个配置来在两个command中进行切换。

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

Primary + Secondary with Fallback.png

  


![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

CommandFacadeWithPrimarySecondary-1.png

  


![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

CommandFacadeWithPrimarySecondary-2.png

  


![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

CommandFacadeWithPrimarySecondary-3.png

  


4.3、回退降级小结

降级的处理方式，返回默认值，返回缓存里面的值（包括远程缓存比如redis和本地缓存比如jvmcache）。  
但回退的处理方式也有不适合的场景：  
1、写操作  
2、批处理  
3、计算  
以上几种情况如果失败，则程序就要将错误返回给调用者。

总结

Hystrix为我们提供了一套线上系统容错的技术实践方法，我们通过在系统中引入Hystrix的jar包可以很方便的使用线程隔离、熔断、回退等技术。同时它还提供了监控页面配置，方便我们管理查看每个接口的调用情况。像spring cloud这种微服务构建模式中也引入了Hystrix，我们可以放心使用Hystrix的线程隔离技术，来防止雪崩这种可怕的致命性线上故障。

