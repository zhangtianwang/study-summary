        # soul网关和soul-admin通过http长轮询进行数据同步 #

#一：  先聊下大致的流程  

	soul-bootstrap 启动要加载配置数据，开始了http长轮询的配置之后，会调用soul-admin的
	获取配置数据的接口，soul-admin会启动一个异步的线程池，异步处理并返回配置数据。

# 二： 项目启动配置 #

	1 soul-admin 开启http配置
		 sync:
      		http:
        		enabled: true

	2 soul-bootstrap 开启配置：
		sync:
           http:
             url : http://localhost:9095

	启动soul-bootstrap和soul-admin项目，项目启动成功。

#三： 源码剖析

	1 先看下HttpSyncDataConfiguration,这个是http长轮询的配置启动类，项目的启动的时候
	  注入到Spring容器

	@Bean
    public SyncDataService httpSyncDataService(final ObjectProvider<HttpConfig> httpConfig, final ObjectProvider<PluginDataSubscriber> pluginSubscriber,
                                           final ObjectProvider<List<MetaDataSubscriber>> metaSubscribers, final ObjectProvider<List<AuthDataSubscriber>> authSubscribers) {
        log.info("you use http long pull sync soul data");
        return new HttpSyncDataService(Objects.requireNonNull(httpConfig.getIfAvailable()), Objects.requireNonNull(pluginSubscriber.getIfAvailable()),
                metaSubscribers.getIfAvailable(Collections::emptyList), authSubscribers.getIfAvailable(Collections::emptyList));
    }
	
----------
	2 进入到HttpSyncDataService，这个是Http长轮询同步数据的核心类，start是初始方法。
	private void start() {
		  ...
		  //第一次执行，会去soul-admin接口获取所有的配置数据，获取到之后
		  //放置到网关的jvm内存
		  this.fetchGroupConfig(ConfigGroupEnum.values());
		  //开启异步线程，去监听soul-admin配置的变化
          this.serverList.forEach(server -> this.executor.execute(new HttpLongPollingTask(server)));
    }

----------
	3 接下来我们看下HttpLongPollingTask，里面的核心方法run，不断的去循环执行，执行里面的
	核心方法doLongPolling。
	 @Override
        public void run() {
            while (RUNNING.get()) {
                for (int time = 1; time <= retryTimes; time++) {
                    try {
                        doLongPolling(server);
                    } catch (Exception e) {
                        ...
                    }
                }
            }
        }
    }

----------
	4 接下来我们看下doLongPolling做了啥，先调用configs/listener接口，看是否有变动的配置，如果有的情况下，会调用doFetchGroupConfig，获取配置数据，加载到内存

		private void doLongPolling(final String server) {
        ...组装参数信息
        HttpEntity httpEntity = new HttpEntity(params, headers);
        String listenerUrl = server + "/configs/listener";
        log.debug("request listener configs: [{}]", listenerUrl);
        JsonArray groupJson = null;
        try {
            String json = this.httpClient.postForEntity(listenerUrl, httpEntity, String.class).getBody();
            groupJson = GSON.fromJson(json, JsonObject.class).getAsJsonArray("data");
        } catch (RestClientException e) {
           ...
        }
        if (groupJson != null) {
            // fetch group configuration async.
            ConfigGroupEnum[] changedGroups = GSON.fromJson(groupJson, ConfigGroupEnum[].class);
            if (ArrayUtils.isNotEmpty(changedGroups)) {
                log.info("Group config changed: {}", Arrays.toString(changedGroups));
				//重新获取配置，加载到网关的内存
                this.doFetchGroupConfig(server, changedGroups);
            }
        }
    }

	

----------
	5 接下来我们看下soul-admin的2个关键接口，一个是 configs/listener，一个是
	configs/fetch。
	
	configs/listener接口的核心方法:
	
	public void doLongPolling(final HttpServletRequest request, final HttpServletResponse response) {

        // 通过md5比较，看下soul-admin和soul-bootstrap的配置是否一致。
        List<ConfigGroupEnum> changedGroup = compareChangedGroup(request);
        String clientIp = getRemoteIp(request);

        //如果有数据变更，接口立刻返回
        if (CollectionUtils.isNotEmpty(changedGroup)) {
            this.generateResponse(response, changedGroup);
            log.info("send response with the changed group, ip={}, group={}", clientIp, changedGroup);
            return;
        }

        ...
        //没有数据变更，阻塞客户端线程，一直等待，防止客户端不断的连接服务端
        scheduler.execute(new LongPollingClient(asyncContext, clientIp, HttpConstants.SERVER_MAX_HOLD_TIMEOUT));
    }

	

----------
	6 configs/fetch接口的核心方法,从configs/listener，拿到配置是否发生变化
	发生变化，则会调用configs/fetch接口，接口方法是 fetchConfigs
	
	public SoulAdminResult fetchConfigs(@NotNull final String[] groupKeys) {
        Map<String, ConfigData<?>> result = Maps.newHashMap();
        for (String groupKey : groupKeys) {
            ConfigData<?> data = longPollingListener.fetchConfig(ConfigGroupEnum.valueOf(groupKey));
            result.put(groupKey, data);
        }
        return SoulAdminResult.success(SoulResultMessage.SUCCESS, result);
    }
	
	然后我们进入到 longPollingListener.fetchConfig 的方法：
	
	public ConfigData<?> fetchConfig(final ConfigGroupEnum groupKey) {
        ConfigDataCache config = CACHE.get(groupKey.name());
        switch (groupKey) {
            case RULE:
                List<RuleData> ruleList = GsonUtils.getGson().fromJson(config.getJson(), new TypeToken<List<RuleData>>() {
                }.getType());
                return new ConfigData<>(config.getMd5(), config.getLastModifyTime(), ruleList);
			...
			
	  	}
		以Rule更新为例，从本地缓存取到最新的Rule的配置数据，然后返回给soul-bootstrap
		soul-bootstrap获取到数据之后，更新到自己本地的缓存，至此流程走完。
	
	

	
	2 


	

	

	