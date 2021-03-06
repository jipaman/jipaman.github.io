---
layout:     series
title:      Dynamic Job Scheduling with Quartz and Spring
date:       2017-09-26 20:17:17 +0000
categories: tutorial
tags:       java maven quartz rest spring-boot
section:    series
author:     juliuskrah
repo:       quartz-manager/tree/master
---
> [Quartz Scheduler][Quartz]{:target="_blank"} is a richly featured, open source job scheduling library that can
  be integrated within virtually any Java application - from the smallest stand-alone application to the largest
  e-commerce system.

# Introduction
Every developer at a certain point in his carreer is faced with the difficult task of scheduling jobs dynamically. 
In this post we are going to create a simple application for dynamically scheduling jobs using a
[REST API]({% post_url 2017-07-16-developing-restful-services-with-jax-rs-jersey %}).  
We will dynamically create jobs that sends emails to a predefined group of people on a user defined schedule using 
[Spring Boot]({% post_url 2017-04-23-crud-operations-with-spring-boot %}).

## Project Structure
At the end of this guide our folder structure will look similar to the following:

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__juliuskrah/
|  |  |  |  |  |__quartz/
|  |  |  |  |  |  |__Application.java
|  |  |  |  |  |  |__AutowiringSpringBeanJobFactory.java 
|  |  |  |  |  |  |__job/
|  |  |  |  |  |  |  |__EmailJob.java
|  |  |  |  |  |  |__model/
|  |  |  |  |  |  |  |__JobDescriptor.java
|  |  |  |  |  |  |  |__TriggerDescriptor.java
|  |  |  |  |  |  |__service/
|  |  |  |  |  |  |  |__EmailService.java
|  |  |  |  |  |  |__web/
|  |  |  |  |  |  |  |__rest/
|  |  |  |  |  |  |  |  |__EmailResource.java
|  |  |__resources/
|  |  |  |  |__application.yaml
|__pom.xml
```


# Prerequisites
To follow along this guide, you should have the following set up:

- [Java Development Kit][JDK]{:target="_blank"}  
_Optional_  
- [Maven][]{:target="_blank"}
- [cURL][]{:target="_blank"}


## Concepts
Before we dive any further, there are a few quartz concepts we need to understand:

1.    **Job** - an interface to be implemented by components that you wish to have executed by the scheduler. The 
      interface has one method `execute(...)`. This is where your scheduled task runs. Information on the 
      `JobDetail` and `Trigger` is retrieved using the `JobExecutionContext`.

      ```java
      package org.quartz;

      public interface Job {
        public void execute(JobExecutionContext context) throws JobExecutionException;
      }
      ```

2.    **JobDetail** - used to define instances of `Jobs`. This defines how a job is run. Whatever 
      data you want available to the `Job` when it is instantiated is provided through the `JobDetail`.  
      Quartz provides a Domain Specific Language (DSL) in the form of `JobBuilder` for constructing `JobDetail`
      instances.

      ```java
      // define the job and tie it to the Job implementation
      JobDetail job = newJob(EmailJob.class)
        .withIdentity("myJob", "group1") // name "myJob", group "group1"
        .build();
      ```

3.    **Trigger** - a component that defines the schedule upon which a given Job will be executed. The trigger  
      provides instruction on when the job is run.  
      Quartz provides a DSL (TriggerBuilder) for constructing `Trigger` instances.

      ```java
      // Trigger the job to run now, and then every 40 seconds
      Trigger trigger = newTrigger()
        .withIdentity("myTrigger", "group1")
        .startNow()
        .withSchedule(simpleSchedule()
          .withIntervalInSeconds(40)
          .repeatForever())            
        .build();
      ```

4.    **Scheduler** - the main API for interacting with the scheduler. A **Scheduler’s** life-cycle is bounded by 
      it’s creation, via a **SchedulerFactory** and a call to its `shutdown()` method. Once created the Scheduler 
      interface can be used to _add_, _remove_, and _list_ `Jobs` and `Triggers`, and perform other
      scheduling-related operations (such as _pausing_ a trigger). However, the Scheduler will not actually act on
      any triggers (execute jobs) until it has been started with the `start()` method.

      ```java
      SchedulerFactory schedFact = new org.quartz.impl.StdSchedulerFactory();

      Scheduler sched = schedFact.getScheduler();
      sched.start();

      // Tell quartz to schedule the job using our trigger
      sched.scheduleJob(job, trigger);
      ```

## Create and Setup Dependencies for the Sample Application
Head over to [start.spring.io][Initializr]{:target="_blank"} and build a Spring Boot template as illustrated in the
image below:

![spring.io](https://i.imgur.com/noNNTb7.png)

{:.image-caption}
*Spring Initializr*

Download the zip archive and extract the contents to a folder of your choice. Open your `pom.xml` located at
the root of the template directory and add the following dependencies:

file: {% include file-path.html file_path='pom.xml' %}

{% highlight xml %}
...
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-context-support</artifactId>
</dependency>
<dependency>
  <groupId>org.quartz-scheduler</groupId>
  <artifactId>quartz</artifactId>
  <version>2.3.0</version>
</dependency>
...
{% endhighlight %}

The above are the dependencies needed for Quartz with Spring integration.

## Scope
By the end of this post we will be able to schedule `Quartz` jobs dynamically to send emails
using a REST API.  
We will `create` jobs:

```java
// 
Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
scheduler.scheduleJob(jobDetail, trigger);
```

`retrieve` existing jobs:

```java
//
scheduler.getJobDetail(jobKey);
```

`update` existing jobs:

```java
// store, and set overwrite flag to 'true'
scheduler.addJob(jobDetail, true);
```

`delete` existing  jobs:

```java
// 
scheduler.deleteJob(jobKey);
```

`pause` jobs:

```java
// 
scheduler.pauseJob(jobKey);
```

and `resume` jobs:

```java
// 
scheduler.resumeJob(jobKey);
```

# About Jobstores
JobStore’s are responsible for keeping track of all the "work data" that you give to the scheduler: `jobs`,
`triggers`, `calendars`, etc. Selecting the appropriate JobStore for your Quartz scheduler instance is an
important step. Luckily, the choice should be a very easy one once you understand the differences between them.  
You declare which `JobStore` your scheduler should use (and it's configuration settings) in the properties file
(or object) that you provide to the `SchedulerFactory` that you use to produce your scheduler instance.

There are three types of Jobstores that are available in Quartz:

1. **RAMJobStore** - is the simplest `JobStore` to use, it is also the most performant (in terms of CPU time). 
   `RAMJobStore` gets its name in the obvious way: it keeps all of its data in RAM. This is why it’s 
   lightning-fast, and also why it’s so simple to configure. The drawback is that when your application ends (or 
   crashes) all of the scheduling information is lost - this means RAMJobStore cannot honor the setting of 
   _"non-volatility"_ on jobs and triggers. For some applications this is acceptable - or even the desired 
   behavior, but for other applications, this may be disastrous. In this part of the series we will be using
   `RAMJobStore`.
2. **JDBCJobStore** - is also aptly named - it keeps all of its data in a database via JDBC. Because of this it is 
   a bit more complicated to configure than RAMJobStore, and it also is not as fast. However, the performance  
   draw-back is not terribly bad, especially if you build the database tables with indexes on the primary keys.   
   On fairly modern set of machines with a decent LAN (between the scheduler and database) the time to retrieve 
   and update a firing trigger will typically be less than 10 milliseconds. We will talk more about `JDBCJobStore`
   in the [next post]({% post_url 2017-10-06-persisting-dynamic-jobs-with-quartz-and-spring %}).
3. **TerracottaJobStore** - provides a means for scaling and robustness without the use of a database. This means 
   your database can be kept free of load from Quartz, and can instead have all of its resources saved for the
   rest of your application.

   `TerracottaJobStore` can be ran clustered or non-clustered, and in either case provides a storage medium for
   your job data that is persistent between application restarts, because the data is stored in the Terracotta
   server. It’s performance is much better than using a database via JDBCJobStore (about an order of magnitude 
   better), but fairly slower than RAMJobStore. This is out of the scope for this series.

## Setting up the Descriptors
In order to set up the REST API for the dyanmic jobs, we will create two abstractions over `JobDetail` and
`Trigger` aptly named `JobDescriptor` and `TriggerDescriptor`:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/model/TriggerDescriptor.java' %}

{% highlight java %}
public class TriggerDescriptor {
  private String name;
  private String group;
  private LocalDateTime fireTime;
  private String cron;

  /**
   * Convenience method for building a Trigger
   */
  public Trigger buildTrigger() {
    //
  }

  /**
   * Convenience method for building a TriggerDescriptor
   */
  public static TriggerDescriptor buildDescriptor(Trigger trigger) {
    //
  }
  // Code ommitted for brevity. Click on link to view full source
}
{% endhighlight %}

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/model/JobDescriptor.java' %}

{% highlight java %}
public class JobDescriptor {
  private String name;
  private String group;
  private String subject;
  private String messageBody;
  private List<String> to;
  private List<String> cc;
  private List<String> bcc;
  private Map<String, Object> data = new LinkedHashMap<>();
  @JsonProperty("triggers")
  private List<TriggerDescriptor> triggerDescriptors = new ArrayList<>();

  /**
   * Convenience method for building triggers of Job
   */
  public Set<Trigger> buildTriggers() {
    // 
  }

  /**
   * Convenience method for building a JobDetail
   */
  public JobDetail buildJobDetail() {
    //
  }
	
  /**
   * Convenience method for building a JobDescriptor
   */
  public static JobDescriptor buildDescriptor(JobDetail jobDetail, List<? extends Trigger> triggersOfJob) {
    // 
  }

  // Code ommitted for brevity. Click on link to view full source
}
{% endhighlight %}

Next we will define our Job class:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/job/EmailJob.java' %}

{% highlight java %}
public class EmailJob implements Job {

  @Override
  public void execute(JobExecutionContext context) throws JobExecutionException {
    // JobDataMap map = context.getJobDetail().getJobDataMap();
    // JobDataMap map = context.getTrigger().getJobDataMap();
    JobDataMap map = context.getMergedJobDataMap();
    System.out.format("Map: [%s]\n", map.getWrappedMap());
  }
}
{% endhighlight %}

> The `JobDataMap` can be used to hold any amount of (_serializable_) data objects which you wish to
  have made available to the job instance when it executes. `JobDataMap` is an implementation of the Java `Map`
  interface, and has some added convenience methods for storing and retrieving data of primitive types.  
  You can retrieve the `JobDataMap` from the `JobExecutionContext` that is stored as part of the `JobDetail` or
  `Trigger`.  
  The `JobDataMap` that is found on the `JobExecutionContext` during Job execution serves as a convenience. It is
  a merge of the `JobDataMap` found on the `JobDetail` and the one found on the `Trigger`, with the values in the
  latter overriding any same-named values in the former.

## Boostrapping with Spring Boot
At the beginning of this post I stated that the life-cycle of a `Scheduler` is bounded by it’s creation, via a
**SchedulerFactory** and a call to its *`shutdown()`* method. For this post we will create a _Singleton_ instance
of a SchedulerFactory. We can achieve this by creating it as a Spring Bean:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/Application.java' %}

{% highlight java %}
...
@Bean
public SchedulerFactoryBean schedulerFactory() {
  SchedulerFactoryBean factoryBean = new SchedulerFactoryBean();
		
  return factoryBean;
}
{% endhighlight %}

The bean definition above is doing several things:-

1. **JobFactory** - The default is Spring's `AdaptableJobFactory`, which supports `java.lang.Runnable`
   objects as well as standard Quartz `org.quartz.Job` instances. Note that this default only applies to a local
   Scheduler, not to a RemoteScheduler (where setting a custom JobFactory is not supported by Quartz).
2. **ThreadPool** - Default is a Quartz `SimpleThreadPool` with a pool size of 10. This is configured through
	 the corresponding Quartz properties.
3. **SchedulerFactory** - The default used here is the `StdSchedulerFactory`, reading in the standard
	 `quartz.properties` from `quartz.jar`.
4. **JobStore** - The default used is `RAMJobStore` which does not support persistence and is not clustered.
5. **Life-Cycle** - The `SchedulerFactoryBean` implements `org.springframework.context.SmartLifecycle` and
   `org.springframework.beans.factory.DisposableBean` which means the life-cycle of the scheduler is managed
   by the Spring container. The `sheduler.start()` is called in the `start()` implementation of `SmartLifecycle`
   after initialization and the `scheduler.shutdown()` is called in the `destroy()` implementation of
   `DisposableBean` at application teardown.  
   You can override the startup behaviour by setting `setAutoStartup(..)` to `false`. With this setting you have
   to manually start the scheduler.

## Creating Some Service and Controller Classes
We will create a service class that will take care of `Creating`, `Fetching`, `Updating`, `Deleting`, `Pausing`
and `Resuming` jobs:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/service/EmailService.java' %}

{% highlight java %}
@Service
@Transactional
public class EmailService {
  private final Scheduler scheduler;

  public EmailService(Scheduler scheduler) {
    this.scheduler = scheduler;
  }
	
  public JobDescriptor createJob(String group, JobDescriptor descriptor) {
    //
  }
	
  public JobDescriptor findJob(String group, String name) {
    //
  }
	
  public void updateJob(String group, String name, JobDescriptor descriptor) {
    //
  }
	
  public void deleteJob(String group, String name) {
    //
  }
	
  public void pauseJob(String group, String name) {
    //
  }
	
  public void resumeJob(String group, String name) {
    //
  }
}
{% endhighlight %}

Now the REST endpoints:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/web/rest/EmailResource.java' %}

{% highlight java %}
@RestController
@RequestMapping("/api/v1.0")
public class EmailResource {
  private final EmailService emailService;

  public EmailResource(EmailService emailService) {
    this.emailService = emailService;
  }

  @PostMapping(path = "/groups/{group}/jobs")
  public ResponseEntity<JobDescriptor> createJob(@PathVariable String group, 
            @RequestBody JobDescriptor descriptor) {
    //
  }

  @GetMapping(path = "/groups/{group}/jobs/{name}")
  public ResponseEntity<JobDescriptor> findJob(@PathVariable String group, 
            @PathVariable String name) {
    //
  }

  @PutMapping(path = "/groups/{group}/jobs/{name}")
  public ResponseEntity<Void> updateJob(@PathVariable String group, 
            @PathVariable String name, @RequestBody JobDescriptor descriptor) {
    //
  }

  @DeleteMapping(path = "/groups/{group}/jobs/{name}")
  public ResponseEntity<Void> deleteJob(@PathVariable String group, @PathVariable String name) {
    //
  }

  @PatchMapping(path = "/groups/{group}/jobs/{name}/pause")
  public ResponseEntity<Void> pauseJob(@PathVariable String group, @PathVariable String name) {
    //
  }

  @PatchMapping(path = "/groups/{group}/jobs/{name}/resume")
  public ResponseEntity<Void> resumeJob(@PathVariable String group, @PathVariable String name) {
    //
  }
}
{% endhighlight %}

Start the server:

{% highlight posh %}
> mvnw clean spring-boot:run    # Windows
$ ./mvnw clean spring-boot:run  # Linux and Mac
{% endhighlight %}

and test the create endpoint `http://localhost:8080/api/v1.0/groups/email/jobs` via post using `curl` or `Postman`
with the following JSON payload and content type `application/json`:

