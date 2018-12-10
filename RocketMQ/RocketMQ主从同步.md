# RocketMQ主从同步

## 一. 启动

1.1 BrokerStartup中开始

```java
//执行初始化操作
boolean initResult = controller.initialize();
```

2.2 BrokerController的initialze()初始化

```java
public boolean initialize() throws CloneNotSupportedException {
……
        if (result) {
            try {
                //初始化message store
                this.messageStore =
                    new DefaultMessageStore(this.messageStoreConfig, this.brokerStatsManager, this.messageArrivingListener,
                        this.brokerConfig);
                this.brokerStats = new BrokerStats((DefaultMessageStore) this.messageStore);
                MessageStorePluginContext context = new MessageStorePluginContext(messageStoreConfig, brokerStatsManager, messageArrivingListener, brokerConfig);
                this.messageStore = MessageStoreFactory.build(context, this.messageStore);
                this.messageStore.getDispatcherList().addFirst(new CommitLogDispatcherCalcBitMap(this.brokerConfig, this.consumerFilterManager));
            } catch (IOException e) {
                result = false;
                log.error("Failed to initialize", e);
            }
        }
……
        if (result) {
           ……
            //如果是SLAVE会定时同步任务
            if (BrokerRole.SLAVE == this.messageStoreConfig.getBrokerRole()) {
                // 如果当前Broker配置中指定了haMasterAddress,则赋值 HAClient 的 masterAddress
                if (this.messageStoreConfig.getHaMasterAddress() != null && this.messageStoreConfig.getHaMasterAddress().length() >= 6) {
                    // 将haMasterAddress的值设置到 HAService 的 HAClient 的masterAddress中
        this.messageStore.updateHaMasterAddress(this.messageStoreConfig.getHaMasterAddress());
                    this.updateMasterHAServerAddrPeriodically = false;
                } else {
                    //如果配置中未指定Master的IP,则定期从Namesrv处更新获取
                    this.updateMasterHAServerAddrPeriodically = true;
                }
                //定时同步任务
                this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

                    @Override
                    public void run() {
                        try {
                            //TODO
                            BrokerController.this.slaveSynchronize.syncAll();
                        } catch (Throwable e) {
                            log.error("ScheduledTask syncAll slave exception", e);
                        }
                    }
                }, 1000 * 10, 1000 * 60, TimeUnit.MILLISECONDS);
         ……
        return result;
    }
```

Broker进行初始化时，会配置定时同步任务，如果有指定，就使用指定的Master Address，如果没有就去Name Server处获取

1.3 BrokerController中start()开启

```java
public void start() throws Exception {
        if (this.messageStore != null) {
            //启动了HAService，用于主从复制
            this.messageStore.start();
        }
……
        //定时向nameserver上报信息
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    BrokerController.this.registerBrokerAll(true, false);
                } catch (Throwable e) {
                    log.error("registerBrokerAll Exception", e);
                }
            }
        }, 1000 * 10, 1000 * 30, TimeUnit.MILLISECONDS);
……
    }
```

