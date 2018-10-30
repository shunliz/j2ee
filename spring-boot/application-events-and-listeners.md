# Application Events and Listeners

## 1前言

在springboot启动的过程中会产生一系列事件，我们开发的时候可以自定义一些事件监听处理器．根据自己的需要在针对每个事件做一些业务处理．

## 2Application Events

springboot 启动的时候会按顺序产生如下几种事件：

* 1`ApplicationStartingEvent`
  :springboot应用启动且未作任何处理（除listener注册和初始化）的时候发送
  `ApplicationStartingEvent`
* 2`ApplicationEnvironmentPreparedEvent`
  :确定springboot应用使用的Environment且context创建之前发送这个事件
* 3`ApplicationPreparedEvent`
  :context已经创建且没有refresh发送个事件
* 4`ApplicationStartedEvent`
  :context已经refresh且application and command-line runners（如果有） 调用之前发送这个事件
* 5`ApplicationReadyEvent`
  ://application and command-line runners （如果有）执行完后发送这个事件，此时应用已经启动完毕.这个事件比较常用，常常在系统启动完后做一些初始化操作
* 6`ApplicationFailedEvent`
  :应用启动失败后产生这个事件

## 3实现

##### 3.1模拟启动，进行初始化操作

首先模拟系统启动后需要进行业务操作，这里只是范例，所以只打印一句话:

```
package com.simos.service;

import org.springframework.stereotype.Service;


@Service
public class SystemInitService {
public void systemInit(){
    System.out.println("应用初始化后，进行一些业务操作，如启动某些工作线程，初始化系统某些参数");
}
}
```

##### 3.２事件监听处理实现

通过实现ApplicationListener接口，完成事件的监听处理：

```
package com.simos.listener;

import com.simos.service.SystemInitService;
import org.springframework.boot.context.event.*;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationEvent;
import org.springframework.context.ApplicationListener;


public class SimosApplicationListener implements ApplicationListener<ApplicationEvent>{
@Override
public void onApplicationEvent(ApplicationEvent event) {
    //springboot应用启动且未作任何处理（除listener注册和初始化）的时候发送ApplicationStartingEvent
    if (event instanceof ApplicationStartingEvent){
        System.out.println("ApplicationStarting");
        return;
    }
    //确定springboot应用使用的Environment且context创建之前发送这个事件
    if (event instanceof ApplicationEnvironmentPreparedEvent){
        System.out.println("ApplicationEnvironmentPrepared");
        return;
    }
    //context已经创建且没有refresh发送个事件
    if (event instanceof ApplicationPreparedEvent){
        System.out.println("ApplicationPrepared");
        return;
    }
    //context已经refresh且application and command-line runners（如果有） 调用之前发送这个事件
    if (event instanceof ApplicationStartedEvent){
        System.out.println("ApplicationStarted");
        return;
    }
    //application and command-line runners （如果有）执行完后发送这个事件，此时应用已经启动完毕
    if (event instanceof ApplicationReadyEvent){
        ApplicationContext context = ((ApplicationReadyEvent) event).getApplicationContext();
        SystemInitService initService = context.getBean(SystemInitService.class);
        initService.systemInit();
        return;
    }
    //应用启动失败后产生这个事件
    if (event instanceof ApplicationFailedEvent){
        System.out.println("ApplicationFailed");
        return;
    }
}
}
```

##### 3.3监听器listener注册

第一种方式是手动注册，即在SpringApplication初始化的时候添加进去

```

@SpringBootApplication
public class QuickStartApplication {
public static void  main(String[]args){
    new SpringApplicationBuilder().sources(QuickStartApplication.class)
            .listeners(new SimosApplicationListener()).run(args);
}
}
```

第二种方式是，自动注册（springboot本身listener实现也是通过这种方式）．在resources目录下添加META-INF目录，然后在META-INF目录里添加spring.factories文件，文件内容是：

```
org.springframework.context.ApplicationListener=com.simos.listener.SimosApplicationListener
```

通过自动注册的方式main入口就与无listener时一样：

```

@SpringBootApplication
public class QuickStartApplication {
public static void  main(String[]args){
  SpringApplication.run(QuickStartApplication.class,args);
}
}
```

启动后控制台打印如下：

![](https://upload-images.jianshu.io/upload_images/6566116-63cf043d78f6bce0.png?imageMogr2/auto-orient/strip|imageView2/2/w/868/format/webp)

启动打印结果

