---
layout: post
title: Nacos 作为配置中心的使用
date: 2023-02-02
categories: blog
description: 文章金句。
---

nacos 是什么，它有什么用？大家可以看[官网的介绍](https://nacos.io/zh-cn/docs/what-is-nacos.html)。
总之，nacos 已经被越来越多的开发者所青睐。今天就和大家一起来看看。
这篇文章主要是从源码层面分析一下 nacos 做配置中心时 client 端的一些设计。

## 拉取远程 nacos 配置

拉取配置的入口在<code>NacosConfigBootstrapConfiguration</code>类。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnProperty(name = "spring.cloud.nacos.config.enabled", matchIfMissing = true)
public class NacosConfigBootstrapConfiguration {

	@Bean
	@ConditionalOnMissingBean
	public NacosConfigProperties nacosConfigProperties() {
		return new NacosConfigProperties();
	}

	@Bean
	@ConditionalOnMissingBean
	public NacosConfigManager nacosConfigManager(
			NacosConfigProperties nacosConfigProperties) {
		return new NacosConfigManager(nacosConfigProperties);
	}

	@Bean
	public NacosPropertySourceLocator nacosPropertySourceLocator(
			NacosConfigManager nacosConfigManager) {
		return new NacosPropertySourceLocator(nacosConfigManager);
	}

}
```

可以看到，该类定义了三个 bean

- NacosConfigProperties
  ​	封装 nacos 的配置信息
- NacosConfigManager
  内部维护了ConfigService （重要接口） 和 NacosConfigProperties。
- NacosPropertySourceLocator
  该类实现了<code>PropertySourceLocator</code>，并重写<code>locate</code>方法进行配置的读取。

  > com.alibaba.cloud.nacos.client.NacosPropertySourceLocator#locate

  ```java
  	@Override
  	public PropertySource<?> locate(Environment env) {
  		nacosConfigProperties.setEnvironment(env);
  		ConfigService configService = nacosConfigManager.getConfigService();
  
  		if (null == configService) {
  			log.warn("no instance of config service found, can't load config from nacos");
  			return null;
  		}
  		long timeout = nacosConfigProperties.getTimeout();
  		nacosPropertySourceBuilder = new NacosPropertySourceBuilder(configService,
  				timeout);
  		String name = nacosConfigProperties.getName();
  
  		String dataIdPrefix = nacosConfigProperties.getPrefix();
  		if (StringUtils.isEmpty(dataIdPrefix)) {
  			dataIdPrefix = name;
  		}
  
  		if (StringUtils.isEmpty(dataIdPrefix)) {
  			dataIdPrefix = env.getProperty("spring.application.name");
  		}
  
  		CompositePropertySource composite = new CompositePropertySource(
  				NACOS_PROPERTY_SOURCE_NAME);
  
  		loadSharedConfiguration(composite);
  		loadExtConfiguration(composite);
  		loadApplicationConfiguration(composite, dataIdPrefix, nacosConfigProperties, env);
  		return composite;
  	}
  ```

  发现 nacos 启动时会加载以下三种配置文件：共享配置文件（spring.cloud.nacos.config.shared-configs）、扩展配置文件（spring.cloud.nacos.config.extension-configs）、应用程序的配置文件。

  > dataIdPrefix：取 spring.cloud.nacos.config.prefix=xx 里的值，若为空，则取 spring.cloud.nacos.config.name=xx 里配置的值，若还为空，则用应用名 spring.application.name=xx

<code>loadApplicationConfiguration</code>加载应用配置时，会加载以下三种配置，优先级依次递增

> - 不带扩展名后缀，application
> - 带扩展名后缀，application.properties
> - 带环境，带扩展名后缀，application-prod.properties

```java
	private void loadApplicationConfiguration(
			CompositePropertySource compositePropertySource, String dataIdPrefix,
			NacosConfigProperties properties, Environment environment) {
		String fileExtension = properties.getFileExtension();
		String nacosGroup = properties.getGroup();
		// load directly once by default
		loadNacosDataIfPresent(compositePropertySource, dataIdPrefix, nacosGroup,
				fileExtension, true);
		// load with suffix, which have a higher priority than the default
		loadNacosDataIfPresent(compositePropertySource,
				dataIdPrefix + DOT + fileExtension, nacosGroup, fileExtension, true);
		// Loaded with profile, which have a higher priority than the suffix
		for (String profile : environment.getActiveProfiles()) {
			String dataId = dataIdPrefix + SEP1 + profile + DOT + fileExtension;
			loadNacosDataIfPresent(compositePropertySource, dataId, nacosGroup,
					fileExtension, true);
		}

	}
```

观察发现加载的核心方法就是<code>loadNacosDataIfPresent</code>，接着会调用<code>loadNacosPropertySource</code>。

```java
	private void loadNacosDataIfPresent(final CompositePropertySource composite,
			final String dataId, final String group, String fileExtension,
			boolean isRefreshable) {
		if (null == dataId || dataId.trim().length() < 1) {
			return;
		}
		if (null == group || group.trim().length() < 1) {
			return;
		}
		NacosPropertySource propertySource = this.loadNacosPropertySource(dataId, group,
				fileExtension, isRefreshable);
		this.addFirstPropertySource(composite, propertySource, false);
	}


	private NacosPropertySource loadNacosPropertySource(final String dataId,
			final String group, String fileExtension, boolean isRefreshable) {
		if (NacosContextRefresher.getRefreshCount() != 0) {
			if (!isRefreshable) {
				return NacosPropertySourceRepository.getNacosPropertySource(dataId,
						group);
			}
		}
		return nacosPropertySourceBuilder.build(dataId, group, fileExtension,
				isRefreshable);
	}
```

<code>loadNacosPropertySource</code>方法返回一个<code>NacosPropertySource</code>对象，该对象会被缓存到<code>NacosPropertySourceRepository</code>，方便后续使用。

> com.alibaba.cloud.nacos.client.NacosPropertySourceBuilder#build

```java
	NacosPropertySource build(String dataId, String group, String fileExtension,
			boolean isRefreshable) {
		List<PropertySource<?>> propertySources = loadNacosData(dataId, group,
				fileExtension);
		NacosPropertySource nacosPropertySource = new NacosPropertySource(propertySources,
				group, dataId, new Date(), isRefreshable);
		NacosPropertySourceRepository.collectNacosPropertySource(nacosPropertySource);
		return nacosPropertySource;
	}
```

> com.alibaba.cloud.nacos.NacosPropertySourceRepository#collectNacosPropertySource

```java

public static void collectNacosPropertySource(
			NacosPropertySource nacosPropertySource) {
		NACOS_PROPERTY_SOURCE_REPOSITORY
				.putIfAbsent(getMapKey(nacosPropertySource.getDataId(),
						nacosPropertySource.getGroup()), nacosPropertySource);
	}
```

<code>NacosPropertySource</code>对象如何得到？

在上面的 build 方法中调用了 loadNacosData 方法，其通过<code>configService</code>去获取配置文件，返回一个字符串，然后通过 NacosDataParserHandler 的 parseNacosData方法解析 nacos 配置内容，支持 json 、xml、properties、yml、ymal 这几种。。

```java
	private List<PropertySource<?>> loadNacosData(String dataId, String group,
			String fileExtension) {
		String data = null;
		try {
            // 获取配置
			data = configService.getConfig(dataId, group, timeout);
			if (StringUtils.isEmpty(data)) {
				log.warn(
						"Ignore the empty nacos configuration and get it based on dataId[{}] & group[{}]",
						dataId, group);
				return Collections.emptyList();
			}
			if (log.isDebugEnabled()) {
				log.debug(String.format(
						"Loading nacos data, dataId: '%s', group: '%s', data: %s", dataId,
						group, data));
			}
            // 解析nacos配置内容
			return NacosDataParserHandler.getInstance().parseNacosData(dataId, data,
					fileExtension);
		}
		catch (NacosException e) {
			log.error("get data from Nacos error,dataId:{} ", dataId, e);
		}
		catch (Exception e) {
			log.error("parse data from Nacos error,dataId:{},data:{}", dataId, data, e);
		}
		return Collections.emptyList();
	}
```

跟进 getConfig方法会来到 <code>NacosConfigService </code>的 getConfigInner 方法

> com.alibaba.nacos.client.config.NacosConfigService#getConfigInner

```java
private String getConfigInner(String tenant, String dataId, String group, long timeoutMs) throws NacosException {
        group = null2defaultGroup(group);
        ParamUtils.checkKeyParam(dataId, group);
        ConfigResponse cr = new ConfigResponse();
        
        cr.setDataId(dataId);
        cr.setTenant(tenant);
        cr.setGroup(group);
        
        // 优先使用本地配置
        String content = LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant);
        if (content != null) {
            LOGGER.warn("[{}] [get-config] get failover ok, dataId={}, group={}, tenant={}, config={}", agent.getName(),
                    dataId, group, tenant, ContentUtils.truncateContent(content));
            cr.setContent(content);
            configFilterChainManager.doFilter(null, cr);
            content = cr.getContent();
            return content;
        }
        
        try {
            // 从远程获取
            String[] ct = worker.getServerConfig(dataId, group, tenant, timeoutMs);
            cr.setContent(ct[0]);
            
            configFilterChainManager.doFilter(null, cr);
            content = cr.getContent();
            
            return content;
        } catch (NacosException ioe) {
            if (NacosException.NO_RIGHT == ioe.getErrCode()) {
                throw ioe;
            }
            LOGGER.warn("[{}] [get-config] get from server error, dataId={}, group={}, tenant={}, msg={}",
                    agent.getName(), dataId, group, tenant, ioe.toString());
        }
        
        LOGGER.warn("[{}] [get-config] get snapshot ok, dataId={}, group={}, tenant={}, config={}", agent.getName(),
                dataId, group, tenant, ContentUtils.truncateContent(content));
    	// 从本地快照中获取
        content = LocalConfigInfoProcessor.getSnapshot(agent.getName(), dataId, group, tenant);
        cr.setContent(content);
        configFilterChainManager.doFilter(null, cr);
        content = cr.getContent();
        return content;
    }
