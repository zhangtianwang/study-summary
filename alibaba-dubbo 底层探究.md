# 一：自己用soul如何来实现alibaba-dubbo的调用 #
	soul网关支持dubbo调用，首先要有一个dubbo服务，dubbo服务注册到zookeeper上，            一个客户端请求进来，网关层
	根据设置的路由规则能定位到这个dubbo服务，然后模拟一个dubbo Client来完成dubbo的调用，dubbo服务处理之后，网关收到服务结果，最后把结果返回客户端。soul网关在整个调用过程中起到一个代理的作用。	

# 二： # dubbo集成源码解析
	 1 soul-bootstrap与soul-admin数据同步，使用socket同步，先看下同步的时候做了什么 
	
	//socketClient接收消息，比如接收到是plugin元数据信息，交给handleResult方法处理
	@Override
    public void onMessage(final String result) {
        handleResult(result);
    }
	 private void handleResult(final String result) {
        ...
		//将传输的内容反序列化之后，调用executor方法
        websocketDataHandler.executor(groupEnum, json, eventType);
    }
	

----------

	接下来，我们进入到executor方法，看看到底做了啥，以刷新全量数据为例：
	public void handle(final String json, final String eventType) {
		...
        switch (eventTypeEnum) {
                case REFRESH:
                case MYSELF:
                    doRefresh(dataList);
                    break;
		}
	}	
	最终我们找到了doRefresh方法，遍历加载的插件列表，
	@Override
    protected void doRefresh(final List<PluginData> dataList) {
        pluginDataSubscriber.refreshPluginDataSelf(dataList);
        dataList.forEach(pluginDataSubscriber::onSubscribe);
    }
	我们找到了CommonPluginDataSubscriber的subscribeDataHandler方法并最终把数据放到了缓存
	 @Override
    public void onSubscribe(final PluginData pluginData) {
        subscribeDataHandler(pluginData, DataEventTypeEnum.UPDATE);
    }
	
	socket同步最终是把数据放到了缓存
	private <T> void subscribeDataHandler(final T classData, final DataEventTypeEnum dataType) {
        Optional.ofNullable(classData).ifPresent(data -> {
            if (data instanceof PluginData) {
                PluginData pluginData = (PluginData) data;
                 BaseDataCache.getInstance().cachePluginData(pluginData);
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.handlerPlugin(pluginData));
            }
			...
        });
    }
	

----------
	2 我们首先看下AlibabaDubboPlugin的doExecute方法，获取请求的参数，url等信息，都封装到context中，然后调用genericInvoker方法

	 @Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
		//获取请求的参数，url等信息，都封装到context中
        String body = exchange.getAttribute(Constants.DUBBO_PARAMS);
        SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        MetaData metaData = exchange.getAttribute(Constants.META_DATA);
        //调用
        Object result = alibabaDubboProxyService.genericInvoker(body, metaData);
        if (Objects.nonNull(result)) {
            exchange.getAttributes().put(Constants.DUBBO_RPC_RESULT, result);
        } else {
            exchange.getAttributes().put(Constants.DUBBO_RPC_RESULT, Constants.DUBBO_RPC_RESULT_EMPTY);
        }
       ...
    }

	然后我们看下genericInvoker这个方法，这个方法是核心方法，发起了dubbo调用。首先拿到
	AlibabaDubboProxyService的调用类GenericService这个dubbo调用的引用,然后把参数等解析到，就可以执行dubbo调用，然后
	拿到返回结果。

	public Object genericInvoker(final String body, final MetaData metaData) throws SoulException {
        ReferenceConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
        GenericService genericService = reference.get();
        try {
            
            return genericService.$invoke(metaData.getMethodName(), pair.getLeft(), pair.getRight());
        } catch (GenericException e) {
            log.error("dubbo invoker have exception", e);
            throw new SoulException(e.getExceptionMessage());
        }
    }

	这个就是soul集成dubbo调用的大致的流程。