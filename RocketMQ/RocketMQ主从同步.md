# RocketMQ主从同步

> 参考：
>
> 1. https://www.2cto.com/kf/201803/730566.html
>
> 2. http://www.cnblogs.com/sunshine-2015/p/8998549.html

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

## 三. Master处理逻辑

### 3.1 ASYNC_MASTER处理逻辑

1. salve连接到master，向master上报slave当前的offset
2. master收到后确认给slave发送数据的开始位置
3. master查询开始位置对应的MappedFIle
4. master将查找到的数据发送给slave
5. slave收到数据后保存到自己的CommitLog

3.1.1 DefaultMessageStore

```java
public void start() throws Exception {
……
        if (this.scheduleMessageService != null && SLAVE != messageStoreConfig.getBrokerRole()) {
            //当为MASTER时执行，处理延时消息
            this.scheduleMessageService.start();
        }
……
    //开启主从同步服务
        this.haService.start();
……
    }
```

3.1.2 HAService

```java
public void start() throws Exception {
        //监听Slave连接请求
        this.acceptSocketService.beginAccept();
        this.acceptSocketService.start();
    
        this.groupTransferService.start();
        //HAClient里面的逻辑对Master没用
        this.haClient.start();
    }
```

3.1.3 AcceptSocketService

```java
@Override
public void run() {
      ……
		//包装成HAConnection
		HAConnection conn = new HAConnection(HAService.this, sc);
		 conn.start();
……
}
```

3.1.4 HAConnection

```java
public void start() {
        //Master负责从socketChannel读取Slave传递的ACK进度
        this.readSocketService.start();
        //Master负责向socketChannel写入commitlog的消息数据
        this.writeSocketService.start();
    }
```

3.1.5 ReadSocketService

```java
 @Override
 public void run() {
            ……
            while (!this.isStopped()) {
                try {
                    this.selector.select(1000);
                    //处理读事件
                    boolean ok = this.processReadEvent();
                   ……
            }
……
}
     
 //处理读事件
private boolean processReadEvent() {
        ……
        //得到Slave Offset信息后通知需要传数据
        HAConnection.this.haService.notifyTransferSome(HAConnection.this.slaveAckOffset);
        ……
}
```

3.1.6 HAService

```java
public void notifyTransferSome(final long offset) {
        for (long value = this.push2SlaveMaxOffset.get(); offset > value; ) {
            boolean ok = this.push2SlaveMaxOffset.compareAndSet(value, offset);
            if (ok) {
                //到了GroupTransferService
                this.groupTransferService.notifyTransferSome();
                break;
            } else {
                value = this.push2SlaveMaxOffset.get();
            }
        }
    }
```

3.1.7 GroupTransferService

```java
public void run() {
         ……
             //等10毫秒
                    this.waitForRunning(10);
                    this.doWaitTransfer();
           ……
        }

//所有Slave中已成功复制的最大偏移量是否大于等于消息生产者发送消息后消息服务端返回下一条消息的起始偏移量
//如果是则表示主从同步复制已经完成，唤醒Consumer
//否则等待1s,再次判断，每一个任务在一批任务中循环判断5次
private void doWaitTransfer() {
            synchronized (this.requestsRead) {
                if (!this.requestsRead.isEmpty()) {
                    for (CommitLog.GroupCommitRequest req : this.requestsRead) {
                        //当transferOK为true时，代表主从同步复制已成功
                        boolean transferOK = HAService.this.push2SlaveMaxOffset.get() >= req.getNextOffset();
                        for (int i = 0; !transferOK && i < 5; i++) {
                            this.notifyTransferObject.waitForRunning(1000);
                            transferOK = HAService.this.push2SlaveMaxOffset.get() >= req.getNextOffset();
                        }

                        if (!transferOK) {
                            log.warn("transfer messsage to slave timeout, " + req.getNextOffset());
                        }

                        req.wakeupCustomer(transferOK);
                    }

                    this.requestsRead.clear();
                }
            }
        }
```

