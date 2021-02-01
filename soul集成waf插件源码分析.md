# 一：soul的waf简单介绍 #

waf,全称web应用防护系统，也叫web防火墙，为了保障web应用的安全，可以做一些安全策略，以soul集成的soul-plugin-waf插件为例，我们可以设置白名单，可以对某些请求设置允许或者禁止访问，
也可以设置放回的http响应的状态码
 
			
	
----------
	

# 二：soul集成waf #

----------

1. soul-admin plugin模块开启waf这个插件, 同时PluginList为waf增加selector和rule。	      
2. 启动soul-admin，启动成功
3. 启动soul-bootstrap，启动成功,加载了RateLimiterPlugin  
	控制台输出： load plugin:[waf] [org.dromara.soul.plugin.waf.WafPlugin]，
	启动成功
4. 启动soul-examples-http 启动成功。
 
----------

# 三：waf插件流程源码分析 #

----------
大体的流程：
>   soul-bootstrap启动，websocket发起连接，连接上soul-admin，soul-admin通过websocket把数据推送到Socket的Client端soul-bootstrap，soul-bootstrap获取数据并加载到内存      
>  客户调用——》请求到soul-bootstrap，soul-bootstrap开始执行plugins，
> 执行到WafPlugin的execute方法，拿后台配置的Selector和Rule等数据和客户请求的url进行规则匹配，如果是加入了白名单，或者允许访问，则通过这个校验，跳转到下一个plugin，继续执行。
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


	接着继续调试，找到了WafPlugin，执行到doExecute方法，然后对重点方法重点剖析。

	protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
		//从缓存中读取waf配置
        WafConfig wafConfig = Singleton.INST.get(WafConfig.class);
        if (Objects.isNull(selector) && Objects.isNull(rule)) {
			//看是否有白名单，如果有，直接跳转到下一个plugin去执行
            if (WafModelEnum.BLACK.getName().equals(wafConfig.getModel())) {
                return chain.execute(exchange);
            }
			//否则返回错误
            exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
            Object error = SoulResultWrap.error(403, Constants.REJECT_MSG, null);
            return WebFluxResultUtils.result(exchange, error);
        }
        String handle = rule.getHandle();
        WafHandle wafHandle = GsonUtils.getInstance().fromJson(handle, WafHandle.class);
		//如果没有设置访问权限，默认通过
        if (Objects.isNull(wafHandle) || StringUtils.isBlank(wafHandle.getPermission())) {
            log.error("waf handler can not configuration：{}", handle);
            return chain.execute(exchange);
        }
		//如果是拒绝，则返回失败。
        if (WafEnum.REJECT.getName().equals(wafHandle.getPermission())) {
            exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
            Object error = SoulResultWrap.error(Integer.parseInt(wafHandle.getStatusCode()), Constants.REJECT_MSG, null);
            return WebFluxResultUtils.result(exchange, error);
        }
        return chain.execute(exchange);
    }


----------

	

	从源码的跟踪，我们了解了waf插件的的执行过程。
	
	
    




	

	


   
	


