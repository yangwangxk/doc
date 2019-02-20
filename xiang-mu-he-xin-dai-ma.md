---
description: 对于部分核心代码的解读
---

# 项目核心代码

Jelly-launcher子项目下，继承Netty通道处理器，具体实现代码如下：

```text
/**
 * 处理字节流
 * @param ctx
 * @param msg
 * @throws Exception
 */
public void handlerBufWebSocketFrame(ChannelHandlerContext ctx,BinaryWebSocketFrame msg) throws Exception{
    Channel incoming = ctx.channel();
    RequestVo request = new RequestVo();
        // 指定Channel
    request.setChannel(incoming);
    ByteBuf buf = msg.content();  
    byte[] byteArray = new byte[buf.capacity()];  
     buf.readBytes(byteArray);  
     request.setBt(byteArray);
     if(byteArray != null && byteArray.length > 0){
         try {
             // 添加到任务队列
              boolean offer = queue.offer(request);
              //System.out.println("TaskQueue添加任务: taskQueue=" + queue.size());
              if (!offer) {
                  // 服务器繁忙
                  System.out.println("服务器繁忙，拒绝服务");
                  // 繁忙响应
                  incoming.writeAndFlush(new TextWebSocketFrame("服务器繁忙，拒绝服务"));
              }
         } catch (Exception e) {
            System.out.println("数据格式错误");
         }
      }else{
         // 服务器繁忙
            System.out.println("参数为空！");
            // 繁忙响应
            incoming.writeAndFlush(new TextWebSocketFrame("参数为空！"));
      }
}
```

该类继承 SimpleChannelInboundHandler通道处理器，重写了ChannelRead0方法，监听Netty框架的通信数据，该方法是处理字节流数据。

首先通过BinaryWebSocketFrame拿到字节数组，封装一个类似与request请求对象，把Channel对象和解析后的字节数组放到request对象中，然后把request对象添加到queue消息对象中



Jelly-service子项目下，消息队列处理类，具体实现如下：

```text
public class Service {
    private static final Logger logger = Logger.getLogger(Service.class);

    public static AtomicBoolean shutdown = new AtomicBoolean(false);

    // 任务队列
    private BlockingQueue<RequestVo> taskQueue;
    // 阻塞式地从taskQueue取MessageHolder
    private ExecutorService takeExecutor;
    // 执行业务的线程池
    private ExecutorService taskExecutor;

    public void initAndStart() {
        init();
        start();
    }

    private void init() {
        takeExecutor = Executors.newSingleThreadExecutor();
        taskExecutor = Executors.newFixedThreadPool(10);
        taskQueue = TaskQueue.getQueue();
        logger.info("初始化服务完成");
    }

    /**
     * 执行任务队列
     */
    private void start() {
        takeExecutor.execute(new Runnable() {
            @Override
            public void run() {
                while (!shutdown.get()) {
                    try {
                       RequestVo bt = taskQueue.take();
                        //logger.info("TaskQueue取出任务: taskQueue=" + taskQueue.size());
                        startTask(bt);
                    } catch (InterruptedException e) {
                        logger.warn("receiveQueue take", e);
                    }
                }
            }

            private void startTask(final RequestVo bt) {
                taskExecutor.execute(new Runnable() {
                    @Override
                    public void run() {
                        Dispatcher.dispatch(bt);
                    }
                });
            }
        });
        logger.info("启动服务完成");
    }
}
```

该类在项目启动的时候就会加载运行，不间断的监听消息队列



Jelly-service子项目下，定时任务管理类

```text
**
 * 定时控制器
 * @author xiang.xin
 *
 */
public class ScheduledService {
    
    @SuppressWarnings("unchecked")
    public static ConcurrentHashMap<String, Future> futureMap = new ConcurrentHashMap<String, Future>(); //线程管理
     public static ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(200);   //线程池

    /**
    * 添加定时器
    * @param task
     */
     @SuppressWarnings("unchecked")
   public static void addTask(GameTask task){
        Future future = scheduler.scheduleAtFixedRate(task, 0, 8, TimeUnit.SECONDS);  //8s循环执行定时任务
        futureMap.put(task.getTaskId(), future);
     }

    /**
    * 移除定时器
    * @param taskId
     */
    @SuppressWarnings("unchecked")
   public static void removeTask(String taskId){
        Future future = futureMap.get(taskId);
        future.cancel(true);  //定时任务停止
        futureMap.remove(taskId);
    }
    
}
```

该类处理定时任务启动，回收定时任务，控制定时任务线程



Jelly-service子项目下，通用返回方法

```text
/**
 * 带时间戳返回
 *
 * @param channel
 * @param response
 * @param result
 * @param time
 */
public static void response(Channel channel, WebSocketHolder response, String result, String time) {
   try {
      BaseResponse john = BaseResponse.newBuilder()
            .setTimestamp(Long.valueOf(time))
            .setAction(response.getAction())
            .setType(response.getType())
            .setFrom(response.getFrom())
            .setTo(response.getTo())
            .setData(result)
            .build();

      byte[] bytes = john.toByteArray();
      ByteBuf wrappedBuffer = Unpooled.wrappedBuffer(bytes);
      BinaryWebSocketFrame binary = new BinaryWebSocketFrame(wrappedBuffer);
      channel.writeAndFlush(binary);
   } catch (Exception e) {
      System.out.println(response.getAction() + "类型响应异常！！！！！");
   }
}
```

调用该方法可以直接返回数据给对应的连接客户端

