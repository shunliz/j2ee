拦截器Interceptor

Spring MVC框架中的Interceptor，与Servlet API中的Filter十分类似，用于对Web请求进行预处理/后处理。通常情况下这些预处理/后处理逻辑是通用的，可以被应用于所有或多个Web请求，例如：

* 记录Web请求相关日志，可以用于做一些信息监控、统计、
* * 检查Web请求访问权限，例如发现用户没有登录后，重定向到登录页面
* 打开/关闭数据库连接——预处理时打开，后处理关闭，可以避免在所有业务方法中都编写类似代码，也不会忘记关闭数据库连接

# Spring MVC请求处理流程![](/assets/springmvc1.png)

上图是Spring MVC框架处理Web请求的基本流程，请求会经过`DispatcherServlet`的分发后，会按顺序经过一系列的`Interceptor`并执行其中的预处理方法，在请求返回时同样会执行其中的后处理方法。

在`DispatcherServlet`和`Controller`之间哪些竖着的彩色细条，是拦截请求进行额外处理的地方，所以命名为**拦截器**（**Interceptor**）。

### HandlerInterceptor接口 {#13}

Spring MVC中拦截器是实现了`HandlerInterceptor`接口的Bean：

```
public interface HandlerInterceptor {
    boolean preHandle(HttpServletRequest request, 
                      HttpServletResponse response, 
                      Object handler) throws Exception;

    void postHandle(HttpServletRequest request, 
                    HttpServletResponse response, 
                    Object handler, ModelAndView modelAndView) throws Exception;

    void afterCompletion(HttpServletRequest request, 
                         HttpServletResponse response, 
                         Object handler, Exception ex) throws Exception;
}
```

* `preHandle()`
  ：预处理回调方法，若方法返回值为`true`
  ，请求继续（调用下一个拦截器或处理器方法）；若方法返回值为`false`
* * * * * ，请求处理流程中断，不会继续调用其他的拦截器或处理器方法，此时需要通过`response`
          产生响应；
* `postHandle()`
  ：后处理回调方法，实现处理器的后处理（但在渲染视图之前），此时可以通过`ModelAndView`
  对模型数据进行处理或对视图进行处理
* `afterCompletion()`
  ：整个请求处理完毕回调方法，即在视图渲染完毕时调用

`HandlerInterceptor`有三个方法需要实现，但大部分时候可能只需要实现其中的一个方法，`HandlerInterceptorAdapter`是一个实现了`HandlerInterceptor`的抽象类，它的三个实现方法都为空实现（或者返回`true`），继承该抽象类后可以仅仅实现其中的一个方法：

```
public class Interceptor extends HandlerInterceptorAdapter {

    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        // 在controller方法调用前打印信息
        System.out.println("This is interceptor.");
        // 返回true，将强求继续传递（传递到下一个拦截器，没有其它拦截器了，则传递给Controller）
        return true;
    }
}
```

### 配置Interceptor {#14}

定义`HandlerInterceptor`后，需要创建`WebMvcConfigurerAdapter`在MVC配置中将它们应用于特定的URL中。一般一个拦截器都是拦截特定的某一部分请求，这些请求通过URL模型来指定。

下面是一个配置的例子：

```
@Configuration
public class WebConfig extends WebMvcConfigurerAdapter {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LocaleInterceptor());
        registry.addInterceptor(new ThemeInterceptor()).addPathPatterns("/**").excludePathPatterns("/admin/**");
        registry.addInterceptor(new SecurityInterceptor()).addPathPatterns("/secure/*");
    }
}
```

## @ModelAttribute {#15}

### 方法使用@ModelAttribute标注 {#16}

`@ModelAttribute`标注可被应用在方法或方法参数上。

标注在方法上的`@ModelAttribute`说明方法是用于添加一个或多个属性到model上。这样的方法能接受与`@RequestMapping`标注相同的参数类型，只不过不能直接被映射到具体的请求上。

在同一个控制器中，标注了`@ModelAttribute`的方法实际上会在`@RequestMapping`方法之前被调用。

以下是示例：

```
// Add one attribute
// The return value of the method is added to the model under the name "account"
// You can customize the name via @ModelAttribute("myAccount")

@ModelAttribute
public Account addAccount(@RequestParam String number) {
    return accountManager.findAccount(number);
}

// Add multiple attributes

@ModelAttribute
public void populateModel(@RequestParam String number, Model model) {
    model.addAttribute(accountManager.findAccount(number));
    // add more ...
}
```

