# 一：zookeeper简介及安装 #

----------

	# zookeeper简介 #
   zookeper的[官方文档](https://zookeeper.apache.org/ "官方文档地址")对zookeeper的
功能做了描述：zookeeper是一个分布式的协调服务，提供配置中心,域名命名，分布式同步，组服务等，在分布式系统中，微服务很多，服务管理比较困难，比如服务节点的故障，服务节点的新增，
zookeeper来管理实现服务的高可用。	

    # zookeeper的安装 # 
	    zookeeper的安装也比较简单，找一个最新的版本的zookeeper的包，比如下载zookeeper-release-3.6.2.tar，下载之后，找到bin下面的zkServer.cmd，然后双击，
    	zookeeper启动成功。
	

----------
	

# 二：soul集成zookeeper进行数据同步 #

----------

	

 soul-admin项目配置文件application.yml，关掉websocket的默认配置，开启zookeeper的配置
    
 > 	soul:  
> 	  sync:  
>         zookeeper:
>           url: localhost:2181  
>           sessionTimeout: 5000  
>           connectionTimeout: 2000

  soul-bootstrap项目引入依赖包
	>  <dependency>
>       <groupId>org.dromara</groupId>
>        <artifactId>soul-spring-boot-starter-sync-data-zookeeper</artifactId>
>        <version></version>
>      </dependency>
 
  
  soul-bootstrap项目引入依赖包    
	 
> 	soul:  
> 	  sync:  
>         zookeeper:
>           url: localhost:2181  
>           sessionTimeout: 5000  
>           connectionTimeout: 2000
 

启动soul-admin，启动成功
启动soul-bootstrap，启动成功。

 

----------

# 三：zookeeper数据同步流程源码分析 #

----------
 
> 大体的流程：1 soul-admin启动——》全量推送配置数据到zookeeper。  
> 		    2 soul-bootstrap启动——》从zookeeper拉取数据到读取到soul-bootstrapd的内存。  
> 		    3 客户调用——》请求到soul-bootstrap，boot-strap从内存中获取到配置数据
> 		      
 # soul-admin项目推送数据到zookeeper： 
		
		1 首先我们找到这个注入DataSyncConfiguration类下注入ZookeeperDataInit的Bean的方法，发现依赖于先把ZkClient和DataSyncConfiguration注入到容器。
		@Bean
        @ConditionalOnMissingBean(ZookeeperDataInit.class)
        public ZookeeperDataInit zookeeperDataInit(final ZkClient zkClient, final SyncDataService syncDataService) {
            return new ZookeeperDataInit(zkClient, syncDataService);
        }

		2 通过查找引用的方式，我们找到了ZookeeperConfiguration类的注入ZkClient的方法，因为配置文件中有zookeeper的配置，所以ZookeeperProperties已经被注入到容器，ZK也可注入到容器

	@Bean
    @ConditionalOnMissingBean(ZkClient.class)
    public ZkClient zkClient(final ZookeeperProperties zookeeperProp) {
        return new ZkClient(zookeeperProp.getUrl(), zookeeperProp.getSessionTimeout(), zookeeperProp.getConnectionTimeout());
    }
   	
	     3 我们再找Zookeeper依赖注入的SyncDataService，猜测是不是有Zookeeper实现的SyncDataService，注入到容器了，果然我们找到了ZookeeperSyncDataService，
		 @Bean
    public SyncDataService syncDataService(final ObjectProvider<ZkClient> zkClient, final ObjectProvider<PluginDataSubscriber> pluginSubscriber,
                                           final ObjectProvider<List<MetaDataSubscriber>> metaSubscribers, final ObjectProvider<List<AuthDataSubscriber>> authSubscribers) {
        return new ZookeeperSyncDataService(zkClient.getIfAvailable(), pluginSubscriber.getIfAvailable(),
                metaSubscribers.getIfAvailable(Collections::emptyList), authSubscribers.getIfAvailable(Collections::emptyList));
     }
		4 ZookeeperDataInit已经初始化完成，接下来我们看下如何同步的数据，ZookeeperDataInit方法实现了CommondLineRunner,spring容器初始化中，会执行run方法，即同步数据的方法。
	> public class ZookeeperDataInit implements CommandLineRunner
	 @Override
    public void run(final String... args) {
        String pluginPath = ZkPathConstants.PLUGIN_PARENT;
        String authPath = ZkPathConstants.APP_AUTH_PARENT;
        String metaDataPath = ZkPathConstants.META_DATA;
        if (!zkClient.exists(pluginPath) && !zkClient.exists(authPath) && !zkClient.exists(metaDataPath)) {
            syncDataService.syncAll(DataEventTypeEnum.REFRESH);
        }
    }
		
	 我们再进一步看下synAll的实现方法：
	@Override
    public boolean syncAll(final DataEventTypeEnum type) {
        List<PluginData> pluginDataList = pluginService.listAll();
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.PLUGIN, type, pluginDataList));
        ...
    }

	eventPublisher执行了publishEvent，定位到DataChangedEventDispatcher实现了ApplicationListener，用onApplicationEvent监听到事件，进行处理
	

	public void onApplicationEvent(final DataChangedEvent event) {
        for (DataChangedListener listener : listeners) {
            switch (event.getGroupKey()) {
                case APP_AUTH:
                    listener.onAppAuthChanged((List<AppAuthData>) event.getSource(), event.getEventType());
                    break;
               ......
            }
        }
    }
	
	我们开始了Zookeeper的配置，从容器中拿到listener是ZookeeperDataChangedListener，
	进入到这个类，这个是核心类，获取到zk的path，往zk里去同步数据。
	 public void onPluginChanged(final List<PluginData> changed, final DataEventTypeEnum eventType) {
        for (PluginData data : changed) {
			
            final String pluginPath = ZkPathConstants.buildPluginPat
            upsertZkNode(pluginPath, data);
			....
        }
    }

	至此，soul-admin里面往zk同步数据代码就到这里

