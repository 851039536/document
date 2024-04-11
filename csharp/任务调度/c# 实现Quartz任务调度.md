使用 Quartz.NET，你可以很容易地安排任务在应用程序启动时运行，或者每天、每周、每月的特定时间运行，甚至可以基于更复杂的调度规则。

官网：[http://www.quartz-scheduler.net/](http://www.quartz-scheduler.net/)

### 实现任务类

创建一个实现了 IJob 接口的类(MailJobTest)，该接口包含一个 Execute 方法，该方法将在作业运行时调用。例如：

```csharp
using Quartz;
using System;
using System.Threading.Tasks;
using UploadLogiData.Util;

namespace UploadLogiData
{
    /// <summary>
    /// 邮件任务测试(每天三点固定时间检测Logi数据并发送内部邮件)
    /// </summary>
    //对于耗时任务，需要上一次执行完成后，才执行下次任务，覆盖之前设置的执行周期
    [DisallowConcurrentExecution]
    public class MailJobTest : IJob
    {
        public async Task Execute(IJobExecutionContext context)
        {
            try
            {
                DisplayListboxMsg("邮箱开始调度");

            } catch (Exception ex)
            {  
                UpUtil.LogWrite("ExceptionLog",$"定时器异常_{ex.StackTrace}");
            }
        }
    }
}
```

实现一个LogiDownloadJob任务类

```csharp
using MechTE_480.DateTimeCategory;
using Quartz;
using System;
using System.Threading.Tasks;
using UploadLogiData.Util;

namespace UploadLogiData.Quartzs
{
    /// <summary>
    /// 监听启动时间,执行任务下载logi数据
    /// </summary>
    //对于耗时任务，需要上一次执行完成后，才执行下次任务，覆盖之前设置的执行周期
    [DisallowConcurrentExecution]
    public class LogiDownloadJob : IJob
    {
        public async Task Execute(IJobExecutionContext context)
        {
            DisplayListboxMsg("启动时间开始调度");
        }
    }
}
```

### 启动调度

实例化并启动调度程序，并调度要执行的作业：

```csharp
IScheduler scheduler1;
/// <summary>
/// 执行任务调度
/// </summary>
/// <returns></returns>
public async Task ExLogiQuartz()
{
    DisplayListboxMsg("检测任务启动!");
    //1、创建一个调度
    var factory = new StdSchedulerFactory();
    var scheduler = await factory.GetScheduler();
    await scheduler.Start();

    //2、创建一个任务
    var job = JobBuilder.Create<LogiDownloadJob>()
        .WithIdentity("LogiJob","LogiGroup")
        .Build();
    //3、创建一个触发器
    var trigger = TriggerBuilder.Create()
        .WithIdentity("LogiTrigger","LogiTriggerGroup")
        .WithCronSchedule("0/5 * * * * ?")     //5秒执行一次
        .Build();
    await scheduler.ScheduleJob(job,trigger);

    // 第二个任务
    var job2 = JobBuilder.Create<MailJobTest>()
       .WithIdentity("MailJob","MailGroup")
       .Build();
    //3、创建一个触发器
    var trigger2 = TriggerBuilder.Create()
        .WithIdentity("MailTrigger","MailTriggerGroup")
        .WithCronSchedule("0/5 * * * * ?")     //5秒执行一次
        .Build();
    await scheduler.ScheduleJob(job2,trigger2);

    scheduler1 = scheduler; // 对象赋值
}

```

为作业指定身份：

```csharp
var job = JobBuilder.Create<SimpleJob>()
        .WithIdentity("LogiJob","LogiGroup") // // 作业名称为 "LogiJob"，组名为 "LogiGroup"  
        .Build();
```

为触发器指定身份：

```csharp
ITrigger trigger = TriggerBuilder.Create()  
    .WithIdentity("myTrigger", "myTriggerGroup") // 触发器名称为 "myTrigger"，组名为 "myTriggerGroup"  
    .StartNow()  
    .WithSimpleSchedule(x => x.WithIntervalInHours(1).RepeatForever())  
    .Build();
```

创建第二个任务:

```csharp
 // 创建并调度第二个作业  
 // 第二个任务
 var job2 = JobBuilder.Create<MailJobTest>()
    .WithIdentity("MailJob","MailGroup")
    .Build();
 //3、创建一个触发器
 var trigger2 = TriggerBuilder.Create()
     .WithIdentity("MailTrigger","MailTriggerGroup")
     .WithCronSchedule("0/5 * * * * ?")     //5秒执行一次
     .Build();
 await scheduler.ScheduleJob(job2,trigger2);
  
    // 可以继续添加更多的作业和触发器...  
```

重要性

* **唯一性**：确保每个作业和触发器在调度器中具有唯一的标识，从而避免冲突。
* **组织性**：通过组名，可以将相关的作业或触发器组织在一起，便于管理。
* **灵活性**：通过使用不同的组，可以轻松地启用、禁用或删除一组相关的作业或触发器。
* **查询和更新**：有了明确的身份标识，可以更容易地查询作业或触发器的状态，或者对其进行更新。

### 取消任务调度

```csharp
// 关闭
scheduler1.Shutdown().Wait();
```

```csharp
 //方法外部访问调度器实例
IScheduler scheduler1;
 public async void ExQuartz()
 {
     DisplayListboxMsg("检测任务启动!");
     //1、创建一个调度
     var factory = new StdSchedulerFactory();
     var scheduler = await factory.GetScheduler();
     await scheduler.Start();
     //2、创建一个任务
     var job = JobBuilder.Create<SimpleJob>()
         .WithIdentity("job1","group1")
         .Build();
     //3、创建一个触发器
     var trigger = TriggerBuilder.Create()
         .WithIdentity("trigger1","group1")
         .WithCronSchedule("0/2 * * * * ?")     //5秒执行一次
         .Build();
     await scheduler.ScheduleJob(job,trigger);

     scheduler1 = scheduler; // 对象赋值
  
 }
```

### 把ExQuartz封装到类里面

`ExLogiQuartz`方法示例，同时考虑了`scheduler`的存储和访问：

QuartzScheduler类中

```csharp
using Quartz;
using Quartz.Impl;
using System.Threading.Tasks;

namespace UploadLogiData.Quartzs
{

    /// <summary>
    /// 封装了ExLogiQuartz方法和对scheduler的访问
    /// </summary>
    public class QuartzScheduler
    {
        /// <summary>
        /// 方法外部访问调度器实例
        /// </summary>
        IScheduler logiScheduler;

        //先静态定义一下主窗体
        public static Form1 form;

        public async Task ExLogiQuartz()
        {
            form.DisplayListboxMsg("检测任务启动!");
            // 创建一个调度器实例  
            var factory = new StdSchedulerFactory();
            logiScheduler = await factory.GetScheduler();
            await logiScheduler.Start();

            // 创建一个任务：LogiDownloadJob，每5秒执行一次  
            var job = JobBuilder.Create<LogiDownloadJob>()
                .WithIdentity("LogiJob","LogiGroup")
                .Build();
            var trigger = TriggerBuilder.Create()
                .WithIdentity("LogiTrigger","LogiTriggerGroup")
                .WithCronSchedule("0/5 * * * * ?") // 5秒执行一次  
                .Build();
            await logiScheduler.ScheduleJob(job,trigger);

            // 创建第二个任务：MailJobTest，每天下午3点10分执行一次  
            var job2 = JobBuilder.Create<MailJobTest>()
                .WithIdentity("MailJob","MailGroup")
                .Build();
            var trigger2 = TriggerBuilder.Create()
                .WithIdentity("MailTrigger","MailTriggerGroup")
                .WithCronSchedule("0 10 15 * * ?") // 每天下午3点10分执行一次  
                .Build();
            await logiScheduler.ScheduleJob(job2,trigger2);

            // 如果需要在其他地方访问scheduler，可以通过类属性或方法获取  
        }
        public IScheduler Scheduler => logiScheduler;

    }
}
```

调用示例

```csharp
// 使用示例  
public class Program  
{  
    public static async Task Main(string[] args)  
    {  
        var quartzScheduler = new QuartzScheduler();  
        await quartzScheduler.ExLogiQuartz();  
        
        // 如果需要在其他地方访问scheduler，可以通过以下方式获取  
        var scheduler = quartzScheduler.Scheduler;  
      
        // 如调用关闭
        quartzScheduler.Scheduler.Shutdown().Wait();
        // 其他逻辑...  
    }  
}
```