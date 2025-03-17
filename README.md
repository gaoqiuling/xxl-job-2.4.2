# xxl-job-2.4.2
源码来自xxl-job-2.4.2版本，拆成了2套工程，可分别调试运行

## xxl-job-admin
#### 注册bean到ioc容器中，XxlJobAdminConfig，初始化XxlJobScheduler,启动线程池
+ 1.触发线程池，JobTriggerPoolHelper，重要功能是XxlJobTrigger.trigger执行触发动作（调用执行器的请求url:beat,idleBeat,run,kill,log)）
+ 2.注册线程池，JobRegistryHelper，更新注册表信息（每30s调用一次）
+ 3.失败线程池，JobFailMonitorHelper
+ 4.任务结果丢失处理线程池，JobCompleteHelper
+ 5.日志线程池，JobLogReportHelper
+ 6.周期任务线程池，JobScheduleHelper（每5s调用一次），调用触发方法

#### 其他：
- 使用方法：在执行器管理，添加一个注册方式为手动录入的的AppName(自动的局域网ip不对,无法正常访问)
- 执行器端调用的该admin的url对应的controller：JobApiController
- AdminBizClient：调用执行器接口的入口

#### 调用JobTriggerPoolHelper.trigger的场景有：
+ 1.任务完成
+ 2.任务失败
+ 3.周期调度任务
+ 4.页面点击按钮执行一次

## xxl-job-executor
#### 注入bean-XxlJobSpringExecutor到容器中,实现如下方法
+ 1.获取所有的bean，找出所有在方法上打上@XxlJob的bean,把这些方法放到静态map中jobHandlerRepository（查看源码：initJobHandlerMethodRepository）
+ 2.实例化GlueFactory
+ 3.如果配置了多个admin地址，则new多个AdminBizClient客户端
+ 4.启动一个callback-TriggerCallbackThread线程，告诉admin这个执行器状态正常（在JobThread执行完方法后，会push一条数据到队列，callback队列收到消息后发送给admin）
+ 5.启动一个内嵌服务EmbedServer，接收来自admin端的请求(beat,idleBeat,run,kill,log)，使用netty.启动执行器注册线程ExecutorRegistryThread。发送一个注册请求到admin。admin接收到后如果有需要立即执行的执行器，返回一个run命令。执行器接收到run命令后，找到对应的jobHandler,new一个JobThread来执行具体业务方法(具体功能见下方)。

#### JobThread具体功能：
+ 1.JobThread在new的时候会注入handler（具体的执行器）和admin的jobId
+ 2.有个线程专门从LinkedBlockingQueue取需要执行的数据
+ 3.具体handler类执行方法
+ 4.往callback对列里面push一条数据

#### 其他：
- ExecutorBizClient：调用admin接口的入口