3.1.8 WriteSocketService

```java
@Override
        public void run() {
            HAConnection.log.info(this.getServiceName() + " service started");

            while (!this.isStopped()) {
                try {
                   ……
                    if (-1 == this.nextTransferFromWhere) {
                        //SLAVE第一次请求
                        if (0 == HAConnection.this.slaveRequestOffset) {
                            long masterOffset = HAConnection.this.haService.getDefaultMessageStore().getCommitLog().getMaxOffset();
                            //从lastMappedFile的0位开始,也就是会忽略之前的MappedFile,只从最后一个文件开始同步
                            masterOffset =
                                masterOffset
                                    - (masterOffset % HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig()
                                    .getMapedFileSizeCommitLog());

                            if (masterOffset < 0) {
                                masterOffset = 0;
                            }

                            this.nextTransferFromWhere = masterOffset;
                        } else {
                            this.nextTransferFromWhere = HAConnection.this.slaveRequestOffset;
                        }

                      ……
                    }

                    //上一次写完了
                    if (this.lastWriteOver) {
							……
                            this.lastWriteOver = this.transferData();
                           ……
                    } else {
                        //上一次没写完，继续上次
                        this.lastWriteOver = this.transferData();
                       ……
                    }

                    // 传输消息到从服务器
                    SelectMappedBufferResult selectResult = HAConnection.this.haService.getDefaultMessageStore().getCommitLogData(this.nextTransferFromWhere);
                    if (selectResult != null) {
                        ……
                        this.lastWriteOver = this.transferData();
                    } else {
                        HAConnection.this.haService.getWaitNotifyObject().allWaitForRunning(100);
                    }
          ……
        }
```

### 3.2 SYNC_MASTER处理逻辑

1. master将需要传输到slave的数据构造为GroupCommitRequest交给GroupTransferService
2. 唤醒传输数据的线程（如果没有更多数据需要传输的的时候HAClient.run会等待新的消息）
3. 等待当前的传输请求完成

SYNC_MASTER和ASYNC_MASTER传输数据到salve的过程是一致的，只是时机上不一样。SYNC_MASTER接收到producer发送来的消息时候，会同步等待消息也传输到salve。

以下为和ASYNC_MASTER不同的地方

3.2.1 DefaultMessageStore

```java
public void start() throws Exception {
……
        if (this.scheduleMessageService != null && SLAVE != messageStoreConfig.getBrokerRole()) {
            //当为MASTER时
            this.scheduleMessageService.start();
        }
……
    //和ASYNC_MASTER一样
    }
```

3.2.2 ScheduleMessageService

```java
public void start() {
……
    //延时消息
                this.timer.schedule(new DeliverDelayedMessageTimerTask(level, offset), FIRST_DELAY_TIME);
……
    }
```

3.2.3 DeliverDelayedMessageTimerTask

```java
@Override
        public void run() {
         ……
                this.executeOnTimeup();
            ……
        }

public void executeOnTimeup() {
          ……
		 PutMessageResult putMessageResult =ScheduleMessageService.this.defaultMessageStore.putMessage(msgInner);
……
        }
```

3.2.4 DefaultMessageStore

```java
public PutMessageResult putMessage(MessageExtBrokerInner msg) {
  ……
        //存入commitLog
        PutMessageResult result = this.commitLog.putMessage(msg);
……
    }
```

3.2.5 CommitLog

```java
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
   ……
        //主从同步
        handleHA(result, putMessageResult, msg);
……
 }
        
public void handleHA(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
    //如果是SYNC_MASTER才会执行如下操作
        if (BrokerRole.SYNC_MASTER == this.defaultMessageStore.getMessageStoreConfig().getBrokerRole()) {
            ……
           //唤醒所有等待线程
                    service.putRequest(request);
                    service.getWaitNotifyObject().wakeupAll();
                   ……
        }
}
```

以上可以看出，当SYNC_MASTER进行刷盘后会等待SLAVE完成，才会进行后续操作