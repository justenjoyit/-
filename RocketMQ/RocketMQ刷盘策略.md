# RocketMQ刷盘策略

## 一. 异步刷盘

1.1 DefaultMessageStore

```java
public void start() throws Exception {
……
        //启动了commitLog
        this.commitLog.start();
……
    }
```

1.2 CommitLog的构造函数

```java
public CommitLog(final DefaultMessageStore defaultMessageStore) {
   ……
        //对刷盘策略进行判断
        if (FlushDiskType.SYNC_FLUSH == defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
            //同步刷盘
            this.flushCommitLogService = new GroupCommitService();
        } else {
            //异步刷盘 默认使用异步
            this.flushCommitLogService = new FlushRealTimeService();
        }
……
    }
```

1.3 在1.1调用`start()`后会进入`CommitLog`的`start()`方法开启线程执行刷盘任务

```java
public void start() {
        this.flushCommitLogService.start();
        if (defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
            this.commitLogService.start();
        }
    }
```

1.4 FlushRealTImeService

```java
public void run() {
            ……
                try {
                    //定时刷
                    if (flushCommitLogTimed) {
                        Thread.sleep(interval);
                    } else {
                        //实时刷
                        this.waitForRunning(interval);
                    }
					……
                    //刷盘
                    CommitLog.this.mappedFileQueue.flush(flushPhysicQueueLeastPages);
                    ……
        }
```

1.5 正式刷盘

```java
public boolean flush(final int flushLeastPages) {
        boolean result = true;
        //根据偏移量获取文件
        MappedFile mappedFile = this.findMappedFileByOffset(this.flushedWhere, this.flushedWhere == 0);
        if (mappedFile != null) {
            long tmpTimeStamp = mappedFile.getStoreTimestamp();
            //刷盘
            int offset = mappedFile.flush(flushLeastPages);
            long where = mappedFile.getFileFromOffset() + offset;
            result = where == this.flushedWhere;
            this.flushedWhere = where;
            if (0 == flushLeastPages) {
                this.storeTimestamp = tmpTimeStamp;
            }
        }
        return result;
    }

public int flush(final int flushLeastPages) {
        if (this.isAbleToFlush(flushLeastPages)) {
            if (this.hold()) {
                int value = getReadPosition();

                try {
                    //We only append data to fileChannel or mappedByteBuffer, never both.
                    if (writeBuffer != null || this.fileChannel.position() != 0) {
                        //刷盘
                        this.fileChannel.force(false);
                    } else {
                        //刷盘
                        this.mappedByteBuffer.force();
                    }
                } catch (Throwable e) {
                    log.error("Error occurred when force data to disk.", e);
                }

                this.flushedPosition.set(value);
                this.release();
            } else {
                log.warn("in flush, hold failed, flush offset = " + this.flushedPosition.get());
                this.flushedPosition.set(getReadPosition());
            }
        }
        return this.getFlushedPosition();
    }
```

从上面可见，异步刷盘开了个线程不断去进行刷盘任务。



## 二. 同步刷盘

除了本身会开启刷盘线程外，对延时消息的处理处体现出同步特性

2.1 DefaultMessageStore

```java
public void start() throws Exception {
……
        //启动了commitLog
        this.commitLog.start();
       ……
        if (this.scheduleMessageService != null && SLAVE != messageStoreConfig.getBrokerRole()) {
            //当为MASTER时
            this.scheduleMessageService.start();
        }
……
    }
```

2.2 到了CommitLog构造函数中配置的同步刷盘Service----GroupCommitService

```java
public void run() {
            CommitLog.log.info(this.getServiceName() + " service started");

            while (!this.isStopped()) {
                try {
                    this.waitForRunning(10);
                    //刷盘
                    this.doCommit();
                } catch (Exception e) {
                    CommitLog.log.warn(this.getServiceName() + " service has exception. ", e);
                }
            }

            // Under normal circumstances shutdown, wait for the arrival of the
            // request, and then flush
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                CommitLog.log.warn("GroupCommitService Exception, ", e);
            }

            synchronized (this) {
                this.swapRequests();
            }

    		//刷盘
            this.doCommit();

            CommitLog.log.info(this.getServiceName() + " service end");
        }

//刷盘
private void doCommit() {
            synchronized (this.requestsRead) {
                if (!this.requestsRead.isEmpty()) {
                    for (GroupCommitRequest req : this.requestsRead) {
                        // There may be a message in the next file, so a maximum of
                        // two times the flush
                        boolean flushOK = false;
                        for (int i = 0; i < 2 && !flushOK; i++) {
                            flushOK = CommitLog.this.mappedFileQueue.getFlushedWhere() >= req.getNextOffset();

                            if (!flushOK) {
                                //刷盘
                                CommitLog.this.mappedFileQueue.flush(0);
                            }
                        }

                        req.wakeupCustomer(flushOK);
                    }

                    long storeTimestamp = CommitLog.this.mappedFileQueue.getStoreTimestamp();
                    if (storeTimestamp > 0) {
                        CommitLog.this.defaultMessageStore.getStoreCheckpoint().setPhysicMsgTimestamp(storeTimestamp);
                    }

                    this.requestsRead.clear();
                } else {
                    // Because of individual messages is set to not sync flush, it
                    // will come to this process
                    //非同步刷盘将会来这
                    CommitLog.this.mappedFileQueue.flush(0);
                }
            }
        }
```

2.3 处理延时队列 ScheduleMessageService

```java
public void start() {
		……
                this.timer.schedule(new DeliverDelayedMessageTimerTask(level, offset), FIRST_DELAY_TIME);
          ……
    }
```

2.4 DeliverDelayedMessageTImerTask

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

2.5 DefaultMessageStore

```java
public PutMessageResult putMessage(MessageExtBrokerInner msg) {
……
        //存入commitLog
        PutMessageResult result = this.commitLog.putMessage(msg);
……
    }
```

2.6 CommitLog处理消息的时候会根据条件进行刷盘

```java
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
……
        //刷盘
        handleDiskFlush(result, putMessageResult, msg);
……
    }

public void handleDiskFlush(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
        //刷盘是根据message store的刷盘策略来决定的
        //同步刷盘
        // Synchronization flush
        if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
            final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
            //消息是否等待存儲完成
            if (messageExt.isWaitStoreMsgOK()) {
                GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
                service.putRequest(request);
                boolean flushOK = request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
                if (!flushOK) {
                    log.error("do groupcommit, wait for flush failed, topic: " + messageExt.getTopic() + " tags: " + messageExt.getTags()
                        + " client address: " + messageExt.getBornHostString());
                    putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_DISK_TIMEOUT);
                }
            } else {
                service.wakeup();
            }
        }
        //异步刷盘
        // Asynchronous flush
        else {
            if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
                flushCommitLogService.wakeup();
            } else {
                commitLogService.wakeup();
            }
        }
    }
```

从上面可以看出，如果消息可等待刷盘完成后才返回