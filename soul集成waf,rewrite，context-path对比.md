# waf,rewrite和context-path插件对比 #
# 1 waf,rewrite，context-path插件简单介绍	

1. waf，全称Web Application Firewall，web应用防火墙，用来保护网站安全，当恶意攻击的求，对网站安全构成威胁时，可以启用waf插件，查找到恶意请求，比如通过恶意请求的链接地址，请求头的请求参数，设置状态码500，网关不再转发，直接把请求拦截，返回错误的响应码。
2. rewrite，对请求地址进行重写，比如将 /http/test/**，设置成 /order/findById 这个地址，启用rewrite插件，则所有以http/test开头的地址，网关将其转换成/order/findById这个地址进行转发调用
3. context-path,插件可以对某些context-path的链接进行拦截过滤，也可以找到实际的地址进行请求。
    		
# 2 我们通过源码来看下waf插件 #

	SoulWebPlugin通过调用链调用各个插件的过程暂时忽略，我们重点看waf插件的核心方法doExecute，调用之前我们要做一些配置操作:
	-   我们在后台开启了waf Plugin，
	- 	对waf设置了selector和rule,
	- 	同时soul-bootstrap引入soul-spring-boot-starter-plugin-waf。
	
	SoulWebPlugin获取到所有插件，然后通过作用链的调用去调用各个插件，waf是对非法请求做拦截，设置插件执行的顺序的时候，waf插件需要放在前面。
	我们先看下 SoulWebHandler 如何设定插件执行顺序的
	@Bean("webHandler")
    public SoulWebHandler soulWebHandler(final ObjectProvider<List<SoulPlugin>> plugins) {
        List<SoulPlugin> pluginList = plugins.getIfAvailable(Collections::emptyList);
		//d对插件列表进行排序，
        final List<SoulPlugin> soulPlugins = pluginList.stream()
                .sorted(Comparator.comparingInt(SoulPlugin::getOrder)).collect(Collectors.toList());
        return new SoulWebHandler(soulPlugins);
    }
	
	通过getOrder，定位到PluginEnum对各个插件的调用顺序进行了设定，在waf插件执行的一个是global和sign插件，像rate_limiter，divide等插件都放在waf之后，
	当然这个可以调整成在soul-admin配置，也算是一个待优化的地方。

	
	然后我们进入到插件的核心方法,如果设定了既没有设定selector和rule,没有设置成BLACK模式
	，则请求直接返回错误，再到之后如果设定了拒绝模式，也是返回错误，不会继续执行下面的插件，
	返回错误的响应。
	@Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        WafConfig wafConfig = Singleton.INST.get(WafConfig.class);
		
        if (Objects.isNull(selector) && Objects.isNull(rule)) {
			//如果设定了waf设定了白名单模式，则请求可以继续执行。
			if (WafModelEnum.BLACK.getName().equals(wafConfig.getModel())) {
                return chain.execute(exchange);
            }
            ...
			//如果没有设定selector和rule，则会直接返回错误
        }
        String handle = rule.getHandle();
        WafHandle wafHandle = GsonUtils.getInstance().fromJson(handle, WafHandle.class);
		//增加校验，必须设定处理方式，是拒绝还是允许以及状态码
        if (Objects.isNull(wafHandle) || StringUtils.isBlank(wafHandle.getPermission())) {
            log.error("waf handler can not configuration：{}", handle);
            return chain.execute(exchange);
        }
		//如果waf的处理方式设置了reject，则直接返回错误不继续执行。
        if (WafEnum.REJECT.getName().equals(wafHandle.getPermission())) {
            exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
            Object error = SoulResultWrap.error(Integer.parseInt(wafHandle.getStatusCode()), Constants.REJECT_MSG, null);
            return WebFluxResultUtils.result(exchange, error);
        }
        return chain.execute(exchange);
    }
  可以看到waf插件就是对做一些安全认证，白名单内的可以直接绕过权限验证，对一些请求可以通过设置拒绝策略和状态码，拒绝执行。

----------
# 3 rewirte plugin 源码解析 #
rewrite是把请求地址经过网关处理，转成可以直接调用服务接口的地址，来进行调用,执行顺序也比较靠前，我们先看下RewritePlugin的核心方法doExecute方法，方法比较简单，要看是否配置了重定向的地址，如果没有配返回错误，如果配置了，将重写的地址，写到exchage的attribute中。
	
@Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        String handle = rule.getHandle();
        final RewriteHandle rewriteHandle = GsonUtils.getInstance().fromJson(handle, RewriteHandle.class);
        if (Objects.isNull(rewriteHandle) || StringUtils.isBlank(rewriteHandle.getRewriteURI())) {
            log.error("uri rewrite rule can not configuration：{}", handle);
            return chain.execute(exchange);
        }
        exchange.getAttributes().put(Constants.REWRITE_URI, rewriteHandle.getRewriteURI());
        return chain.execute(exchange);
    }
	然后我们看下Constants.REWRITE_URI，通过查找引用，我们定位到DividePlugin的buildRealURL方法，取出重定向的url,转化成直接可调用的服务地址，然后放到exchage的attribute中，在后面WebClientPlugin进行调用。
	
	  private String buildRealURL(final String domain, final SoulContext soulContext, final ServerWebExchange exchange) {
        String path = domain;
		//获取rewrite的url
        final String rewriteURI = (String) exchange.getAttributes().get(Constants.REWRITE_URI);
		//转换成真实的地址
        if (StringUtils.isNoneBlank(rewriteURI)) {
            path = path + rewriteURI;
        } else {
            final String realUrl = soulContext.getRealUrl();
            if (StringUtils.isNoneBlank(realUrl)) {
                path = path + realUrl;
            }
        }
		//拼接请求的参数
        String query = exchange.getRequest().getURI().getQuery();
        if (StringUtils.isNoneBlank(query)) {
            return path + "?" + query;
        }
        return path;
    }
	
	rewrite做的事情也比较简单，将网关的请求地址按照规则重写成可调用的服务地址，然后在其他插件中去调用。
	

----------
# 4 Context path的源码 #
	这个插件是对请求地址的context path进行过滤，如果不满足context path，则对请求进行拦截，然后解析出真实的地址，进行调用。

	 @Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        final String handle = rule.getHandle();
        final ContextMappingHandle contextMappingHandle = GsonUtils.getInstance().fromJson(handle, ContextMappingHandle.class);
		//没有配置上下文的规则，继续往下执行，但是还记error日志，此处难以理解。
        if (Objects.isNull(contextMappingHandle) || StringUtils.isBlank(contextMappingHandle.getContextPath())) {
            log.error("context path mapping rule configuration is null ：{}", rule);
            return chain.execute(exchange);
        }
        //对比接口的请求地址和后台配置的context path是否一致。
        if (!soulContext.getPath().startsWith(contextMappingHandle.getContextPath())) {
            Object error = SoulResultWrap.error(SoulResultEnum.CONTEXT_PATH_ERROR.getCode(), SoulResultEnum.CONTEXT_PATH_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
		//查找到真实的地址。
        this.buildContextPath(soulContext, contextMappingHandle);
        return chain.execute(exchange);
    }
	
ContextPathMappingPlugin 这个插件理解，一些带有某个context path的请求，可以转向某个地址，和rewrite plugin比较接近。
	
	
	



	
	
	

		