```

步骤如下：

1. 优先使用本地配置，如果有则直接返回。
2. 通过发起 HTTP 请求（nacos 1.4.1版本）去获取，成功则保存为快照并返回。
2. 如果第二步出现异常，则从上一次的快照中获取。这也是为什么 nacos 服务端挂了后还能获取到配置的原因。

至此，在项目启动时我们就获取到了 nacos 的远程配置，并且封装成<code>NacosPropertySource</code>放到了 Spring 的环境变量里。

## 注册监听器

配置拿到了，哪如何实现配置文件的动态刷新呢？那就是监听器。

关键在于<code>NacosContextRefresher</code>类，源码的描述是：在应用启动时，NacosContextRefresher 为所有应用层的 dataId 添加 nacos 监听器，当数据发生变化时，监听器会刷新配置。

该类实现了<code>ApplicationListener</code>接口，监听的是<code>ApplicationReadyEvent</code>事件。

在重写的<code>onApplicationEvent</code>方法中，调用了<code>registerNacosListenersForApplications</code>方法：

```java
	@Override
	public void onApplicationEvent(ApplicationReadyEvent event) {
		// many Spring context
		if (this.ready.compareAndSet(false, true)) {
			this.registerNacosListenersForApplications();
		}
	}

	/**
	 * register Nacos Listeners.
	 */
	private void registerNacosListenersForApplications() {
		if (isRefreshEnabled()) {
			for (NacosPropertySource propertySource : NacosPropertySourceRepository
					.getAll()) {
				if (!propertySource.isRefreshable()) {
					continue;
				}
				String dataId = propertySource.getDataId();
				registerNacosListener(propertySource.getGroup(), dataId);
			}
		}
	}
