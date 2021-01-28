# Soul的divide和spring cloud插件底层原理初探 #
# 1 divide插件	
# 1.1 如何自己实现一个divide插件   #
		
		divide插件，主要用来对流量进行控制，可以做负载均衡，对恶意请求的过滤和拦截。
	如何让我来设计的话，会考虑一下几个方面：
		1.1.1 服务启动，通知网关，网关获取到可用的服务列表, divide做的流量控制也是基于
	服务的可用状态，为了保证服务可用，可以设定一个定时器，定期探活，如果发现调用的服务不可用，从服务列表剔除掉，
	新流量进来，会不再切向这个服务器。更近一步，为了能及时发现并处理，可以为服务设置最小数量的阈值，如果超过阈值，
	可以进行通知，从而保证服务的可靠性。`
	
 		1.1.2 可以设置不同的分配策略，比较常见的有，平均分配，随机分配，也可以根据服务器的响应时间计算服务器的处理能力，
      有权重的分配，还可以根据服务器的同时处理的请求数的多少，将流量去往流量小的服务器去倾斜，后2种策略
      需要网关做计算，把数据记录下来，同时这2种策略都有时效性。
	` 
# 1.2 我们通过源码来看下divide插件怎么实现 #

	我们先看下DividePlugin的核心方法doExecute

	@Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        //获取可用的服务列表
        final List<DivideUpstream> upstreamList = UpstreamCacheManager.getInstance().findUpstreamListBySelectorId(selector.getId());
        ...
		//获取clientId，根据后台配置的负载策略，从服务列表找到一个服务。
        final String ip = Objects.requireNonNull(exchange.getRequest().getRemoteAddress()).getAddress().getHostAddress();
        DivideUpstream divideUpstream = LoadBalanceUtils.selector(upstreamList, ruleHandle.getLoadBalance(), ip);
        
        //构建真实服务的地址 
        String domain = buildDomain(divideUpstream);
        String realURL = buildRealURL(domain, soulContext, exchange);
       	...
		//继续执行下一个Plugin
        return chain.execute(exchange);
    }


----------

	我们重点看下LoadBalanceUtils.selector 这个方法：根据后台配置的algorithm，通过
	找到对应的LoadBalance的实现类，

	 public static DivideUpstream selector(final List<DivideUpstream> upstreamList, final String algorithm, final String ip) {
        LoadBalance loadBalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getJoin(algorithm);
        return loadBalance.select(upstreamList, ip);
    }

	以随机算法为例，实现类为RandomLoadBalance，分为2种，一种是带权重的随机，一种是权重值
	相同的完全随机，根据传入的服务列表，从列表中随机找到一个服务。
	 @Override
    public DivideUpstream doSelect(final List<DivideUpstream> upstreamList, final String ip) {
        int totalWeight = calculateTotalWeight(upstreamList);
        boolean sameWeight = isAllUpStreamSameWeight(upstreamList);
        if (totalWeight > 0 && !sameWeight) {
            return random(totalWeight, upstreamList);
        }
        //完全随机算法
        return random(upstreamList);
    }

	

----------
# 1.3 探寻soul的探活机制 #
	
	由soul的文档获知，soul.upstream.check 设置为true,
	soul.upstream.scheduledTime，开启定时任务，定时去探活。
	我们根据配置首先定位到UpstreamCheckService这个类，找到核心方法setup。
	
	一开始，探活是个频繁的操作，防止对数据库压力过大,先从数据库读取配置并放到缓存里，
	之后的操作，从缓存中读取。
	 public void setup() {
        PluginDO pluginDO = pluginMapper.selectByName(PluginEnum.DIVIDE.getName());
        if (pluginDO != null) {
            List<SelectorDO> selectorDOList = selectorMapper.findByPluginId(pluginDO.getId());
            for (SelectorDO selectorDO : selectorDOList) {
                List<DivideUpstream> divideUpstreams = GsonUtils.getInstance().fromList(selectorDO.getHandle(), DivideUpstream.class);
                if (CollectionUtils.isNotEmpty(divideUpstreams)) {
					//放置到缓存
                    UPSTREAM_MAP.put(selectorDO.getName(), divideUpstreams);
                }
            }
        }
		//启动定时任务，定时去执行扫描，
        if (check) {
            new ScheduledThreadPoolExecutor(Runtime.getRuntime().availableProcessors(), SoulThreadFactory.create("scheduled-upstream-task", false))
                    .scheduleWithFixedDelay(this::scheduled, 10, scheduledTime, TimeUnit.SECONDS);
        }
    }
	

----------


	然后我们进入到扫描方法scheduled，
		private void scheduled() {
        if (UPSTREAM_MAP.size() > 0) {
            UPSTREAM_MAP.forEach(this::check);
        }
    }
	
	可以看到最终的核心调用方式是check，从Map里面变量所有的服务地址，用IP和端口号
	去探活，如果可达，则把selector对应的服务列表放到内存，如果不可达，则移除掉。

	private void check(final String selectorName, final List<DivideUpstream> upstreamList) {
        List<DivideUpstream> successList = Lists.newArrayListWithCapacity(upstreamList.size());
        for (DivideUpstream divideUpstream : upstreamList) {
            final boolean pass = UpstreamCheckUtils.checkUrl(divideUpstream.getUpstreamUrl());
        }
        if (successList.size() == upstreamList.size()) {
            return;
        }
        if (successList.size() > 0) {
            UPSTREAM_MAP.put(selectorName, successList);
            updateSelectorHandler(selectorName, successList);
        } else {
            UPSTREAM_MAP.remove(selectorName);
            updateSelectorHandler(selectorName, null);
        }
    }
	

----------


	checkUrl，探活的时候，区分是用IP和域名访问，如果是IP直接用socket连接，如果是域名
	直接请求，看下服务是否可达。
	  if (checkIP(url)) {
            String[] hostPort;
            if (url.startsWith(HTTP)) {
                final String[] http = StringUtils.split(url, "\\/\\/");
                hostPort = StringUtils.split(http[1], Constants.COLONS);
            } else {
                hostPort = StringUtils.split(url, Constants.COLONS);
            }
            return isHostConnector(hostPort[0], Integer.parseInt(hostPort[1]));
        } else {
            return isHostReachable(url);
       }
	
	探活的源码分析完成。

	
	
	

		
