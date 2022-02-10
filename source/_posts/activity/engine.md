---
title: activity5 engine
date: 2022-01-05 19:00:00
tags:
- activity
categories:
- 技术
---

### 关键词：

```
processDefinitionKey：流程定义key
processDefinitionId: 模型部署后生成流程定义ID
businessKey：业务参数
category：用户对自己模型、定义和实例做的分类；
tenantId：租户概念，对应多个系统共享同一个数据库的数据。
```

### 数据库配置：

```
databaseType：h2, mysql, oracle, postgres, mssql, db2
databaseSchemaUpdate：
	false （default）：在构建过程引擎是，检查数据库模式是否和包版本一致，如果不一致就抛出异常
	true：在构建过程引擎是，如果有必要，将执行检查，并执行模式的更新。如果模式不存在就创建
	create-drop：当创建引擎时候就创建数据库模式，当流程引擎关闭的时候就删除模式
	
```

### 历史配置：history

```
    none：跳过有所有历史数据的归档，此配置会使流程高性能执行，但是不会提供历史信息
    activity: 归档所有流程实例和活动实例。在流程实例的末尾，顶级流程实例变量的最新值将被复制到历史变量实例中。没有任何细节将被存档。
    audit：这是默认的。它存档所有流程实例、活动实例、不断保持变量值的同步以及提交的所有表单属性，以便所有通过表单进行的用户交互都是可跟踪的，并且可以进行审计
    full: 这是历史存档的最高水平，因此也是最慢的。此级别存储审计级别中的所有信息，以及所有其他可能的详细信息，主要是流程变量更新。
```

### 表结构：

```
ACT_RE_*: RE stands for repository，表记录着一些静态信息，如流程的定义，流程资源，如图片，规则等
ACT_RU_*: RU stands for runtime，包含流程，任务，参数，任务等等运行时数据，Activity只存储执行中的流程运行时数据，并且当流程结束后将被删除，使运行时表不至于过大，并且执行快速
ACT_ID_*: ID stands for identity  包含一些身份信息，如用户用户组
ACT_HI_*: HI stands for history   包含过去的流程历史数据，如流程实例，任务实例，参数
ACT_GE_*: general  存储多样化数据

ACT_RU_EXECUTION：
ID_：EXECUTION主键，这个主键有可能和PROC_INST_ID_相同，相同的情况表示这条记录为主实例记录。
REV_：表示数据库表更新次数。
PROC_INST_ID_：一个流程实例不管有多少条分支实例，这个ID都是一致的。
BUSINESS_KEY_:这个为业务主键，主流程才会使用业务主键，另外这个业务主键字段在表中有唯一约束。
PARENT_ID_：这个记录表示父实例ID，如上图，同步节点会产生两条执行记录，这两条记录的父ID为主线的ID。
PROC_DEF_ID_ :流程定义ID
SUPER_EXEC ： 这个如果存在表示这个实例记录为一个外部子流程记录，对应主流程的主键ID。
ACT_ID_：表示流程运行到的节点，如上图主实例运行到的节点ID。
IS_ACTIVE_ : 是否活动流程实例
IS_CONCURRENT_:是否并发。上图同步节点后为并发，如果是并发多实例也是为1。
IS_SCOPE_: 这个字段我跟踪了一下不同的流程实例，如会签，子流程，同步等情况，发现主实例的情况这个字段为1，子实例这个字段为0。
TENANT_ID_ :  这个字段表示租户ID。可以应对多租户的设计。
IS_EVENT_SCOPE: 没有使用到事件的情况下，一般都为0。
SUSPENSION_STATE_： 这个表示是否暂停。
CACHE_ENT_STATE :这个暂时还不明白有什么作用。

------------------------------------------------------
act_ge_bytearray:通用资源表
act_re_model：模型表
act_re_deployment：部署信息表
act_re_procdef：流程定义表

act_hi_procinst：启动的流程实例
act_hi_actinst：启动的活动实例

act_ru_execution：启动的执行记录
```

