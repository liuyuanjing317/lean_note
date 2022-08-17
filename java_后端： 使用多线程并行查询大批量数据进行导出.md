# java_使用多线程并行查询大批量数据进行导出

应用场景：需要导出大批量数据到excel 表，数据本身的业务处理比较长，所以采用多线程来做

步骤：

1.创建线程池配置；

2.实现callable 接口，因为需要执行数据进行返回，

3，将线程放入队列里执行，并存储返回的数据，

4.获取返回数据，移除线程任务

代码如下：

##### 1.====================================================================

```
@Configuration
public class ThreadPoolConfiguration {
    @Bean
    public ThreadPoolTaskExecutor commonThreadPoolTaskExecutor() {
        ThreadPoolTaskExecutor pool = new org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor();
        int processNum = Runtime.getRuntime().availableProcessors(); // 返回可用处理器的Java虚拟机的数量
        int corePoolSize = (int) (processNum / (1 - 0.2));
        int maxPoolSize = (int) (processNum / (1 - 0.5));
        pool.setCorePoolSize(corePoolSize); // 核心池大小
        pool.setMaxPoolSize(maxPoolSize); // 最大线程数
        pool.setQueueCapacity(maxPoolSize * 1000); // 队列程度
        pool.setThreadPriority(Thread.MAX_PRIORITY);
        pool.setDaemon(false);
        pool.setKeepAliveSeconds(300);// 线程空闲时间
        return pool;
    }

}

2.====================================================================
public class OrderExcelHandller  implements Callable<List<OrderExcel>> {
    private OrderAppService appService;
    private CmsOrderExcelMapper orderExcelMapper;

    private List<OrderExcel> list=new CopyOnWriteArrayList();

    public OrderExcelHandller() {

    }

    public OrderExcelHandller(List<OrderExcel> list) {
        this.list=list;
    }

    @Override
    public List<OrderExcel> call() throws Exception {
        this.appService= SpringUtil.getBean(OrderAppService.class);
        this.orderExcelMapper= SpringUtil.getBean(CmsOrderExcelMapper.class);
        业务
    }

}
3.4.====================================================================
@Autowired
private ThreadPoolTaskExecutor threadPoolTaskExecutor; // 注入线程池
public List<OrderExcel> selectOrderToExcel(Long regionId) throws InterruptedException, ExecutionException {
    // 获取队列
   ConcurrentLinkedDeque concurrentLinkedDeque = new ConcurrentLinkedDeque();
    List<OrderExcel> res=orderExcelMapper.selectList(regionId);
    log.info("【selectOrderToExcel】res.size:{}",res.size());
    long l = System.currentTimeMillis();
    int size=100;
    int clycle=res.size()/size;

    int yushu=res.size()%size;
    int offset=size;
    int start=0;int end=size;
    // 执行器
    ExecutorCompletionService<List<OrderExcel>> completionService = new ExecutorCompletionService<List<OrderExcel>>(
            threadPoolTaskExecutor);
    for(int i=0;i<clycle;i++){
        List<OrderExcel> subIds = new ArrayList(res.subList(start,end));
        OrderExcelHandller orderExcelHandller = new OrderExcelHandller(subIds);
        // 提交任务
        Future<List<OrderExcel>> submit = completionService.submit(orderExcelHandller);
        concurrentLinkedDeque.addFirst(submit);
        start=end;
        end=start+offset;
    }
    if(yushu >0 && clycle>0){
        List<OrderExcel> subIds = new ArrayList(res.subList(size*clycle,res.size()));
        OrderExcelHandller orderExcelHandller = new OrderExcelHandller(subIds);
        Future<List<OrderExcel>> submit = completionService.submit(orderExcelHandller);
        concurrentLinkedDeque.addFirst(submit);
    }
    else if(yushu >0 && clycle<=0){
        List<OrderExcel> subIds =res;
        OrderExcelHandller orderExcelHandller = new OrderExcelHandller(subIds);
        Future<List<OrderExcel>> submit = completionService.submit(orderExcelHandller);
        concurrentLinkedDeque.addFirst(submit);
    }
    List<OrderExcel> list = new CopyOnWriteArrayList<>();
    while (concurrentLinkedDeque.size()>0){
        // 获取返回信息
        Future<List<OrderExcel>> first = (Future<List<OrderExcel>>) concurrentLinkedDeque.getFirst();
        if(first.isDone()){
            list.addAll(first.get());
            // 队里移除
            concurrentLinkedDeque.removeFirst();
        }
    }
    log.info("【selectOrderToExcel】list.size:{}",list.size());
    long l1 = System.currentTimeMillis();
    log.info("【selectOrderToExcel】使用时长: 用前：{} 用后：{}，耗时：{}",l,l1,l1-l);
    return list;
}
4.====================================================================
5.====================================================================
由于项目是前后端分离，输出数据不做赘述
```