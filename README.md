
## 前言


本文记录一下 StarBlog 项目的当前状态与接下来 v2 版本的开发规划。


StarBlog 项目从 2022 年开始至今已经 2 年多了，本来早就该给第一期做个小结的，但这种博客类型的项目，一旦稳定能用之后，我就没多大的动力去更新了 😂


博客地址是: [blog.deali.cn](https://github.com)


另外还有个用 NextJS 重构的[预览版](https://github.com):[豆荚加速器官网PodHub](https://doujiaa.com)



> PS：对了，还有个 StarBlog 的 Vue 前端系列，我之前已经写好了，一直没有发出来，接下来会连载更新，感兴趣的同学可以关注一下。
> 
> 
> 内容比较简单（当时我是边学 Vue 边做的），不过从这个角度来说也挺适合刚入门的同学阅读，毕竟以初学者的角度写的。


当时开发这个项目的本意是边学边做，作为熟悉 AspNetCore 的练手项目，现在说实话也无法投入很多时间去开发这类博客项目了……毕竟这种类型的项目太基础了。


还记得我年初规划的几个新项目，到现在的进度都还很有限，所以接下来还是得把时间放在这些新项目上。


这里来回顾一下新项目开发计划：


* StarSSO \- 单点认证服务
* EchoSubs \- 视频字幕识别、翻译服务
* SnapMix \- 随机图片接口服务
* AIHub 2\.0 \- AIHub 的升级版
* StarBlogHub \- 实现一个去中心化的博客聚合平台，不同的个人博客都可以接入，共享流量
* TodayTV \- 看电视，用于代替传统的电视直播
* Clipify \- 基于 Blazor 的视频剪辑工具，已开源


这么看下来也太多了，摊子铺太大了，不好收场啊…


不过精力放在其他项目上也不是意味着不管 StarBlog 项目了，事实上我一直有在做小修小改，这个看 Github commit 就知道了。


## 当前版本的不足之处


从 StarBlog 项目上线至今，我不断学习关于 AspNetCore 的细节知识，相比起刚刚开发这个项目的时候，对框架的熟悉程度提升了一些，自然也发现了之前代码里的局限之处：


* 增删改查的「查」应该使用 patch 方法
* 在 Get 方法接口加上 `[HttpHead]` 来实现对 Head 方法的支持
* 过滤和搜索的接口需要对参数进行 trim
* 不应该将接口的返回值都修改为 `ApiResponse` 类型，应该保留框架的 `ActionResult` 类型，这样功能更多
* 只统一了接口的返回值，没有对异常进行包装，应该使用 `app.UseExceptionHandler` 中间件来实现统一错误处理（也可以使用异常过滤器）
* 对 markdown 的 toc、公式、代码块、表格嵌套图片等还是支持不佳


这些问题将是 v2 版本要解决的。


## v2 新版规划


目前规划了一些新的功能和优化，但这肯定不是 v2 版本的全部，各位同学如果有好的建议也可以留言讨论一下\~


### 博客前台重构


* 使用 Next.js 重构
* 使用 nodejs 技术栈的 markdown 解析


### 管理后台重构


* 使用基于 react 的技术栈重构


### 新的访问统计功能


* 地理信息可视化
* 搜索引擎收录分析
* 反爬虫功能
* 文章阅读量统计


### 文章编辑功能


* 使用新的 markdown 编辑器（最好像 wagtail 那样所见即所得的）
* 支持在文章中加入更多内容（如视频）


### 文章阅读体验优化


* 使用新的 markdown 渲染工具（目前使用的是我 fork 魔改的 editor.md，用起来还可以，但这个工具很老了，而且也停更了，我希望找一个维护良好更现代的渲染工具来替代）


### 文章加密


* 设置固定密码
* 关注公众号获取动态密码


### 新版搜索功能


* 使用全文检索引擎
* 加入 Embedding


### AI 功能


* 知识库
* 对话功能
* 文章 AI 总结
* 自动评论


## AspNetCore 温故知新


24 年初我又复习了一些 AspNetCore 框架的功能，比较零散不成体系，与 StarBlog 的开发是息息相关的，所以在本文记录一下吧\~


### 统一错误处理


#### 异常过滤器


编写过滤器



```
public class ApiExceptionFilter : IExceptionFilter {
  public void OnException(ExceptionContext context) {
    var response = new ApiResponse {
      StatusCode = 500,
      Successful = false,
      // 可以根据需求修改此处来更详细地描述错误信息
      Message = context.Exception.Message
    };

    context.Result = new ObjectResult(response) {
      StatusCode = 500
    };

    // 标记异常已处理
    context.ExceptionHandled = true;
  }
}

```

注册



```
builder.Services.AddControllers(options => {
  options.Filters.Add();
  options.Filters.Add();
});

```

过滤器可以捕获在 Action 方法或控制器中抛出的异常，并允许开发者对其进行处理，然后返回一个统一的响应格式。


但不是在 Action 方法或控制器中抛出的异常，是捕获不到的，例如加了 `[Authorize]` 特性的接口，没有提供认证信息的时候访问报 401 错误，这种是捕获不到的。


#### 中间件


如果想要在整个应用程序中处理异常，使用中间件可能是更好的选择。中间件可以捕获在请求处理管道中发生的所有类型的异常。


使用 `app.UseExceptionHandler` 中间件来实现统一错误处理


一个简单的例子，在 `Program.cs` 里配置内置的 `ExceptionHandler` 中间件



```
if (app.Environment.IsDevelopment()) {
  app.UseDeveloperExceptionPage();
}
else {
  app.UseExceptionHandler(applicationBuilder => {
    applicationBuilder.Run(async context => {
      // 记录日志啥的
      context.Response.StatusCode = StatusCodes.Status500InternalServerError;
      await context.Response.WriteAsJsonAsync(new { message = "Unexpected error!" });
    });
  });
}

```

自己手写中间件



```
public class ErrorHandlingMiddleware {
  private readonly RequestDelegate _next;

  public ErrorHandlingMiddleware(RequestDelegate next) {
    _next = next;
  }

  public async Task Invoke(HttpContext context) {
    try {
      await _next(context);
    }
    catch (Exception ex) {
      await HandleExceptionAsync(context, ex);
    }
  }

  private static Task HandleExceptionAsync(HttpContext context, Exception exception) {
    var response = new ApiResponse {
      StatusCode = 500,
      Successful = false,
      Message = exception.Message
    };

    context.Response.ContentType = "application/json";
    context.Response.StatusCode = (int)HttpStatusCode.InternalServerError;
    return context.Response.WriteAsync(JsonConvert.SerializeObject(response));
  }
}

```

使用中间件



```
app.UseMiddleware();

```

### 自定义认证授权相关的返回值


在 ASP.NET Core 中，当使用 `app.UseAuthentication()` 和 `app.UseAuthorization()` 中间件处理认证和授权逻辑时，如果认证或授权失败，这些中间件会直接修改响应，返回 HTTP 状态码如 401（未认证）或 403（未授权）。


这些响应并不是通过异常机制处理的，因此常规的异常处理中间件或 `UseExceptionHandler` 无法捕获和修改这些特定的错误响应。


要自定义这些错误响应，需要配置认证中间件以使用特定的事件来修改响应。


这通常涉及到在认证方案的配置中添加事件处理逻辑。下面以 JWT 认证为例说明如何自定义 401 和 403 的响应：


#### 配置 JWT 认证以自定义 401 和 403 响应


在 `services.AddAuthentication().AddJwtBearer(options => { ... })` 里面添加事件配置



```
options.Events = new JwtBearerEvents {
  OnChallenge = async context => {
    if (!context.Response.HasStarted) {
      context.HandleResponse(); // 阻止默认的401响应
      var response = new ApiResponse {
        StatusCode = 401,
        Successful = false,
        Message = "Authentication failed. You are not authorized."
      };
      context.Response.ContentType = "application/json";
      await context.Response.WriteAsync(JsonConvert.SerializeObject(response));
    }
  },
  OnForbidden = async context => {
    var response = new ApiResponse {
      StatusCode = 403,
      Successful = false,
      Message = "You do not have permission to access this resource."
    };
    context.Response.ContentType = "application/json";
    await context.Response.WriteAsync(JsonConvert.SerializeObject(response));
  }
};

```

在 JWT 认证流程中，`JwtBearerEvents` 类提供了多个事件来处理不同的认证相关情景：


1. **OnChallenge** \- 这个事件是在认证失败时触发的，通常是因为请求中没有提供有效的 JWT 令牌。例如，如果请求没有包含令牌，或者令牌不符合预期的格式，或者令牌已过期等情况，都会触发此事件。`OnChallenge` 事件是处理返回 401 未认证响应的正确位置。
2. **OnAuthenticationFailed** \- 这个事件在认证过程中出现异常时触发。这通常涉及到令牌解析或验证中出现的错误，比如令牌被篡改。在此事件中，你可以记录异常或修改认证失败时的处理逻辑。
3. **OnForbidden** \- 当用户通过了认证但是不符合特定的授权条件时触发。例如，用户的角色或权限不足以访问某个资源。在此事件中，你可以自定义返回 403 禁止访问的响应。


### 自定义模型绑定


实现 `IModelBinder` 接口可以自定义接口的 model bind 行为，这种叫做 `Custom Model Binder`，建议放在 `Helpers` 目录下


例子：输入 `guid1,guid2,guid3,guid4` 形式的参数，获取公司列表



```
[HttpGet("{ids}")]
public async Task GetCompanyCollection([FromRoute] IEnumerable ids) {
}

```

新增 `Helpers/ArrayModelBinder.cs` 文件



```
public class ArrayModelBinder : IModelBinder {
  public Task BindModelAsync(ModelBindingContext bindingContext) {
    if (!bindingContext.ModelMetadata.IsEnumerableType) {
      bindingContext.Result = ModelBindingResult.Failed();
      return Task.CompletedTask;
    }

    var value = bindingContext.ValueProvider.GetValue(bindingContext.ModelName).ToString();
    if (string.IsNullOrWhiteSpace(value)) {
      bindingContext.Result = ModelBindingResult.Success(null);
      return Task.CompletedTask;
    }

    // 使用反射获取参数类型
    var elementType = bindingContext.ModelType.GetTypeInfo().GenericTypeParameters[0];

    // 根据参数类型，新建一个类型转换器
    var converter = TypeDescriptor.GetConverter(elementType);

    // 按照 , 分割字符串，
    // 指定 `StringSplitOptions.RemoveEmptyEntries` 参数，用以清除分割后的空值，比如 `1,,3,4` -> `["1", "3", "4"]`
    var values = value.Split(new[] { "," }, StringSplitOptions.RemoveEmptyEntries)
      .Select(x => converter.ConvertFromString(x.Trim()))
      .ToArray();

    var typedValues = Array.CreateInstance(elementType, values.Length);
    values.CopyTo(typedValues, 0);
    bindingContext.Model = typedValues;

    bindingContext.Result = ModelBindingResult.Success(bindingContext.Model);

    return Task.CompletedTask;
  }
}

```

最后在接口签名里指定使用的 `ModelBinder`



```
[HttpGet("{ids}")]
public async Task GetCompanyCollection(
  [FromRoute]
  [ModelBinder(BinderType = typeof(ArrayModelBinder))]
  IEnumerable ids) {
}

```

PS：为了更直观表示出这个 route parameter 是支持传入多个值，可以在参数上加个括号。



```
[HttpGet("({ids})")]
public async Task GetCompanyCollection(

```

## 小结


不知道说啥，现在很少写 C\#了，最近.Net9 好像出来了，优化了 GC 算法，等有空来试试\~