### 变量

```
#流程或执行相关参数设置 多条执行并行，通过 getVariableLocal setVariableLocal 来区分，非多执行情况下效果一样
RuntimeService.getVariable
RuntimeService.getVariableLocal
RuntimeService.setVariable
RuntimeService.setVariableLocal

#任务参数设置，和任务相关，并行任务通过setVariableLocal getVariableLocal来区分，单执行任务效果一样
TaskService.setVariable
TaskService.setVariableLocal
TaskService.getVariable
TaskService.getVariableLocal

```

### 表达式

```
Expressions： Activiti使用UEL进行表达-解析。Uel表示统一表达式语言，是EE6规范的一部分
Activiti uses UEL for expression-resolving，UEL stands for Unified Expression Language and is part of the EE6 specification

有2中类型的表达式：value-expression and method-expression

Value expression：值表达式
解析为一个Value值，默认情况下，所有的流程变量是可以使用的，并且所有的springbean也是可以使用的
resolves to a value. By default, all process variables are available to use. Also all spring-beans (if using Spring) are available to use in expressions. Some examples
${myVar}
${myBean.myProperty}

Method expression:方法表达式
调用带有或不带参数的方法。当调用没有参数的方法时，请确保在方法名称之后添加空括号(因为这将表达式与值表达式区分开来)。传递的参数可以是本身解析的文字值或表达式。
invokes a method, with or without parameters. When invoking a method without parameters, be sure to add empty parentheses after the method-name (as this distinguishes the expression from a value expression). The passed parameters can be literal values or expressions that are resolved themselves
${printer.print()}
${myBean.addNewOrder('orderName')}
${myBean.doSomething(myVar, execution)}
```

这些表达式支持解析原语(包括比较它们)、bean、列表、数组和映射
       

#### 在所有进程变量之上，有一些默认对象可用于表达式中默认情况下

（所有的流程变量是可以使用的，并且所有的springbean也是可以使用的）

+ **execution**：DelegateExecution，它保存有关正在执行的其他信息。

+ **task**：包含有关当前任务的其他信息的DelegateTask。注意：只在任务侦听器计算的表达式中工作

+ **authenticatedUserId**：当前已通过身份验证的用户的id。如果没有对用户进行身份验证，则变量不可用。

**Expressions可以在  Java Service tasks, Execution Listeners, Task Listeners and Conditional sequence flows 中使用**

默认情况下，当使用ProcessEngineFactoryBean时，BPMN进程中的所有表达式也将看到所有Springbean。可以使用您可以配置的映射来限制要在表达式中公开的bean，甚至完全不公开bean。下面的示例公开一个bean(打印机)，可在键“打印机”下使用。若要完全不公开bean，只需将一个空列表作为SpringProcessEngineConfiguration上的Beans属性传递即可。当没有设置bean属性时，上下文中的所有Springbean都是可用的

```
<bean id="processEngineConfiguration" class="org.activiti.spring.SpringProcessEngineConfiguration">
  <property name="beans">
    <map>
      <entry key="printer" value-ref="printer" />
    </map>
  </property>
</bean>

<bean id="printer" class="org.activiti.examples.spring.Printer" />
```



####  Java Service tasks

- activiti:delegateExpression 配置一个委派的Bean对象 通常这个bean在spring中，任务交给指定的对象来执行 对象实现了JavaDelegate
- activiti:class 配置一个类，任务交给指定的类来执行 类实现JavaDelegate
- activiti:expression 配置一个表达式，表达式的执行作为任务的内容

