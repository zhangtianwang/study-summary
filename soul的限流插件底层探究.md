# 一：soul的限流简单介绍 #

	    服务处理同一时间请求的数量是有上限的，随着同时请求数量的增大， 服务器的CPU和内存等
	系统资源，会消耗的很厉害，最终响应会越来越慢，甚至崩溃。
		我们在客户端和服务器之间加了一层网关层，网关层就可以对一段时间内的请求数进行控制，让并发数小于单台服务器的QPS,超过最大QPS的请求，则不处理。
			
	

----------
	

# 二：soul集成ratelimiter #

----------

1. soul-admin plugin模块开启rate_limiter这个插件, 同时PluginList为rate_limiter增加selector和rule。	      
2. 启动soul-admin，启动成功
3. 启动soul-bootstrap，启动成功,加载了RateLimiterPlugin
	控制台输出: load plugin:[rate_limiter] [org.dromara.soul.plugin.ratelimiter.RateLimiterPlugin]
4. 启动soul-examples-http 启动成功。
 
----------

# 三：ratelimiter插件流程源码分析 #

----------
 
> 大体的流程：1 前面第二步已经介绍了开始rate_limiter配置，并启动项目          
> 		     2 客户调用——》请求到soul-bootstrap，soul-bootstrap开始执行plugins，
> 		     执行到RateLimiterPlugin，如果流量超过最大限制，直接返回异常，如果未超
> 		     ，正常往下执行。
> 		      
   先找到我们熟悉的SoulWebHandler，加载pluginList，并调用作用链循环执行Plugin。   
  在执行RateLimiterPlugin之前，先调用了SoulAbstractPlugin的execute方法,主要是根据context去获取到Selector和Rule。
	
    3.1 public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        String pluginName = named();
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
        if (pluginData != null && pluginData.getEnabled()) {
            final Collection<SelectorData> selectors = BaseDataCache.getInstance=
            final SelectorData selectorData = matchSelector(exchange, selectors);
            ...
            RuleData rule;
            if (selectorData.getType() == SelectorTypeEnum.FULL_FLOW.getCode()) {
                //get last
                rule = rules.get(rules.size() - 1);
            } else {
                rule = matchRule(exchange, rules);
            }
            ...
            return doExecute(exchange, chain, selectorData, rule);
        }
        return chain.execute(exchange);
    }

----------


	接着继续调试，找到了RateLimiterPlugin,最核心的方法就是isAllowed方法，决定了
	流量是否超限。
	3.2 
	protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        final String handle = rule.getHandle();
        final RateLimiterHandle limiterHandle = GsonUtils.getInstance().fromJson(handle, RateLimiterHandle.class);
        //isAllowed是核心方法，根据配置好的流量参数和当前的流量，计算下是否继续往下执行。
        return redisRateLimiter.isAllowed(rule.getId(), limiterHandle.getReplenishRate(), limiterHandle.getBurstCapacity())
                .flatMap(response -> {
                    if (!response.isAllowed()) {
                        exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
                        Object error = SoulResultWrap.error(SoulResultEnum.TOO_MANY_REQUESTS.getCode(), SoulResultEnum.TOO_MANY_REQUESTS.getMsg(), null);
                        return WebFluxResultUtils.result(exchange, error);
                    }
                    return chain.execute(exchange);
                });
    }


----------
	
	4.3	

	@Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        String pluginName = named();
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
        if (pluginData != null && pluginData.getEnabled()) {
  
            ruleLog(rule, pluginName);
            return doExecute(exchange, chain, selectorData, rule);
        }
        return chain.execute(exchange);
    }
	
   已经加载了RateLimiterPlugin，执行到了RateLimiterPlugin的doExecute方法，
   	
		
    4.4 @Override
    protected Mono<VoiddoExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        final String handle = rule.getHandle();
        final RateLimiterHandle limiterHandle = GsonUtils.getInstance().fromJson(handle, RateLimiterHandle.class);
        return redisRateLimiter.isAllowed(rule.getId(), limiterHandle.getReplenishRate(), limiterHandle.getBurstCapacity())
                .flatMap(response -{
                    if (!response.isAllowed()) {
                        exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
                        Object error = SoulResultWrap.error(SoulResultEnum.TOO_MANY_REQUESTS.getCode(), SoulResultEnum.TOO_MANY_REQUESTS.getMsg(), null);
                        return WebFluxResultUtils.result(exchange, error);
                    }
                    return chain.execute(exchange);
                });
       }

----------

	      继续跟踪，找到方法isAllowed的实现,用redis的lua脚本来存取流量的大小，并进行控制。
	 public Mono<RateLimiterResponse> isAllowed(final String id, final double replenishRate, final double burstCapacity) {
      
        List<String> keys = getKeys(id);
        List<String> scriptArgs = Arrays.asList(replenishRate + "", burstCapacity + "", Instant.now().getEpochSecond() + "", "1");
        Flux<List<Long>> resultFlux = Singleton.INST.get(ReactiveRedisTemplate.class).execute(this.script, keys, scriptArgs);
        return resultFlux.onErrorResume(throwable -> Flux.just(Arrays.asList(1L, -1L)))
                .reduce(new ArrayList<Long>(), (longs, l) -> {
                   ...
                }).doOnError(throwable -> log.error("Error determining if user allowed from redis:{}", throwable.getMessage()));
    }
		
	4.5 我们截取一段request_rate_limiter.lua脚本的代码，看流量怎么控制的 
	
	  local delta = math.max(0, now-last_refreshed)
		//获取需要填满的请求数量
		local filled_tokens = math.min(capacity, last_tokens+(delta*rate))

		//请求数和需要填满的请求数量
		local allowed = filled_tokens >= requested

		local new_tokens = filled_tokens
		local allowed_num = 0
		if allowed then
  			new_tokens = filled_tokens requested
  			allowed_num = 1
    end

	

	从源码的跟踪，我们了解了rate_limiter的执行流程。
	
	
    




	

	


   
	