`@ModelAttribute`方法通常被用来填充一些公共需要的属性或数据，比如一个下拉列表所预设的几种状态，或者宠物的几种类型，或者去取得一个HTML表单渲染所需要的命令对象，比如`Account`等。

`@ModelAttribute`标注方法有两种风格：

* 在第一种写法中，方法通过返回值的方式默认地将添加一个属性；
* 在第二种写法中，方法接收一个
  `Model`
  对象，然后可以向其中添加任意数量的属性。

可以在根据需要，在两种风格中选择合适的一种。

一个控制器可以拥有多个`@ModelAttribute`方法。同个控制器内的所有这些方法，都会在`@RequestMapping`方法之前被调用。

`@ModelAttribute`方法也可以定义在`@ControllerAdvice`标注的类中，并且这些`@ModelAttribute`可以同时对许多控制器生效。

> 属性名没有被显式指定的时候又当如何呢？在这种情况下，框架将根据属性的类型给予一个默认名称。举个例子，若方法返回一个`Account`类型的对象，则默认的属性名为"account"。可以通过设置`@ModelAttribute`标注的值来改变默认值。当向`Model`中直接添加属性时，请使用合适的重载方法`addAttribute(..)`-即带或不带属性名的方法。

`@ModelAttribute`标注也可以被用在`@RequestMapping`方法上。这种情况下，`@RequestMapping`方法的返回值将会被解释为model的一个属性，而非一个视图名，此时视图名将以视图命名约定来方式来确定。

### 方法参数使用@ModelAttribute标注 {#17}

`@ModelAttribute`标注既可以被用在方法上，也可以被用在方法参数上。

标注在方法参数上的`@ModelAttribute`说明了该方法参数的值将由model中取得。如果model中找不到，那么该参数会先被实例化，然后被添加到model中。在model中存在以后，请求中所有名称匹配的参数都会填充到该参数中。

