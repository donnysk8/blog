# 动态定时任务

在项目里遇到一个业务场景，有一个报表系统，数据需要从业务数据查询出来，由于报表较多，每个报表的查询又很复杂，所以确定的方案是，先从业务表抽取一批中间表数据出来，报表再去中间表查询数据，由于已经经过一次加工，报表查询相对来说变得简单了一些。原本提出数据抽取这里应使用ETL，但是客户希望自己写代码实现，所以最终是通过oracle的merge into进行增量抽取，主要条件是时间段，遇到重复数据则覆盖，不重复则插入，由于主体条件是时间段，就涉及到定时执行的问题。最终流程：

```mermaid
graph LR;
定时服务-->|RestTemplate|RESTful;
RESTful-->|调用|Service;
Service-->|调用|Dao层的Merge;
```

随着报表的不断增多，定时任务也越来越多，于是我想到把定时任务的配置信息都放在数据库里，通过不重启定时任务服务实现动态增删改定时任务。思路如下：

1. 实际要执行的任务提供RESTful接口
2. 数据库，主表存储每个任务的url、cron表达式
3. 数据库，子表存储每个定时任务的传递参数（给RESTful接口传递的）
4. 定时服务里先启动一个定时任务（60分钟一次），读取数据库里的所有定时任务信息，当然60分钟可能不够及时，5分钟又太频繁，所以可以开一个接口，主动调用刷新任务
5. 根据数据库信息动态创建、修改、删除定时任务

关键代码

```java
package com.framework.config;

import com.framework.dao.CkLxSjbMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.scheduling.Trigger;
import org.springframework.scheduling.TriggerContext;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler;
import org.springframework.scheduling.support.CronTrigger;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

import java.text.SimpleDateFormat;
import java.util.*;
import java.util.concurrent.ScheduledFuture;

/**
 * @文件名 SchedulePool.java
 * @描述 定时任务池
 * @author cuijiyong
 * @创建日期 2018/12/3
 */
@Component
public class SchedulePool {

    private static final Logger logger = LoggerFactory.getLogger(SchedulePool.class);

    /**
     * 数据抽取工程的远程调用地址
     */
    @Value(value = "${api.gateway.datastatistics}")
    private String url;

    /**
     * 定时任务池中的线程最大数
     */
    @Value(value = "${schedule.pool.max.size}")
    private int schedulePoolSize;

    @Autowired
    private ThreadPoolTaskScheduler threadPoolTaskScheduler;

    @Autowired
    private CkLxSjbMapper ckLxSjbMapper;

    /**
     * 每隔多少秒重新读取数据库中的定时任务配置
     */
    @Value(value = "${schedule.search.job.delay}")
    private int scheduleSearchDelay;

    @Autowired
    private RestTemplate restTemplate;

    @Bean
    public ThreadPoolTaskScheduler threadPoolTaskScheduler(){
        ThreadPoolTaskScheduler threadPoolTaskScheduler = new ThreadPoolTaskScheduler();
        threadPoolTaskScheduler.setPoolSize(schedulePoolSize);
        threadPoolTaskScheduler.setThreadNamePrefix("task pool-");
        return threadPoolTaskScheduler;
    }

    private Map<String, Map<String, Object>> jobs = new HashMap<>();

    public String getCron(String key){
        return (String) jobs.get(key).get("cron");
    }

    @Scheduled(fixedDelayString = "${schedule.search.job.delay}", initialDelay = 0)
    public void updateTask(){
        logger.info("开始重新获取定时任务列表并更新...下次更新时间：{}秒后", scheduleSearchDelay / 1000);
        // 查询所有task
        List<Map<String, Object>> list = ckLxSjbMapper.selectAllWithParam();

        // 转换为一个task对应多个参数的结构
        Map<String, Map<String, Object>> taskMap = new HashMap<>();
        for (Map<String, Object> rowMap : list) {
            String key = (String) rowMap.get("id");
            Map<String, Object> task;
            if(taskMap.containsKey(key)){
                task = taskMap.get(key);
            }else{
                task = new HashMap<>();
                taskMap.put(key, task);
                task.put("params", new HashMap<String, Object>());
            }
            task.put("key", key);
            task.put("cron", rowMap.get("xgsj"));
            task.put("path", url + rowMap.get("jkdz"));
            if(rowMap.get("csm") != null){
                Map<String, Object> params = (Map<String, Object>) task.get("params");
                params.put(String.valueOf(rowMap.get("csm")),  rowMap.get("csz"));
            }
        }

        // 对比本次查询的任务与之前任务的区别
        for (String key : taskMap.keySet()) {
            Map<String, Object> task = taskMap.get(key);
            // 如果当前任务已经在任务池中
            if(jobs.containsKey(key)){
                // 如果当前查询到的任务参数与之前不一致
                Map<String, Object> source = taskMap.get(key);
                Map<String, Object> target = jobs.get(key);
                if(!source.get("cron").equals(target.get("cron")) || !source.get("path").equals(target.get("path"))
                        || !source.get("params").equals(target.get("params"))){
                    // 更新task
                    runTask(task);
                }
            }else{
                // 添加task
                runTask(taskMap.get(key));
            }
        }
        // 删除本次查询不存在的任务
        Set<String> removedKeys = new HashSet<>();
        removedKeys.addAll(jobs.keySet());
        removedKeys.removeAll(taskMap.keySet());
        for (String removedKey : removedKeys) {
            stopTask(removedKey);
            jobs.remove(removedKey);
        }

    }

    public void stopTask(String key){
        Map<String, Object> task = jobs.get(key);
        if(task != null){
            ScheduledFuture scheduledFuture = (ScheduledFuture) task.get("future");
            if(scheduledFuture != null){
                scheduledFuture.cancel(true);
                jobs.remove(key);
            }
        }
    }

    public void runTask(Map<String, Object> task) {
        // String key, String cron, String path, Map<String, Object> param
        String key = (String) task.get("key");
        stopTask(key);
        String cron = (String) task.get("cron");
        String path = (String) task.get("path");
        jobs.put(key, task);
        // 先取消上次的任务
        ScheduledFuture future = threadPoolTaskScheduler.schedule(new Runnable() {

            @Override
            public void run() {
                try {
                    Map<String, Object> params = (Map<String, Object>) task.get("params");
                    if(params == null){
                        params = new HashMap<>(0);
                    }
                    Map<String, Object> result = restTemplate.postForObject(path, params, Map.class);
                    logger.info("执行任务：{}，执行结果：{}", task, result);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }, new Trigger() {
            @Override
            public Date nextExecutionTime(TriggerContext triggerContext) {
                if ("".equals(cron) || cron == null) {
                    return null;
                }
                // 定时任务触发，可修改定时任务的执行周期
                CronTrigger trigger = new CronTrigger(cron);
                Date nextExecDate = trigger.nextExecutionTime(triggerContext);
                return nextExecDate;
            }
        });
        Map<String, Object> row = new HashMap<>();
        row.put("cron", cron);
        row.put("future", future);
        jobs.put(key, row);
    }

}
```



