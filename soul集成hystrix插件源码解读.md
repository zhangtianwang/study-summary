# 一：熔断器能做什么 #
# 1服务A调用B为例，直接调用和熔断器调用区别 #
1. 直接调用是在一个线程内执行，熔断器是会新开一个线程，后续的请求是在一个新线程内执行。  
2. 服务A调用服务B如果发生异常，直接调用服务A所在的线程会抛出异常，用熔断器，则A线程不会受到
影响，A调用还可以做其他后续的处理。
3  直接调用处理异常，需要自己去做分类处理，而熔断器做了一些降级策略，回滚策略，我们在往熔断器发的Commond的实现类，实现方法就可
4 熔断器还可以控制并发量和超时时间，如果失败到了一定次数，可以选择不调用服务B，而直接调用
则控制不了

# 2调用关系 #
直接调用： 服务A——》服务B  
熔断器调用：请求： 服务A——》发出Commond——》熔断器收到命令——》 执行调用服务B  
				响应： 服务B返回结果——》熔断器——》服务A。

# 二：soul集成hystrix插件源码分析 #
1 首先，还是我们熟悉的SoulWebHandler的execute方法，获取到插件的列表，遍历执行插件列表中
启用的插件	

	public Mono<Void> execute(final ServerWebExchange exchange) {
    		return Mono.defer(() -> {
    		if (this.index < plugins.size()) {
    			SoulPlugin plugin = plugins.get(this.index++);
    		Boolean skip = plugin.skip(exchange);
    		if (skip) {
    			return this.execute(exchange);
    		}
    		return plugin.execute(exchange, this);
    	}
    		return Mono.empty();
    });

----------
2 我们在soul-admin已经开启了hystrix plugin的配置，所以会执行HystrixPlugin的doExecute方法,首先doExecute方法先获取到上下文对象，然后获取到Command对象,向熔断器发出命令用的就是
这个command对象，
	    
	protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        ...
        Command command = fetchCommand(hystrixHandle, exchange, chain);
        return Mono.create(s -> {
            Subscription sub = command.fetchObservable().subscribe(s::success,
                    s::error, s::success);
            s.onCancel(sub::unsubscribe);
        }).doOnError(throwable -> {
            log.error("hystrix execute exception:", throwable);
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.ERROR.getName());
            chain.execute(exchange);
        }).then();
    }

----------

	
	//可以看到根据配置的熔断器的隔离策略，采用的command对象是不一致的，信号量用的HystrixCommand对象，而线城池用的是HystrixCommandOnThread对象,命令如果执行的有
	问题，会执行下面的resumeWithFallback（服务降级）方法。
	
	private Command fetchCommand(final HystrixHandle hystrixHandle, final ServerWebExchange exchange, final SoulPluginChain chain) {
        if (hystrixHandle.getExecutionIsolationStrategy() == HystrixIsolationModeEnum.SEMAPHORE.getCode()) {
            return new HystrixCommand(HystrixBuilder.build(hystrixHandle),
                    exchange, chain, hystrixHandle.getCallBackUri());
        }
        return new HystrixCommandOnThread(HystrixBuilder.buildForHystrixCommand(hystrixHandle),
                exchange, chain, hystrixHandle.getCallBackUri());
    }
	//将执行操作后续交给熔断器去执行。
    protected Observable<Void> construct() {
        return RxReactiveStreams.toObservable(chain.execute(exchange));
    }

	//服务降级的回调
    @Override
    protected Observable<Void> resumeWithFallback() {
    private Mono<Void> doFallback() {
       
        final Throwable exception = getExecutionException();
        return doFallback(exchange, exception);
        return RxReactiveStreams.toObservable(doFallback());
    }}



	
				
				