这在[Spring MVC](https://www.tianmaying.com/tutorial/spring-mvc-quickstart)中被称为数据绑定，一个非常有用的特性，我们不用每次都手动从表格数据中转换这些字段数据。

```
@RequestMapping(path = "/owners/{ownerId}/pets/{petId}/edit", method = RequestMethod.POST)
public String processSubmit(@ModelAttribute Pet pet) { }
```

以上面的代码为例，这个Pet类型的实例可能来自哪里呢？有几种可能:

* 它可能因为
  `@SessionAttributes`
  标注的使用已经存在于model中
* 它可能因为在同个控制器中使用了
  `@ModelAttribute`
  方法已经存在于model中——正如上一小节所叙述的
* 它可能是由URI模板变量和类型转换中取得的（下面会详细讲解）
* 它可能是调用了自身的默认构造器被实例化出来的

`@ModelAttribute`方法常用于从数据库中取一个属性值，该值可能通过`@SessionAttributes`标注在请求中间传递。在一些情况下，使用URI模板变量和类型转换的方式来取得一个属性是更方便的方式。这里有个例子：

```
@RequestMapping(path = "/accounts/{account}", method = RequestMethod.PUT)
public String save(@ModelAttribute("account") Account account) {

}
```

这个例子中，model属性的名称（"account"）与URI模板变量的名称相匹配。如果配置了一个可以将`String`类型的账户值转换成`Account`类型实例的转换器`Converter<String, Account>`，那么上面这段代码就可以工作的很好，而不需要再额外写一个`@ModelAttribute`方法。

下一步就是数据的绑定。`WebDataBinder`类能将请求参数——包括字符串的查询参数和表单字段等——通过名称匹配到model的属性上。成功匹配的字段在需要的时候会进行一次类型转换（从String类型到目标字段的类型），然后被填充到model对应的属性中。

进行了数据绑定后，则可能会出现一些错误，比如没有提供必须的字段、类型转换过程的错误等。若想检查这些错误，可以在标注了`@ModelAttribute`的参数紧跟着声明一个`BindingResult`参数：

```
@RequestMapping(path = "/owners/{ownerId}/pets/{petId}/edit", method = RequestMethod.POST)
public String processSubmit(@ModelAttribute("pet") Pet pet, BindingResult result) {
    if (result.hasErrors()) {
        return "petForm";
    }

    // ...

}
```

拿到`BindingResult`参数后，可以检查是否有错误，可以通过Spring的`<errors>`表单标签来在同一个表单上显示错误信息。

`BindingResult`被用于记录数据绑定过程的错误，因此除了数据绑定外，还可以把该对象传给自己定制的验证器来调用验证。这使得数据绑定过程和验证过程出现的错误可以被搜集到一起，然后一并返回给用户：

```
@RequestMapping(path = "/owners/{ownerId}/pets/{petId}/edit", method = RequestMethod.POST)
public String processSubmit(@ModelAttribute("pet") Pet pet, BindingResult result) {

    new PetValidator().validate(pet, result);
    if (result.hasErrors()) {
        return "petForm";
    }

    // ...

}
```

又或者可以通过添加一个JSR-303规范的`@Valid`标注，这样验证器会自动被调用。

```
@RequestMapping(path = "/owners/{ownerId}/pets/{petId}/edit", method = RequestMethod.POST)
public String processSubmit(@Valid @ModelAttribute("pet") Pet pet, BindingResult result) {

    if (result.hasErrors()) {
        return "petForm";
    }

    // ...

}
```

## 异常处理 {#18}

Spring MVC框架提供了多种机制用来处理异常，初次接触可能会对他们用法以及适用的场景感到困惑。现在以一个简单例子来解释这些异常处理的机制。

假设现在我们开发了一个博客应用，其中最重要的资源就是文章（Post），应用中的URL设计如下：

* 获取文章列表：
  `GET /posts/`
* 添加一篇文章：
  `POST /posts/`
* 获取一篇文章：
  `GET /posts/{id}`
* 更新一篇文章：
  `PUT /posts/{id}`
* 删除一篇文章：
  `DELETE /posts/{id}`

这是非常标准的复合RESTful风格的URL设计，在Spring MVC实现的应用过程中，相应也会有5个对应的用`@RequestMapping`注解的方法来处理相应的URL请求。在处理某一篇文章的请求中（获取、更新、删除），无疑需要做这样一个判断——请求URL中的文章id是否在于系统中，如果不存在需要返回`404 Not Found`。

### 使用HTTP状态码 {#19}

在默认情况下，Spring MVC处理Web请求时如果发现存在没有应用代码捕获的异常，那么会返回HTTP 500（Internal Server Error）错误。但是如果该异常是我们自己定义的并且使用`@ResponseStatus`注解进行修饰，那么Spring MVC则会返回指定的HTTP状态码：

```
@ResponseStatus(value = HttpStatus.NOT_FOUND, reason = "No Such Post")//404 Not Found
public class PostNotFoundException extends RuntimeException {
}
```

在`Controller`中可以这样使用它：

```
@RequestMapping(value = "/posts/{id}", method = RequestMethod.GET)
public String showPost(@PathVariable("id") long id, Model model) {
    Post post = postService.get(id);
    if (post == null) throw new PostNotFoundException("post not found");
    model.addAttribute("post", post);
    return "postDetail";
}
```

这样如果我们访问了一个不存在的文章，那么Spring MVC会根据抛出的`PostNotFoundException`上的注解值返回一个HTTP 404 Not Found给浏览器。

### 最佳实践 {#20}

上述场景中，除了获取一篇文章的请求，还有更新和删除一篇文章的方法中都需要判断文章id是否存在。在每一个方法中都加上`if (post == null) throw new PostNotFoundException("post not found");`是一种解决方案，但如果有10个、20个包含`/posts/{id}`的方法，虽然只有一行代码但让他们重复10次、20次也是非常不优雅的。

为了解决这个问题，可以将这个逻辑放在Service中实现：

    @Service
    public class PostService {

        @Autowired
        private PostRepository postRepository;

        public Post get(long id) {
            return postRepository.findById(id)
                    .orElseThrow(() -
    >
     new PostNotFoundException("post not found"));
        }
    }

    这里`PostRepository`继承了`JpaRepository`，可以定义`findById`方法返回一个`Optional
    <
    Post
    >
    `——如果不存在则Optional为空，抛出异常。

这样在所有的`Controller`方法中，只需要正常获取文章即可，所有的异常处理都交给了Spring MVC。

### 在`Controller`中处理异常 {#21}

`Controller`中的方法除了可以用于处理Web请求，还能够用于处理异常处理——为它们加上`@ExceptionHandler`即可：

```
@Controller
public class ExceptionHandlingController {

  // @RequestHandler methods
  ...

  // Exception handling methods

  // Convert a predefined exception to an HTTP Status code
  @ResponseStatus(value=HttpStatus.CONFLICT, reason="Data integrity violation")  // 409
  @ExceptionHandler(DataIntegrityViolationException.class)
  public void conflict() {
    // Nothing to do
  }

  // Specify the name of a specific view that will be used to display the error:
  @ExceptionHandler({SQLException.class,DataAccessException.class})
  public String databaseError() {
    // Nothing to do.  Returns the logical view name of an error page, passed to
    // the view-resolver(s) in usual way.
    // Note that the exception is _not_ available to this view (it is not added to
    // the model) but see "Extending ExceptionHandlerExceptionResolver" below.
    return "databaseError";
  }

  // Total control - setup a model and return the view name yourself. Or consider
  // subclassing ExceptionHandlerExceptionResolver (see below).
  @ExceptionHandler(Exception.class)
  public ModelAndView handleError(HttpServletRequest req, Exception exception) {
    logger.error("Request: " + req.getRequestURL() + " raised " + exception);

    ModelAndView mav = new ModelAndView();
    mav.addObject("exception", exception);
    mav.addObject("url", req.getRequestURL());
    mav.setViewName("error");
    return mav;
  }
}
```

首先需要明确的一点是，在`Controller`方法中的`@ExceptionHandler`方法只能够处理同一个`Controller`中抛出的异常。这些方法上同时也可以继续使用`@ResponseStatus`注解用于返回指定的HTTP状态码，但同时还能够支持更加丰富的异常处理：

* 渲染特定的视图页面
* 使`ModelAndView`返回更多的业务信息

大多数网站都会使用一个特定的页面来响应这些异常，而不是直接返回一个HTTP状态码或者显示Java异常调用栈。当然异常信息对于开发人员是非常有用的，如果想要在视图中直接看到它们可以这样渲染模板（以JSP为例）：

```
<h1>Error Page</h1>
<p>Application has encountered an error. Please contact support on ...</p>

<!--
Failed URL: ${url}
Exception:  ${exception.message}
<c:forEach items="${exception.stackTrace}" var="ste">    ${ste} 
</c:forEach>
-->
```

### 全局异常处理 {#22}

[@ControllerAdvice](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#mvc-ann-controller-advice)提供了和上一节一样的异常处理能力，但是可以被应用于Spring应用上下文中的所有`@Controller`：

```
@ControllerAdvice
class GlobalControllerExceptionHandler {
    @ResponseStatus(HttpStatus.CONFLICT)  // 409
    @ExceptionHandler(DataIntegrityViolationException.class)
    public void handleConflict() {
        // Nothing to do
    }
}
```

Spring MVC默认对于没有捕获也没有被`@ResponseStatus`以及`@ExceptionHandler`声明的异常，会直接返回500，这显然并不友好，可以在`@ControllerAdvice`中对其进行处理（例如返回一个友好的错误页面，引导用户返回正确的位置或者提交错误信息）：

```
@ControllerAdvice
class GlobalDefaultExceptionHandler {
    public static final String DEFAULT_ERROR_VIEW = "error";

    @ExceptionHandler(value = Exception.class)
    public ModelAndView defaultErrorHandler(HttpServletRequest req, Exception e) throws Exception {
        // If the exception is annotated with @ResponseStatus rethrow it and let
        // the framework handle it - like the OrderNotFoundException example
        // at the start of this post.
        // AnnotationUtils is a Spring Framework utility class.
        if (AnnotationUtils.findAnnotation(e.getClass(), ResponseStatus.class) != null)
            throw e;

        // Otherwise setup and send the user to a default error-view.
        ModelAndView mav = new ModelAndView();
        mav.addObject("exception", e);
        mav.addObject("url", req.getRequestURL());
        mav.setViewName(DEFAULT_ERROR_VIEW);
        return mav;
    }
}
```

### 总结 {#23}

Spring在异常处理方面提供了一如既往的强大特性和支持，那么在应用开发中我们应该如何使用这些方法呢？以下提供一些经验性的准则：

* 不要在
  `@Controller`
  中自己进行异常处理逻辑。即使它只是一个Controller相关的特定异常，在
  `@Controller`
  中添加一个
  `@ExceptionHandler`
  方法处理。
* 对于自定义的异常，可以考虑对其加上
  `@ResponseStatus`
  注解
* 使用
  `@ControllerAdvice`
  处理通用异常（例如资源不存在、资源存在冲突等）



