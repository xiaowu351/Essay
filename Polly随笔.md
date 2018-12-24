# 开源服务容错处理库Polly笔记
在进入SOA之后，我们的代码从本地方法调用变成了跨机器的通信。任何一个新技术的引入都会为我们解决特定的问题，都会带来一些新的问题。比如网络故障、依赖服务崩溃、超时、服务器内存与CPU等其它问题。正是因为这些问题无法避免，所以我们在进行系统设计、特别是进行分布式系统设计的时候以“Design For Failure”（为失败而设计）为指导原则。把一些边缘场景以及服务之间的调用发生的异常和超时当成一定会发生的情况来预先进行处理。
> Design For Failure  
>  _1.一个依赖服务的故障不会严重破坏用户的体验；_  
> _2.系统能自动或半自动处理故障，具备自我恢复能力；_

以下是一些经验的服务容错模式：详见美团总结的[服务容错模式](https://zhuanlan.zhihu.com/p/23711137)
* 超时与重试（Timeout and Retry）
* 限流（Rate Limiting）
* 熔断器（Circuit Breaker）
* 舱壁隔离（Bulkhead Isolation）
* 回退（Fallback）


Polly是开源社区针对以上服务容错模式的一个C#实现的弹性瞬时错误处理库。在Polly中，对这些服务容错模式分为两类：
* 错误处理 fault handing:重试、熔断、回退；
* 弹性应变resilience：超时、舱壁、缓存；

可以说错误处理当错误已经发生时，防止由于该错误对整个系统造成更坏的影响而设置的；而弹性应变，则是在错误发生前，针对有可以发生错误的地方进行预先处理，从而达到保护整个系统的目的；

## Polly错误处理使用三部曲：
* 定义条件：定义你要处理的 错误异常/返回结果；
* 定义处理方式：重试、熔断、回退；
* 执行；

先看一个简单的例子

```
//这个例子展示了当DoSomething方法执行的时候如果遇到SomeExceptionType的异常则会进行重试调用。
var policy = Policy
.Handle<SomeExceptionType>()//1.定义条件
.Retry();//2. 定义处理方式

//3.执行
policy.Execute(()=>DoSomething());
```
### 1.定义条件
我们可以针对两种情况来定义条件：错误异常和返回结果。
#### _1.1 错误异常_
```
//单个异常类型
Policy.Handle<HttpRequestException>();

//限定条件的单个异常
Policy.Handle<SqlException>(ex=>ex.Number == 1205)

//多个异常类型
Policy
.Handle<HttpRequestException>()
.Or<OperationCanceledException>()

// 限定条件的多个异常
Policy
.Handle<SqlException>(ex=>ex.Number == 1205)
.Or<ArgumentException>(ex=>ex.ParamName == "example")

//Inner Exception 异常里面的异常类型
Policy
.HandleInner<HttpRequestException>()
.OrInner<OperationCanceledException>(ex=>ex.CancelationToken != myToken)
```

#### _1.2 返回结果_

```
// 返回结果限制条件
Policy
.HandleResult<HttpResponseMessage>(r=>r.StatusCode == HttpStatusCode.NotFound)

// 处理多个返回结果限制条件
Policy
.HandleResult<HttpResponseMessage>(r=>r.StatusCode == HttpStatusCode.InternalServerError)
.OrResult<HttpResponseMessage>(r=>r.StatusCode == HttpStatusCode.BadGateway)

//处理元类型结果（用 .Equals）进行比较
Policy.
HandleResult<HttpStatusCode>(HttpStatusCode.InternalServerError)
.OrResult<HttpStatusCode>(HttpStatusCode.BadGateway)

// 在一个policy里面同时处理异常和返回结果。
HttpStatusCode[] httpStatusCodesWorthRetrying = {
   HttpStatusCode.RequestTimeout, // 408
   HttpStatusCode.InternalServerError, // 500
   HttpStatusCode.BadGateway, // 502
   HttpStatusCode.ServiceUnavailable, // 503
   HttpStatusCode.GatewayTimeout // 504
}; 
HttpResponseMessage result = Policy
  .Handle<HttpRequestException>()
  .OrResult<HttpResponseMessage>(r => httpStatusCodesWorthRetrying.Contains(r.StatusCode))
  .RetryAsync(...)
  .ExecuteAsync( /* some Func<Task<HttpResponseMessage>> */ )

```
### 2.定义处理方式
我们将介绍以下三种：重试、熔断、回退。
#### _2.1 重试_
重试很好理解，当发生某种错误或者返回某种结果的时候进行重试。Polly里面提供了以下几种重试机制:
* 按次数重试
* 不断重试（直到成功）
* 等待之后按次数重试
* 等待之后不断重试（直到成功）

##### _2.1.1按次数重试_
```
// 重试1次
Policy
  .Handle<SomeExceptionType>()
  .Retry()

// 重试3（N）次
Policy
  .Handle<SomeExceptionType>()
  .Retry(3)

// 重试多次，加上重试时的action参数
Policy
    .Handle<SomeExceptionType>()
    .Retry(3, (exception, retryCount) =>
    {
        // 干点什么，比如记个日志之类的 
    });
```
##### _2.1.2不断重试_
```
// 不断重试,直到成功
Policy
  .Handle<SomeExceptionType>()
  .RetryForever()

// 不断重试，带action参数在每次重试的时候执行
Policy
  .Handle<SomeExceptionType>()
  .RetryForever(exception =>
  {
        // do something       
  });
```
##### _2.1.3等待之后再重试_
```
// 重试3次，分别等待1、2、3秒。
Policy
  .Handle<SomeExceptionType>()
  .WaitAndRetry(new[]
  {
    TimeSpan.FromSeconds(1),
    TimeSpan.FromSeconds(2),
    TimeSpan.FromSeconds(3)
  });
```
当然也可以在每次重试的时候添加一些处理，这里我们可以从上下文中获取一些数据，这些数据在policy启动执行的时候可以传进来。
```
Policy
  .Handle<SomeExceptionType>()
  .WaitAndRetry(new[]
  {
    TimeSpan.FromSeconds(1),
    TimeSpan.FromSeconds(2),
    TimeSpan.FromSeconds(3)
  }, (exception, timeSpan, context) => {
    // do something    
  });
```
##### _2.1.4等待之后不断重试_
把WiatAndRetry抱成WaitAndRetryForever()则可以实现重试直到成功。

#### _2.2 熔断_
熔断也可以被作为当遇到某种错误场景下的一个操作。以下代码展示了当发生2次SomeExceptionType的异常的时候则会熔断1分钟，该操作后续如果继续尝试执行则会直接返回错误。
```
Policy
    .Handle<SomeExceptionType>()
    .CircuitBreaker(2, TimeSpan.FromMinutes(1));
```
可以在熔断和恢复的时候定义委托来做一些额外的处理。onBreak会在被熔断时执行，而onReset则会在恢复时执行。
```
Action<Exception, TimeSpan> onBreak = (exception, timespan) => { ... };
Action onReset = () => { ... };
CircuitBreakerPolicy breaker = Policy
    .Handle<SomeExceptionType>()
    .CircuitBreaker(2, TimeSpan.FromMinutes(1), onBreak, onReset);
```
###### 熔断器状态
我们的CircuitBreakPolicy的State定义了当前熔断器的状态，我们也可能调用它的Isolate和Reset方法来手动熔断和恢复。
```
CircuitState state = breaker.CircuitState;
```
* Closed 关闭状态，允许执行；
* Open 自动打开，执行会被阻断；
* Isolate 手动打开，执行会被阻断；
* HalfOpen 从自动打开状态恢复中，在熔断时间到了之后会从Open状态切换到Closed
```
// 手动打开熔断器，阻止执行
breaker.Isolate(); 
// 恢复操作，启动执行 
breaker.Reset(); 
```
#### _2.3 回退（Fallback）_
```
// 如果执行失败则返回UserAvatar.Blank
Policy
   .Handle<Whatever>()
   .Fallback<UserAvatar>(UserAvatar.Blank)

// 发起另外一个请求去获取值
Policy
   .Handle<Whatever>()
   .Fallback<UserAvatar>(() => UserAvatar.GetRandomAvatar()) // where: public UserAvatar GetRandomAvatar() { ... }

// 返回一个指定的值，添加额外的处理操作。onFallback
Policy
   .Handle<Whatever>()
   .Fallback<UserAvatar>(UserAvatar.Blank, onFallback: (exception, context) => 
    {
        // do something
    });
```
### 3.执行Policy
声明了一个Policy，并定义了它的异常条件和处理方式，那么接下来就是执行它。执行是把我们具体要运行的代码放到Policy里面。
```
// 执行一个Action
var policy = Policy
              .Handle<SomeExceptionType>()
              .Retry();

policy.Execute(() => DoSomething());
```
这就是我们最开始的例子，还记得我们在异常处理的时候有一个context上下文吗？我们可以在执行的时候带一些参数进去。
```
// 看我们在retry重试时被调用的一个委托，它可以从context中拿到我们在execute的时候传进来的参数 。
var policy = Policy
    .Handle<SomeExceptionType>()
    .Retry(3, (exception, retryCount, context) =>
    {
        var methodThatRaisedException = context["methodName"];
		Log(exception, methodThatRaisedException);
    });

policy.Execute(
	() => DoSomething(),
	new Dictionary<string, object>() {{ "methodName", "some method" }}
);
```
当然，我们也可以将Handle，Retry, Execute 这三个阶段都串起来写。
```
Policy
  .Handle<SqlException>(ex => ex.Number == 1205)
  .Or<ArgumentException>(ex => ex.ParamName == "example")
  .Retry()
  .Execute(() => DoSomething());
```
## Polly弹性应变处理Resilience
我们在上面讲了Polly在错误处理方面的使用，接下来我们介绍Polly在弹性应变这块的三个应用： 超时、舱壁和缓存。

**超时**
```
Policy
  .Timeout(TimeSpan.FromMilliseconds(2500))
```
支持传入action回调
```
Policy
  .Timeout(30, onTimeout: (context, timespan, task) => 
    {
        // do something 
    });
```
超时分为乐观超时与悲观超时，乐观超时依赖于CancellationToken ，它假设我们的具体执行的任务都支持CancellationToken。那么在进行timeout的时候，它会通知执行线程取消并终止执行线程，避免额外的开销。下面的乐观超时的具体用法 
```
// 声明 Policy 
Policy timeoutPolicy = Policy.TimeoutAsync(30);
HttpResponseMessage httpResponse = await timeoutPolicy
    .ExecuteAsync(
      async ct => await httpClient.GetAsync(endpoint, ct), 
      CancellationToken.None 
// 最后可以把外部的 CacellationToken附加到 timeoutPollcy的 CT上，在这里我们没有附加
      );
```
悲观超时与乐观超时的区别在于，如果执行的代码不支持取消CancellationToken，它还会继续执行，这会是一个比较大的开销。
```
Policy
  .Timeout(30, TimeoutStrategy.Pessimistic)
```
上面的代码也有悲观sad…的写法
```
Policy timeoutPolicy = Policy.TimeoutAsync(30, TimeoutStrategy.Pessimistic);
var response = await timeoutPolicy
    .ExecuteAsync(
      async () => await FooNotHonoringCancellationAsync(), 
      );// 在这里我们没有 任何与CancllationToken相关的处理
```
**舱壁** 

舱壁模式，它是用来限制某一个操作的最大并发执行数量 。比如限制为12
```
Policy
  .Bulkhead(12)
```
同时，我们还可以控制一个等待处理的队列长度
```
Policy
  .Bulkhead(12, 2)
```
以及当请求执行操作被拒绝的时候，执行回调
```
Policy
  .Bulkhead(12, context => 
    {
        // do something 
    });
```
**缓存**

Polly的缓存需要依赖于一个外部的Provider。
```
var memoryCacheProvider = new Polly.Caching.MemoryCache.MemoryCacheProvider(MemoryCache.Default);
var cachePolicy = Policy.Cache(memoryCacheProvider, TimeSpan.FromMinutes(5));

// 设置一个绝对的过期时间
var cachePolicy = Policy.Cache(memoryCacheProvider, new AbsoluteTtl(DateTimeOffset.Now.Date.AddDays(1));

// 设置一个滑动的过期时间，即每次使用缓存的时候，过期时间会更新
var cachePolicy = Policy.Cache(memoryCacheProvider, new SlidingTtl(TimeSpan.FromMinutes(5));

// 我们用Policy的缓存机制来实现从缓存中读取一个值，如果该值在缓存中不存在则从提供的函数中取出这个值放到缓存中。
// 借且于Polly Cache  这个操作只需要一行代码即可。
TResult result = cachePolicy.Execute(() => getFoo(), new Context("FooKey")); // "FooKey" is the cache key used in this execution.

// Define a cache Policy, and catch any cache provider errors for logging. 
var cachePolicy = Policy.Cache(myCacheProvider, TimeSpan.FromMinutes(5), 
   (context, key, ex) => { 
       logger.Error($"Cache provider, for key {key}, threw exception: {ex}."); // (for example) 
   }
);
```
**组合Policy**

最后我们要说的是如何将多个policy组合起来。大致的操作是定义多个policy，然后用Wrap方法即可。
```
var policyWrap = Policy
  .Wrap(fallback, cache, retry, breaker, timeout, bulkhead);
policyWrap.Execute(...)
```
在另一个Policy声明时组合使用其它外部声明的Policy。
```
PolicyWrap commonResilience = Policy.Wrap(retry, breaker, timeout);

Avatar avatar = Policy
   .Handle<Whatever>()
   .Fallback<Avatar>(Avatar.Blank)
   .Wrap(commonResilience)
   .Execute(() => { /* get avatar */ });
```





