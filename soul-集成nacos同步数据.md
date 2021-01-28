# 一：nacos简介及安装 #

----------


	# nacos简介 #
	
	Nacos 致力于发现、配置和管理微服务,提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。


    # nacos的安装 # 
	    下载nacos-server-1.1.4.zip，下载之后，解压，然后双击startup.cmd，
    	启动成功。
		然后访问： http://127.0.0.1:8848/nacos/index.html，进入到nacos的管理后台。
	

----------
	

# 二：soul如何集成nacos进行数据同步 #

----------

	

 soul-admin和soul-bootstrap配置文件引入nacos的配置
    
	 nacos:
      url: localhost:8848
      namespace: 1c10d748-af86-43b9-8265-75f487d20c6c
      acm:
        enabled: false
        endpoint: acm.aliyun.com
        namespace:
        accessKey:
        secretKey:

  soul-admin和soul-bootstrap项目引入依赖包
    
	 <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-client</artifactId>
            <version>${nacos-client.version}</version>
        </dependency>
 
  分别启动soul-admin，soul-bootstrap，项目启动成功。
  
	

----------

# 三：nacos数据同步流程源码分析 #

----------
 
> 大体的流程：1 soul-admin同步数据到nacos的  
> 		    2 soul-bootstrap启动——》nacosr拉取数据到读取到本地缓存。  
> 		    3 客户调用——》请求到soul-bootstrap，boot-strap从内存中获取到配置数据
> 		      
 # soul-admin项目推送数据到nacos： 
		
		1 update一条rule数据，会把rule更新到数据库，同时publishEvent
	 	private void publishEvent(final RuleDO ruleDO, final    List<RuleConditionDTO> ruleConditions) {
              ...
              eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.RULE, DataEventTypeEnum.UPDATE,
                Collections.singletonList(RuleDO.transFrom(ruleDO, pluginDO.getName(), conditionDataList))));
    }

----------

	 2 NacosDataChangedListener 实现了DataChangedListener,同时开始了nacos的配置，
	 项目启动时，会把NacosDataChangedListener注入到容器，由于DataChangedListener实现了ApplicationListener, 监听到Spring的publishEvent事件

	 public void onApplicationEvent(final DataChangedEvent event) {
        for (DataChangedListener listener : listeners) {
            switch (event.getGroupKey()) {
                ...
                case RULE:
					//开始调用NacosDataChangedListener的onRuleChanged方法
                    listener.onRuleChanged((List<RuleData>) event.getSource(), event.getEventType());
                    break;
			}
	   }
    

----------
	3 NacosDataChangeListener类 做了两个事，一个是更新到本地缓存，另一个是把配置发布出去。
	public void onRuleChanged(final List<RuleData> changed, final DataEventTypeEnum eventType) {
		//更新到本地缓存
        updateRuleMap(getConfig(RULE_DATA_ID));
		//发布配置
        publishConfig(RULE_DATA_ID, RULE_MAP);
    }
	
	

----------
	4 NacosConfigService 类最终是调用封装类，组装参数，通过http调用将参数信息发布到nacos
	private boolean publishConfigInner(String tenant, String dataId, String group, String tag, String appName, String betaIps, String content) throws NacosException {
        //拼装参数和地址
        content = cr.getContent();
        String url = "/v1/cs/configs";
        List<String> params = new ArrayList();
        params.add("dataId");
        params.add(dataId);
        params.add("group");
        params.add(group);
        params.add("content");
        params.add(content);
        ...

        try {
			//走接口调用将，参数信息提交到nacos配置中心
            result = this.agent.httpPost(url, headers, params, this.encode, 3000L);
        } catch (IOException var14) {
          
        }
		结果处理
		....
       
    }


	至此，通过跟踪代码，把数据发布到了nacos的配置中心

----------
	
	
# soul-bootsrap从Nacos拉取配置数据，并监控后台数据的变化缓存数据同步更新
	
> soul-bootstrap项目引入soul-spring-boot-starter-sync-data-nacos，找到  NacosSyncDataConfiguration这个核心配置类，soul-bootstrap启动，会把NacosSyncDataConfiguration下的nacosSyncDataService注入到容器，调用NacosSyncDataService的改造函数

    @Bean
    public SyncDataService nacosSyncDataService(final ObjectProvider<ConfigService> configService, final ObjectProvider<PluginDataSubscriber> pluginSubscriber,final ObjectProvider<List<MetaDataSubscriber>> metaSubscribers, final ObjectProvider<List<AuthDataSubscriber>> authSubscribers) {
        log.info("you use nacos sync soul data.......");
        return new NacosSyncDataService(configService.getIfAvailable(), pluginSubscriber.getIfAvailable(),
                metaSubscribers.getIfAvailable(Collections::emptyList), authSubscribers.getIfAvailable(Collections::emptyList));
    }

    

----------
	NacosSyncDataService构造函数进行初始化，进入start函数，watcherData方法是核心的
	方法
	public NacosSyncDataService(final ConfigService configService, final PluginDataSubscriber pluginDataSubscriber,
                                final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {

        super(configService, pluginDataSubscriber, metaDataSubscribers, authDataSubscribers);
        start();
    }

    /**
     * Start.
     */
    public void start() {
        watcherData(PLUGIN_DATA_ID, this::updatePluginMap);
        watcherData(SELECTOR_DATA_ID, this::updateSelectorMap);
        watcherData(RULE_DATA_ID, this::updateRuleMap);
        watcherData(META_DATA_ID, this::updateMetaDataMap);
        watcherData(AUTH_DATA_ID, this::updateAuthMap);
    }

----------
	然后我们进入watcherData方法，首先创建了一个listener对象，receiveConfigInfo方法接收
	change事件，执行 this::updateRuleMap 这个方法。
	 protected void watcherData(final String dataId, final OnChange oc) {
        Listener listener = new Listener() {
            @Override
            public void receiveConfigInfo(final String configInfo) {
                oc.change(configInfo);
            }

            @Override
            public Executor getExecutor() {
                return null;
            }
        };
        oc.change(getConfigAndSignListener(dataId, listener));
        LISTENERS.getOrDefault(dataId, new ArrayList<>()).add(listener);
    }
	

----------

	//接下来我们看下updateRuleMap这个方法，方法里面先执行了删除缓存，后重新写入本地缓存。

	protected void updateRuleMap(final String configInfo) {
        try {
            List<RuleData> ruleDataList = GsonUtils.getInstance().toObjectMapList(configInfo, RuleData.class).values()
                    .stream().flatMap(Collection::stream)
                    .collect(Collectors.toList());
            ruleDataList.forEach(ruleData -> Optional.ofNullable(pluginDataSubscriber).ifPresent(subscriber -> {
				//删除缓存
                subscriber.unRuleSubscribe(ruleData);
				//更新缓存
                subscriber.onRuleSubscribe(ruleData);
            }));
        } catch (JsonParseException e) {
            log.error("sync rule data have error:", e);
        }
    }
     
----------

	

	从源码的跟踪，追溯了初始化soul-admin怎么把数据写入到nacos，soul-bootstrap项目启动
	从nacos从读取数据，加载到自己内存的过程。
	
	
    




	

	


   
	