```
指定task执行类
<serviceTask id="javaService"name="My Java Service Task" activiti:class="org.activiti.MyJavaDelegate" />

指定task执行类，delegateExpressionBean 实现了JavaDelegate接口，并且定义在spring中
<serviceTask id="serviceTask" activiti:delegateExpression="${delegateExpressionBean}" />

执行执行脚本方法
<serviceTask id="javaService" name="My Java Service Task" activiti:expression="#{printer.printMessage()}" />

执行带参数的脚本方法
<serviceTask id="javaService"name="My Java Service Task" activiti:expression="#{printer.printMessage(execution, myVar)}" />
```

#### Execution Listeners

- delegateExpression:配置一个委派的Bean对象，通常这个bean在spring中，任务交给指定的对象来执行 对象实现了**org.activiti.engine.delegate.ExecutionListener**
- class:配置一个类，任务交给指定的类来执行 类实现ExecutionListener
- expression：配置一个表达式，表达式的执行作为任务的内容

```
<activiti:executionListener event="start" delegateExpression="${myExecutionListenerBean}" />
<activiti:executionListener expression="${myPojo.myMethod(execution.eventName)}" event="end" />
<activiti:executionListener class="org.activiti.executionlistener.ExampleExecutionListenerOne" event="start" />
```

#### Task Listeners 

**event** 

- assignment:当任务分配给某人时发生。注意：当进程执行到达userTask时，首先会触发一个分配事件，然后再触发CREATE事件。		这似乎是一种不自然的顺序，但其原因是务实的：在接收创建事件时，我们通常希望检查任务的所有属性，包括受让人
- create:在已创建任务并设置了“所有任务”属性时发生
- complete:在任务完成时和任务从运行时数据中删除之前发生。
- delete:在删除任务之前发生。注意，当任务通常通过完全任务完成时，它也将被执行。

**class**

- delegateExpression:配置一个委派的Bean对象，通常这个bean在spring中，任务交给指定的对象来执行 对象实现了**org.activiti.engine.delegate.TaskListener**
- class:配置一个类，任务交给指定的类来执行 类实现ExecutionListener
- expression：配置一个表达式，表达式的执行作为任务的内容

```
<activiti:taskListener event="create" delegateExpression="${myTaskListenerBean}" />
<activiti:taskListener event="create" expression="${myObject.callMethod(task, task.eventName)}" />
<activiti:taskListener event="create" class="org.activiti.MyTaskCreateListener" />

```

#### Conditional sequence flows

目前条件表达式只能与UEL一起使用，有关这些的详细信息可以在部分表达式中找到。使用的表达式应该解析为布尔值，否则在计算条件时会引发异常

```
下面的示例通过getter引用典型JavaBean样式中流程变量的数据
<conditionExpression xsi:type="tFormalExpression">
    <![CDATA[${order.price > 100 && order.price < 250}]]>
</conditionExpression>


此示例调用解析为布尔值的方法
<conditionExpression xsi:type="tFormalExpression">
  <![CDATA[${order.isStandardOrder()}]]>
</conditionExpression>
```



### 表单概念

```
普通表单：每个节点的表单内容都写死在JSP或者HTML中。

动态表单：表单内容存放在流程定义文件中（包含在启动事件以及每个用户任务中）。

外置表单：每个用户任务对应一个单独的<b>.form</b>文件，和流程定义文件同时部署（打包为zip/bar文件）。

综合流程：可以查询到所有的流程（普通、动态、外置固定查询某些流程的表单，为了演示所以单独分开）；综合流程的目的在于可以启动用户上传或者设计    后部署的流程定义

设置表单地址

l 全局表单：新建流程时或活动元素上未设置表单标识时调用的表单，位于开始事件属性中“表单标识”字段，指定表单访问地址。

l 活动表单：当前步骤使用的表单，使用活动节点属性“表单标识”字段。
```

### 设置流程参与者