```

如果开启了自动刷新，则从<code>NacosPropertySourceRepository</code>（在前面步骤中已把获取到的配置封装成了 NacosPropertySource 缓存到该类里）中的缓存里获取所有的<code>NacosPropertySource</code>，依次注册 Nacos 监听器。

```java
	private void registerNacosListener(final String groupKey, final String dataKey) {
		String key = NacosPropertySourceRepository.getMapKey(dataKey, groupKey);
		Listener listener = listenerMap.computeIfAbsent(key,
				lst -> new AbstractSharedListener() {
					@Override
					public void innerReceive(String dataId, String group,
							String configInfo) {
						refreshCountIncrement();
						nacosRefreshHistory.addRefreshRecord(dataId, group, configInfo);
						// todo feature: support single refresh for listening
						applicationContext.publishEvent(
								new RefreshEvent(this, null, "Refresh Nacos config"));
						if (log.isDebugEnabled()) {
							log.debug(String.format(
									"Refresh Nacos config group=%s,dataId=%s,configInfo=%s",
									group, dataId, configInfo));
						}
					}
				});
		try {
            // 在配置中添加监听器，服务端修改配置后，客户端会使用传入的监听器回调。
            // 推荐异步处理，应用程序可以在ManagerListener中实现getExecutor方法，提供执行的线程池。
            // 如果提供，使用主线程回调，可能会阻塞其他配置或被其他配置阻塞
			configService.addListener(dataKey, groupKey, listener);
		}
		catch (NacosException e) {
			log.warn(String.format(
					"register fail for nacos listener ,dataId=[%s],group=[%s]", dataKey,
					groupKey), e);
		}
	}