----------
	
	
# soul-bootsrap从zookeeper拉取配置信息加载到缓存
	
> soul-bootstrap项目引入soul-spring-boot-starter-sync-data-zookeeper，我们找到这个项目的源码，找到ZookeeperSyncDataConfiguration这个核心配置类，

    @Bean
    public SyncDataService syncDataService(final ObjectProvider<ZkClientzkClient, final ObjectProvider<PluginDataSubscriberpluginSubscriber,
       final ObjectProvider<List<MetaDataSubscriber>metaSubscribers, final ObjectProvider<List<AuthDataSubscriber>authSubscribers) {
    log.info("you use zookeeper sync soul data.......");    
    return new ZookeeperSyncDataService(zkClient.getIfAvailable(),    pluginSubscriber.getIfAvailable(),
    metaSubscribers.getIfAvailable(Collections::emptyList), authSubscribers.getIfAvailable(Collections::emptyList));
    }
    

----------
	
从日志可以看出这是开始用zookeeper同步数据。进入到实例话方法：


     public ZookeeperSyncDataService(final ZkClient zkClient, final PluginDataSubscriber pluginDataSubscriber,
                                    final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {
        this.zkClient = zkClient;
        this.pluginDataSubscriber = pluginDataSubscriber;
        this.metaDataSubscribers = metaDataSubscribers;
        this.authDataSubscribers = authDataSubscribers;
        watcherData();
        watchAppAuth();
        watchMetaData();
    }

----------
	然后我们进入watcherData方法内部
	 private void watcherData() {
        final String pluginParent = ZkPathConstants.PLUGIN_PARENT;
        List<String> pluginZKs = zkClientGetChildren(pluginParent);
        for (String pluginName : pluginZKs) {
			//获取到数据，将数据写入缓存
            watcherAll(pluginName);
        }
        zkClient.subscribeChildChanges(pluginParent, (parentPath, currentChildren) -> {
            if (CollectionUtils.isNotEmpty(currentChildren)) {
                for (String pluginName : currentChildren) {
                    watcherAll(pluginName);
                }
            }
        });
    }
	进入到watchAll方法：
	private void watcherAll(final String pluginName) {
        watcherPlugin(pluginName);
        watcherSelector(pluginName);
        watcherRule(pluginName);
    }
	然后进入到一种的监控plugin方法，这个是核心的方法，
	private void watcherPlugin(final String pluginName) {
        String pluginPath = ZkPathConstants.buildPluginPath(pluginName);
        if (!zkClient.exists(pluginPath)) {
            zkClient.createPersistent(pluginPath, true);
        }
		//写入缓存数据
        cachePluginData(zkClient.readData(pluginPath));
        subscribePluginDataChanges(pluginPath, pluginName);
    }
	
	//可以看到把数据写入了本地的缓存PLUGIN_MAP，是一个Map。
	public void cachePluginData(final PluginData pluginData) {
        Optional.ofNullable(pluginData).ifPresent(data -> PLUGIN_MAP.put(data.getName(), data));
    }
	

----------

	//接下来我们看下接口调用soul-bootstrap网关，调用到AbstractPlugin, 从缓存中取数据，取不到的话，直接返回

	public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        String pluginName = named();
		//从缓存中取数据，不做处理直接返回。
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
        if (pluginData != null && pluginData.getEnabled()) {
            final Collection<SelectorData> selectors = BaseDataCache.getInstance().obtainSelectorData(pluginName);
            if (CollectionUtils.isEmpty(selectors)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
     
----------

	

	从源码的跟踪，追溯了初始化soul-admin怎么把数据写入到zookeeper，soul-bootstrap项目启动
	从zookeeper从读取数据，加载到自己内存的过程。
	soul-admin增删改配置数据之后，soul-bootstrap数据及时生效的数据同步，也基本一致，这里我们不再赘述。
	
    




	

	


   
	