```
assignee：任务执行人，设置系统中的“登录名”（loginName）。            参与人类型ACT_IDENTITYLINK： participant， 发起人类型:starter

candidateUsers：任务执行人，可以填写多个。                          候选人类型ACT_IDENTITYLINK：candidate   

candidateGroups：任务执行组，可以填写多个，设置系统中的“角色英文名（enname）”。

assignee和candidateUsers的区别是：assignee不需要签收任务，直接可执行任务；candidateUsers为竞争方式分配任务，被指定人待办中都有一条任务，谁先签收谁就获得任务的执行权。参与者可指定流程变量（EL表达式），动态指定参与者，如：${processer}

ASSIGNEE_（受理人）
Task task=taskService.createTaskQuery().singleResult();
//签收
taskService.claim(task.getId(), "billy");


OWNER_（委托人）：受理人委托其他人操作该TASK的时候，受理人就成了委托人OWNER_，其他人就成了受理人ASSIGNEE_
Task task=taskService.createTaskQuery().singleResult();
//委托
taskService.delegateTask(task.getId(), "cc");


当前线程使用申请用户id，并留存在审批流程中
activiti:initiator 
申请用户
try {
  identityService.setAuthenticatedUserId("bono");
  runtimeService.startProcessInstanceByKey("someProcessKey");
} finally {
  identityService.setAuthenticatedUserId(null);
}
```

 	任务操作流程：

```
生成任务并设置候选执行人，候选执行组

候选任或者候选组人进行任务签收

签收人执行处理任务
```

### api

```
//设置部署记录类型，将联动部署相关的流程定义一并更新类型
repositoryService.setDeploymentCategory("10001","JTDLS");
//设置流程定义类型
repositoryService.setProcessDefinitionCategory("process_test_finance:1:10004","JTDLS");
//设置部署的租户信息
repositoryService.changeDeploymentTenantId("10001","AGENT");



```

### Activity与spring集成:

#### 相关依赖

```
<dependency>
    <groupId>org.activiti</groupId>
    <artifactId>activiti-engine</artifactId>
    <version>5.22.0</version>
</dependency>

<dependency>
    <groupId>org.activiti</groupId>
    <artifactId>activiti-spring</artifactId>
    <version>5.22.0</version>
</dependency>
<dependency>
    <groupId>org.activiti</groupId>
    <artifactId>activiti-bpmn-model</artifactId>
    <version>5.22.0</version>
</dependency>
<dependency>
    <groupId>org.activiti</groupId>
    <artifactId>activiti-bpmn-converter</artifactId>
    <version>5.22.0</version>
</dependency>
<dependency>
    <groupId>org.activiti</groupId>
    <artifactId>activiti-image-generator</artifactId>
    <version>5.22.0</version>
</dependency>


```

#### 相关配置

```properties
server.port=80
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/activity_local?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8
spring.datasource.username=root
spring.datasource.password=CHENxiao1989119
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

activity.deployment.deploymentResources[0]=processes/*.bpmn20.xml
```

#### 相关BEAN初始化