{% highlight json %}
{
  "name": "manager",
  "subject": "Daily Fuel Report",
  "messageBody": "Sample fuel report",
  "to": ["juliuskrah@example.com", "juliuskrah@example.net"],
  "triggers":
    [
       {
         "name": "manager",
         "group": "email",
         "cron": "0/10 * * * * ?"
       }
    ]
}
{% endhighlight %}

This will execute every 10 seconds by printing to STDOUT.

Update at `http://localhost:8080/api/v1.0/groups/email/jobs/manager` with content type `application/json` and
payload: 

{% highlight json %}
{
  "name": "manager",
  "subject": "Daily Fuel Report",
  "messageBody": "Sample fuel report",
  "to": ["juliuskrah@example.com", "juliuskrah@example.net", "juliuskrah@example.org"],
  "cc": ["juliuskrah@example.io"]
}
{% endhighlight %}

# Setting up Email
Create a configuration file and add the `SMTP` details of your mail server which defines the behaviour of the
`JavaMailSender` bean. We will inject this bean into the job class:

file: {% include file-path.html file_path='src/main/resources/application.yaml' %}

{% highlight yaml %}
spring:
  mail:
    host: smtp.example.com
    port: 587
    username: username@example.com
    password: password
{% endhighlight %}

One thing we did not talk about, the `AdaptableJobFactory` does not do any 
[dependency]({% post_url 2016-12-10-introduction-to-spring-and-dependency-injection-jsr-330 %}) 
[injection]({% post_url 2017-07-30-dependency-injection-jsr-330-on-the-java-platform-using-hk2 %}). Each (and every)
time the scheduler executes the job, it creates a new instance of the class before calling its `execute(..)`
method. When the execution is complete, references to the job class instance are dropped, and the instance is then
garbage collected. One of the ramifications of this behavior is the fact that jobs must have a no-argument 
constructor (when using the AdaptableJobFactory implementation). Another ramification is that it does not make
sense to have state data-fields defined on the job class - as their values would not be preserved between job 
executions.

