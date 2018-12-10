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

1.3 BrokerController中start()开启主从同步服务

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

## 二. SLAVE处理方式

2.1 由BrokerController::start()的this.messageStore.start()进去到DefaultMessageStore中开启主从同步服务

```java
public void start() throws Exception {
    ……
    //开启主从同步服务
        this.haService.start();
    ……
    }
```

2.2  HAService 主从同步需开启的服务包括以下：

```java
public void start() throws Exception {
……
        this.haClient.start();
    }
```

2.3 HAClient执行过程，只有当为SLAVE时会执行以下逻辑：

```java
@Override
        public void run() {
            log.info(this.getServiceName() + " service started");

            while (!this.isStopped()) {
                try {
                    //当master地址不空时，认为自己是SLAVE执行以下逻辑，当存在masterAdress != null && 连接Master成功
                    if (this.connectMaster()) {

                        // 若距离上次上报时间超过5S（初始），上报Master进度
                        if (this.isTimeToReportOffset()) {
                            boolean result = this.reportSlaveMaxOffset(this.currentReportedOffset);
                           ……
                        }

                        //最多阻塞1S,上报Master自己的同步进度后等待Master返回新的同步数据
                        this.selector.select(1000);
                        // 处理新的读取事件
                        boolean ok = this.processReadEvent();
                       ……
                        // 若进度有变化，上报到Master进度
                        if (!reportSlaveMaxOffsetPlus()) {
                            continue;
                        }

                        long interval = HAService.this.getDefaultMessageStore().getSystemClock().now()- this.lastWriteTimestamp;
                        // Master超过20S未返回数据，关闭连接
                        if (interval > HAService.this.getDefaultMessageStore().getMessageStoreConfig()
                            .getHaHousekeepingInterval()) {
                            log.warn("HAClient, housekeeping, found this connection[" + this.masterAddress
                                + "] expired, " + interval);
                            this.closeMaster();
                            log.warn("HAClient, master not response some time, so close connection");
                        }
                    } else {
                        this.waitForRunning(1000 * 5);
                    }
                } catch (Exception e) {
                    log.warn(this.getServiceName() + " service has exception. ", e);
                    this.waitForRunning(1000 * 5);
                }
            }
            log.info(this.getServiceName() + " service end");
        }
```

为什么需要调用`this.reportSlaveMaxOffset`两次？第二次上报是因为前面处理了读取事件，该事件是因为Master可能传了新的数据过来，处理完之后需要向Master报告同步情况。所以为什么会有第一次报告？

（1）第一次报告的执行条件是距离上次上报时间超过5s。如果SLAVE正常，两次处理Master发过来的消息时间间隔不会超过5s，如果不正常会默认等待5s，所以如果执行了第一次报告，有可能是Slave第一次启动或者Slave出现故障

（2）执行了第二次报告之后会循环，如果Slave正常，不会执行第一次报告，进入正常流程。