```

先获取一个监听器，步骤如下：

1. REFRESH_COUNT 自增加1，该值在上面的 <code>loadNacosPropertySource</code> 方法中有用到。
2. 添加一条刷新记录到 <code>com.alibaba.cloud.nacos.refresh.NacosRefreshHistory#records</code> 中，放到链表最前面，若长度大于20，则移除最后一个元素。
3. 发布一个 RefreshEvent 事件。

接下来将监听器添加到 configService 中，

> com.alibaba.nacos.client.config.impl.ClientWorker#addTenantListeners

```java
    public void addTenantListeners(String dataId, String group, List<? extends Listener> listeners)
            throws NacosException {
        group = null2defaultGroup(group);
        String tenant = agent.getTenant();
        CacheData cache = addCacheDataIfAbsent(dataId, group, tenant);
        for (Listener listener : listeners) {
            cache.addListener(listener);
        }
    }
```

在<code>addCacheDataIfAbsent</code>中，先去缓存 <code>com.alibaba.nacos.client.config.impl.ClientWorker#cacheMap</code> 中看是否存在，有则直接返回，没有则会 new 出一个 CacheData，添加到缓存后再返回。最后通过 addListener 方法完成监听器的添加。

> com.alibaba.nacos.client.config.impl.CacheData#addListener

```java
    public void addListener(Listener listener) {
        if (null == listener) {
            throw new IllegalArgumentException("listener is null");
        }
        ManagerListenerWrap wrap =
                (listener instanceof AbstractConfigChangeListener) ? new ManagerListenerWrap(listener, md5, content)
                        : new ManagerListenerWrap(listener, md5);
        
        if (listeners.addIfAbsent(wrap)) {
            LOGGER.info("[{}] [add-listener] ok, tenant={}, dataId={}, group={}, cnt={}", name, tenant, dataId, group,
                    listeners.size());
        }
    }
```

ManagerListenerWrap 对 Listener 做了层包装，内部会保存 listener、上次变更的 content 以及 md5（用来判断配置有没有变更用）。

**至此，在服务启动后向每一个需要支持热更新的配置都注册了一个监听器，用来监听远程配置的变动，以及做相应的处理。**

## 热更新

前面已经获取到了所有的 nacos 远程配置，并给需要支持热更新的配置都注册了一个监听器，那接下来就是在配置发生变化后的处理逻辑了。

回到上面的 <code>com.alibaba.cloud.nacos.client.NacosPropertySourceLocator#locate</code> 方法，该方法会获取一个 ConfigService，创建过程是由 ConfigFactory 工厂，通过反射，得到一个 NacosConfigService。该类实现了 ConfigService 接口，是一个很核心的类，配置的获取，监听器的注册都需要经此。

> com.alibaba.nacos.api.config.ConfigFactory#createConfigService(java.util.Properties)

```java
    public static ConfigService createConfigService(Properties properties) throws NacosException {
        try {
            Class<?> driverImplClass = Class.forName("com.alibaba.nacos.client.config.NacosConfigService");
            Constructor constructor = driverImplClass.getConstructor(Properties.class);
            ConfigService vendorImpl = (ConfigService) constructor.newInstance(properties);
            return vendorImpl;
        } catch (Throwable e) {
            throw new NacosException(NacosException.CLIENT_INVALID_PARAM, e);
        }
    }
```

进入 NacosConfigService 的构造函数，可以看到会创建一个 ClientWorker 对象，它是实现配置热更新的核心类。

> com.alibaba.nacos.client.config.NacosConfigService#NacosConfigService

```java
    public NacosConfigService(Properties properties) throws NacosException {
        ValidatorUtils.checkInitParam(properties);
        String encodeTmp = properties.getProperty(PropertyKeyConst.ENCODE);
        if (StringUtils.isBlank(encodeTmp)) {
            this.encode = Constants.ENCODE;
        } else {
            this.encode = encodeTmp.trim();
        }
        initNamespace(properties);
        
        this.agent = new MetricsHttpAgent(new ServerHttpAgent(properties));
        this.agent.start();
        this.worker = new ClientWorker(this.agent, this.configFilterChainManager, properties);
    }