It is at this point that things get interesting. We will create our own implementation of `JobFactory` that 
supports dependency injection:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/AutowiringSpringBeanJobFactory.java' %}

{% highlight java %}
public class AutowiringSpringBeanJobFactory extends 
            SpringBeanJobFactory implements ApplicationContextAware {
  private transient AutowireCapableBeanFactory beanFactory;

  @Override
  public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
    beanFactory = applicationContext.getAutowireCapableBeanFactory();
  }

  @Override
  protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {
    final Object job = super.createJobInstance(bundle);
    beanFactory.autowireBean(job); // Dependency Injection is done here
    return job;
  }
}
{% endhighlight %}

Override the `AdaptableJobFactory` in the `SchedulerFactoryBean` and use `AutowiringSpringBeanJobFactory` instead:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/Application.java' %}

{% highlight java %}
...
@Bean
public SchedulerFactoryBean schedulerFactory(ApplicationContext applicationContext) {
  SchedulerFactoryBean factoryBean = new SchedulerFactoryBean();
  AutowiringSpringBeanJobFactory jobFactory = new AutowiringSpringBeanJobFactory();
  jobFactory.setApplicationContext(applicationContext); 
		
  factoryBean.setJobFactory(jobFactory);    // Set jobFactory to AutowiringSpringBeanJobFactory
  return factoryBean;
}
{% endhighlight %}

Now inject the `JavaMailSender` into the job class:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/job/EmailJob.java' %}

{% highlight java %}
public class EmailJob implements Job {
  @Autowired
  private JavaMailSender mailSender;

  ...

  private void sendEmail(Map<String, Object> map) {
    // Get job details from map and send email
    try {
      this.mailSender.send(..);
    } catch (MailException ex) {
      // simply log it and go on...
    }
  }
}
{% endhighlight %}

That's all folks.

# Conclusion
In this post we learned how to schedule quartz jobs dynamically using `RAMJobStore`. In the 
[next post]({% post_url 2017-10-06-persisting-dynamic-jobs-with-quartz-and-spring %})
we will learn how to persist these jobs into the database using `JDBCJobStore`.

You can find the source to this guide {% include source.html %}. Until the next post, keep doing cool things :smile:.


[cURL]:                     https://curl.haxx.se/
[Initializr]:               https://start.spring.io
[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                    http://maven.apache.org
[Quartz]:                   http://www.quartz-scheduler.org/
