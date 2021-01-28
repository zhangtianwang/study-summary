## soul集成熔断器hystrix ##
# 1 hystrix有什么用 #
1.  微服务现在越来越多，服务之间的调用也越来越频繁，比如服务A和B都调用服务C，如果服务C的某一个节点坏了,服务A，B都会受到影响，调用A和B服务的调用者也会受到影响，依次类推，从而形成
雪崩效应
2. 	为了保证服务的高可用，避免雪崩效应，熔断器对调用有一个记忆功能，如果超过一定次数的调用
失败，新请求进来，直接把异常返回调用端（参考Spring Cloud的hystrix）

----------

# 2 hystrix是怎么集成的 #
1. 启动soul-admin，在soul-admin后台开始hystrix插件，divide插件
2. soul-bootrap， 引入依赖soul-spring-boot-starter-plugin-hystrix,启动成功
3. 启动项目soul-examples-http，启动成功。
3. 我们首先访问地址：http://localhost:9195/http/order/findById?id=2
   返回{"id":"2","name":"hello world findById"}，成功。
   然后，我们在 order/findById 方法增加：
   try {
            Thread.sleep(5000);
   }catch (Exception ex){}

	再次访问：{"id":"2","name":"hello world findById"}
	返回：{"code":-104,"message":"Service call timeout!","data":null}
	hystrix设置的默认超时时间是3秒，代码超过了5s,触发了熔断，hystrix开始有
	动作了


----------
# 3 hystrix源码分析 #
1. 首先，我们调试代码，第一步到了fetchCommand方法，这个方法将exchange对象，

	 > private Command fetchCommand(final HystrixHandle hystrixHandle, final ServerWebExchange exchange, final SoulPluginChain chain) {
>         if (hystrixHandle.getExecutionIsolationStrategy() == HystrixIsolationModeEnum.SEMAPHORE.getCode()) {
>             return new HystrixCommand(HystrixBuilder.build(hystrixHandle),
>                     exchange, chain, hystrixHandle.getCallBackUri());
>         }
>         return new HystrixCommandOnThread(HystrixBuilder.buildForHystrixCommand(hystrixHandle),
>                 exchange, chain, hystrixHandle.getCallBackUri());
>     }

时间有限，明天继续调试代码，把这块的代码补充完。

	