```

ClientWorker 的构造函数里会去创建两个线程池，executorService 主要是用来处理长轮询请求的，executor 会每隔 10ms 检查一下配置信息。

> com.alibaba.nacos.client.config.impl.ClientWorker#ClientWorker

```java
public ClientWorker(final HttpAgent agent, final ConfigFilterChainManager configFilterChainManager,
            final Properties properties) {
        this.agent = agent;
        this.configFilterChainManager = configFilterChainManager;
        
        // Initialize the timeout parameter
        
        init(properties);
        
        this.executor = Executors.newScheduledThreadPool(1, new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r);
                t.setName("com.alibaba.nacos.client.Worker." + agent.getName());
                t.setDaemon(true);
                return t;
            }
        });
        
        this.executorService = Executors
                .newScheduledThreadPool(Runtime.getRuntime().availableProcessors(), new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        Thread t = new Thread(r);
                        t.setName("com.alibaba.nacos.client.Worker.longPolling." + agent.getName());
                        t.setDaemon(true);
                        return t;
                    }
                });
        
        this.executor.scheduleWithFixedDelay(new Runnable() {
            @Override
            public void run() {
                try {
                    checkConfigInfo();
                } catch (Throwable e) {
                    LOGGER.error("[" + agent.getName() + "] [sub-check] rotate check error", e);
                }
            }
        }, 1L, 10L, TimeUnit.MILLISECONDS);
    }
```

checkConfigInfo 方法中会创建一个长轮询任务放到 executorService 线程池中去处理。

```java
    public void checkConfigInfo() {
        // Dispatch taskes.
        int listenerSize = cacheMap.size();
        // Round up the longingTaskCount.
        int longingTaskCount = (int) Math.ceil(listenerSize / ParamUtil.getPerTaskConfigSize());
        if (longingTaskCount > currentLongingTaskCount) {
            for (int i = (int) currentLongingTaskCount; i < longingTaskCount; i++) {
                // The task list is no order.So it maybe has issues when changing.
                executorService.execute(new LongPollingRunnable(i));
            }
            currentLongingTaskCount = longingTaskCount;
        }
    }
