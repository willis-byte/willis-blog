下面为你介绍基于 Spring Boot 集成 Quartz 和 Redisson 实现分布式调度的详细步骤。

1. 添加依赖
   在 G:/lycoris btyes/lycoris/pom.xml 文件里添加 Quartz 和 Redisson 相关依赖：


G:\lycoris btyes\lycoris\pom.xml
Apply
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
<!-- ... 已有代码 ... -->
<dependencies>
<!-- Spring Boot Quartz 依赖 -->
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
<!-- Redisson 依赖 -->
<dependency>
<groupId>org.redisson</groupId>
<artifactId>redisson-spring-boot-starter</artifactId>
<version>3.18.0</version>
</dependency>
</dependencies>
<!-- ... 已有代码 ... -->
</project>
2. 配置 Redisson
   在 application.properties 或者 application.yml 中配置 Redisson：


yaml
Apply
spring:
redisson:
file: classpath:redisson-config.yml
在 src/main/resources 目录下创建 redisson-config.yml 文件：


yaml
Apply
singleServerConfig:
address: "redis://127.0.0.1:6379"
database: 0
threads: 0
nettyThreads: 0
codec:
class: "org.redisson.codec.JsonJacksonCodec"
transportMode: "NIO"
3. 创建 Quartz Job
   创建一个实现 Job 接口的 Quartz Job 类：


java
Apply
import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Component
public class DistributedJob implements Job {

    @Autowired
    private RedissonClient redissonClient;

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        // 获取分布式锁
        RLock lock = redissonClient.getLock("distributedJobLock");
        try {
            // 尝试获取锁，设置等待时间和释放时间
            if (lock.tryLock(0, 10, TimeUnit.SECONDS)) {
                // 执行调度任务
                System.out.println("Distributed job is running on " + Thread.currentThread().getName());
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            // 释放锁
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
4. 配置 Quartz Scheduler
   创建一个配置类来配置 Quartz Scheduler：


java
Apply
import org.quartz.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class QuartzConfig {

    @Bean
    public JobDetail distributedJobDetail() {
        return JobBuilder.newJob(DistributedJob.class)
                .withIdentity("distributedJob")
                .storeDurably()
                .build();
    }

    @Bean
    public Trigger distributedJobTrigger() {
        SimpleScheduleBuilder scheduleBuilder = SimpleScheduleBuilder.simpleSchedule()
                .withIntervalInSeconds(10)
                .repeatForever();

        return TriggerBuilder.newTrigger()
                .forJob(distributedJobDetail())
                .withIdentity("distributedJobTrigger")
                .withSchedule(scheduleBuilder)
                .build();
    }
}
5. 运行 Spring Boot 应用
   启动 Spring Boot 应用，Quartz 会按照配置的调度规则执行任务，借助 Redisson 实现分布式锁，确保同一时间只有一个节点执行任务。

总结
添加依赖：在 pom.xml 里添加 Quartz 和 Redisson 依赖，为项目集成相关功能。
配置 Redisson：在配置文件里设置 Redis 连接信息，用于获取分布式锁。
创建 Quartz Job：实现 Job 接口，在任务执行时获取分布式锁，保证任务的分布式执行。
配置 Quartz Scheduler：配置 JobDetail 和 Trigger，定义任务的执行规则。
运行应用：启动 Spring Boot 应用，验证分布式调度功能。
通过以上步骤，你就可以基于 Spring Boot 集成 Quartz 和 Redisson 实现分布式调度。