```java
@Bean
public SpringProcessEngineConfiguration springProcessEngineConfiguration(DataSource dataSource, PlatformTransactionManager transactionManager){
    SpringProcessEngineConfiguration springProcessEngineConfiguration = new SpringProcessEngineConfiguration();
    springProcessEngineConfiguration.setDataSource(dataSource);
    springProcessEngineConfiguration.setTransactionManager(transactionManager);
    springProcessEngineConfiguration.setDatabaseSchemaUpdate("true");
    springProcessEngineConfiguration.setDatabaseSchema("mysql");
    springProcessEngineConfiguration.setHistoryLevel(HistoryLevel.FULL);
    return  springProcessEngineConfiguration;
}
@Bean
public ProcessEngineFactoryBean processEngineFactoryBean(SpringProcessEngineConfiguration springProcessEngineConfiguration){
    ProcessEngineFactoryBean processEngineFactoryBean = new ProcessEngineFactoryBean();
    processEngineFactoryBean.setProcessEngineConfiguration(springProcessEngineConfiguration);
    return processEngineFactoryBean;
}

@Bean
public RepositoryService repositoryService(ProcessEngine processEngine){
    try {
        return processEngine.getRepositoryService();
    } catch (Exception exception) {
        exception.printStackTrace();
        throw exception;
    }
}

@Bean
public RuntimeService runtimeService(ProcessEngine processEngine){
    try {
        return processEngine.getRuntimeService();
    } catch (Exception exception) {
        exception.printStackTrace();
        throw exception;
    }
}


@Bean
public TaskService taskService(ProcessEngine processEngine){
    try {
        return processEngine.getTaskService();
    } catch (Exception exception) {
        exception.printStackTrace();
        throw exception;
    }
}

@Bean
public HistoryService historyService(ProcessEngine processEngine){
    try {
        return processEngine.getHistoryService();
    } catch (Exception exception) {
        exception.printStackTrace();
        throw exception;
    }
}

@Bean
public ManagementService managementService(ProcessEngine processEngine){
    try {
        return processEngine.getManagementService();
    } catch (Exception exception) {
        exception.printStackTrace();
        throw exception;
    }
}
@Bean
public IdentityService identityService(ProcessEngine processEngine){
    try {
        return processEngine.getIdentityService();
    } catch (Exception exception) {
        exception.printStackTrace();
        throw exception;
    }
}



```

#### 自动资源部署

##### deploymentResources：

```
<bean id="processEngineConfiguration" class="org.activiti.spring.SpringProcessEngineConfiguration">
<property name="deploymentResources"
    value="classpath*:/org/activiti/spring/test/autodeployment/autodeploy.*.bpmn20.xml" />
</bean>
```

```
@Data
@Component
@ConfigurationProperties("activity.deployment")
public class DeploymentResourceConfig {

    private String[] deploymentResources;

    public Resource[] getDeploymentResources() {
        PathMatchingResourcePatternResolver pathMatchingResourcePatternResolver = new PathMatchingResourcePatternResolver();
        List<Resource> resources = new ArrayList<>();
        for (String deploymentResource : deploymentResources) {
            try {
                Resource[] search_res =  pathMatchingResourcePatternResolver.getResources(deploymentResource);
                resources.addAll(Arrays.asList(search_res));
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return resources.toArray(new Resource[]{});
    }

    public void setDeploymentResources(String[] deploymentResources) {
        this.deploymentResources = deploymentResources;
    }
}

```

默认情况下，上面的配置将匹配筛选的所有资源分组到Activiti引擎的单个部署中。防止重新部署未更改资源的重复筛选适用于整个部署。在某些情况下，这可能不是你想要的。例如，如果您以这种方式部署了一组流程资源，并且这些资源中只有一个流程定义已经更改，则整个部署将被视为新的，而部署中的所有流程定义都将被重新部署，从而生成每个流程定义的新版本，即使实际上只有一个被更改。

为了能够自定义确定部署的方式，您可以在SpringProcessEngineConfiguration，DeploymentMode中指定一个附加属性。此属性定义从匹配筛选器的一组资源中确定部署的方式。对于此属性，默认情况下支持3个值。

##### deploymentMode：

- default:将所有资源分组到单个部署中，并对该部署应用重复筛选。这是默认值，如果不指定值，将使用它
- single-resource:为每个单独的资源创建一个单独的部署，并对该部署应用重复筛选。此值用于将每个流程定义分别部署，并仅在更改后创建一个新的流程定义版本
- resource-parent-folder:为共享同一个父文件夹的资源创建单独的部署，并对该部署应用重复筛选。此值可用于为大多数资源创建单独的部署，但仍可以通过将它们放置在共享文件夹中对某些资源进行分组。下面是如何为部署模式指定单资源配置的示例

```
<bean id="processEngineConfiguration" class="org.activiti.spring.SpringProcessEngineConfiguration">
  <property name="deploymentResources" value="classpath*:/activiti/*.bpmn" />
  <property name="deploymentMode" value="single-resource" />
</bean>
```