```

<code>LongPollingRunnable</code> 的 run 方法执行流程如下：

1. 遍历上个章节中说到的缓存 cacheMap，如果是使用了本地缓存信息，则调用 checkListenerMd5 方法去检查读到的本地缓存文件中内容的 Md5 跟上次更新的 Md5 是不是一样，不一样则调用 safeNotifyListener 去通知监听器处理，并更新 listenerWrap 中的 lastContent、lastCallMd5。

   ```java
                   // check failover config
                   for (CacheData cacheData : cacheMap.values()) {
                       if (cacheData.getTaskId() == taskId) {
                           cacheDatas.add(cacheData);
                           try {
                               checkLocalConfig(cacheData);
                               if (cacheData.isUseLocalConfigInfo()) {
                                   cacheData.checkListenerMd5();
                               }
                           } catch (Exception e) {
                               LOGGER.error("get local config info error", e);
                           }
                       }
                   }
   ```

2. 检查服务器配置。 checkUpdateDataIds 方法中，会将所有的 CacheData 按指定格式拼接出一个字符串，构造一个长轮询请求，发给服务端，Long-Pulling-Timeout 超时时间默认为 30s，如果服务端没有配置变更，则会保持该请求直到超时，有配置变更则直接返回有变更的 dataId 列表。

   ```java
       List<String> checkUpdateDataIds(List<CacheData> cacheDatas, List<String> inInitializingCacheList) throws Exception {
           StringBuilder sb = new StringBuilder();
           for (CacheData cacheData : cacheDatas) {
               if (!cacheData.isUseLocalConfigInfo()) {
                   sb.append(cacheData.dataId).append(WORD_SEPARATOR);
                   sb.append(cacheData.group).append(WORD_SEPARATOR);
                   if (StringUtils.isBlank(cacheData.tenant)) {
                       sb.append(cacheData.getMd5()).append(LINE_SEPARATOR);
                   } else {
                       sb.append(cacheData.getMd5()).append(WORD_SEPARATOR);
                       sb.append(cacheData.getTenant()).append(LINE_SEPARATOR);
                   }
                   if (cacheData.isInitializing()) {
                       // It updates when cacheData occours in cacheMap by first time.
                       inInitializingCacheList
                               .add(GroupKey.getKeyTenant(cacheData.dataId, cacheData.group, cacheData.tenant));
                   }
               }
           }
           boolean isInitializingCacheList = !inInitializingCacheList.isEmpty();
           return checkUpdateConfigStr(sb.toString(), isInitializingCacheList);
       }
   
   
     List<String> checkUpdateConfigStr(String probeUpdateString, boolean isInitializingCacheList) throws Exception {
           
           Map<String, String> params = new HashMap<String, String>(2);
           params.put(Constants.PROBE_MODIFY_REQUEST, probeUpdateString);
           Map<String, String> headers = new HashMap<String, String>(2);
           headers.put("Long-Pulling-Timeout", "" + timeout);
           
           // told server do not hang me up if new initializing cacheData added in
           if (isInitializingCacheList) {
               headers.put("Long-Pulling-Timeout-No-Hangup", "true");
           }
           
           if (StringUtils.isBlank(probeUpdateString)) {
               return Collections.emptyList();
           }
           
           try {
               // In order to prevent the server from handling the delay of the client's long task,
               // increase the client's read timeout to avoid this problem.
               
               long readTimeoutMs = timeout + (long) Math.round(timeout >> 1);
               HttpRestResult<String> result = agent
                       .httpPost(Constants.CONFIG_CONTROLLER_PATH + "/listener", headers, params, agent.getEncode(),
                               readTimeoutMs);
               
               if (result.ok()) {
                   setHealthServer(true);
                   return parseUpdateDataIdResponse(result.getData());
               } else {
                   setHealthServer(false);
                   LOGGER.error("[{}] [check-update] get changed dataId error, code: {}", agent.getName(),
                           result.getCode());
               }
           } catch (Exception e) {
               setHealthServer(false);
               LOGGER.error("[" + agent.getName() + "] [check-update] get changed dataId exception", e);
               throw e;
           }
           return Collections.emptyList();
       }
   ```

3. 拿到第二步有变更的 GroupKey 后会调用 getServerConfig 获取最新的配置内容，然后遍历调用 checkListenerMd5 去检查最新拉取的配置内容的 Md5 值跟上次更新的 Md5 值是否一样，不一样则调用 safeNotifyListener 去通知监听器处理，并更新 listenerWrap 中的 lastContent、lastCallMd5。

   ```java
                   for (String groupKey : changedGroupKeys) {
                       String[] key = GroupKey.parseKey(groupKey);
                       String dataId = key[0];
                       String group = key[1];
                       String tenant = null;
                       if (key.length == 3) {
                           tenant = key[2];
                       }
                       try {
                           String[] ct = getServerConfig(dataId, group, tenant, 3000L);
                           CacheData cache = cacheMap.get(GroupKey.getKeyTenant(dataId, group, tenant));
                           cache.setContent(ct[0]);
                           if (null != ct[1]) {
                               cache.setType(ct[1]);
                           }
                           LOGGER.info("[{}] [data-received] dataId={}, group={}, tenant={}, md5={}, content={}, type={}",
                                   agent.getName(), dataId, group, tenant, cache.getMd5(),
                                   ContentUtils.truncateContent(ct[0]), ct[1]);
                       } catch (NacosException ioe) {
                           String message = String
                                   .format("[%s] [get-update] get changed config exception. dataId=%s, group=%s, tenant=%s",
                                           agent.getName(), dataId, group, tenant);
                           LOGGER.error(message, ioe);
                       }
                   }
                   for (CacheData cacheData : cacheDatas) {
                       if (!cacheData.isInitializing() || inInitializingCacheList
                               .contains(GroupKey.getKeyTenant(cacheData.dataId, cacheData.group, cacheData.tenant))) {
                           cacheData.checkListenerMd5();
                           cacheData.setInitializing(false);
                       }
                   }
   ```  

**至此，Nacos 的处理流程已经结束了，RefreshEvent 事件主要由 SpringCloud 相关类来处理。**
















