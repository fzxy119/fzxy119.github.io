---
title: 工作流引擎Activity事件
date: 2022-01-07 16:27:10
tags:
- activity
categories:
- 技术
---

### 事件功能

事件用来表明流程的生命周期中发生了什么事。 事件总是画成一个圆圈。在BPMN 2.0中，事件有两大分类：*捕获（catching）* 或 *触发（throwing）* 事件。        

- **捕获（Catching）：**当流程执行到事件，它会等待被触发。触发的类型是由内部图表或XML中的类型声明来决定的。 捕获事件与触发事件在显示方面是根据内部图表是否被填充来区分的（白色的）。            
- **触发（Throwing）：**当流程执行到事件，会触发一个事件。触发的类型是由内部图表或XML中的类型声明来决定的。触发事件与捕获事件在显示方面是根据内部图表是否被填充来区分的（被填充为黑色）。

### 事件位置：

起始：事件位于流程的起始位置

中间：事件位于流程的中间位置

边界：事件位于节点或者子流程的边界

结束：事件位于流程的结束位置

### 事件定义:

空事件：可用于开始事件，中间事件，结束事件

定时器事件：可用于开始事件，中间事件，边界事件

错误事件：可用于开始事件，边界事件，结束事件

信号事件：可用于开始事件，中间事件，边界事件

消息事件：可用于开始事件，中间事件，边界事件

补偿事件：可用于边界事件

取消事件：可用于边界事件，结束事件

|        | 起始 | 中间 | 边界 | 结束 |      |      |
| ------ | ---- | ---- | ---- | ---- | ---- | ---- |
| 空事件 | -    | -    |      | -    |      |      |
| 错误   | -    |      | -    | -    |      |      |
| 定时   | -    | -    | -    |      |      |      |
| 信号   | -    | -    | -    |      |      |      |
| 消息   | -    | -    | -    |      |      |      |
| 补偿   |      |      | -    |      |      |      |
| 取消   |      |      | -    |      |      |      |



### 开始事件

#### 空的开始事件 

#### 消息开始事件

- **消息开始事件的名称在给定流程定义中不能重复**。流程定义不能包含多个名称相同的消息开始事件。如果两个或以上消息开始事件应用了相同的事件，或两个或以上消息事件引用的消息名称相同，activiti会在发布流程定义时抛出异常

- **消息开始事件的名称在所有已发布的流程定义中不能重复**，如果一个或多个消息开始事件引用了相同名称的消息，而这个消息开始事件已经部署到不同的流程定义中，activiti就会在发布时抛出一个异常

- 流程版本：**在发布新版本的流程定义时，之前订阅的消息订阅会被取消**。如果新版本中没有消息事件也会这样处理
  

  #### 启动流程实例

  ```
  ProcessInstance startProcessInstanceByMessage(String messageName);
  ProcessInstance startProcessInstanceByMessage(String messageName, Map<String, Object> processVariables);
  ProcessInstance startProcessInstanceByMessage(String messageName, String businessKey, Map<String, Object< processVariables);
  ```

  - 消息开始事件只支持顶级流程。消息开始事件不支持内嵌子流程。						 
  - 如果流程定义有多个消息开始事件，`runtimeService.startProcessInstanceByMessage(...)`会选择对应的开始事件。						 
  - 如果流程定义有多个消息开始事件和一个空开始事件。`runtimeService.startProcessInstanceByKey(...)`和							 `runtimeService.startProcessInstanceById(...)`会使用空开始事件启动流程实例。 
  - 如果流程定义有多个消息开始事件，而且没有空开始事件，`runtimeService.startProcessInstanceByKey(...)`和							 `runtimeService.startProcessInstanceById(...)`会抛出异常。						 
  - 如果流程定义只有一个消息开始事件，`runtimeService.startProcessInstanceByKey(...)`和							 `runtimeService.startProcessInstanceById(...)`会使用这个消息开始事件启动流程实例。						 
  - 如果流程被调用环节（callActivity）启动，消息开始事件只支持如下情况：							 
    - 在消息开始事件以外，还有一个单独的空开始事件
    - 流程只有一个消息开始事件，没有空开始事件。

#### 定时开始事件：

- **timeDate**：

使用 [ISO 8601](http://en.wikipedia.org/wiki/ISO_8601#Dates) 格式指定一个确定的时间，触发事件的时间

```
<timerEventDefinition>
    <timeDate>2011-03-11T12:13:14</timeDate>
</timerEventDefinition>
```

- **timeDuration**:

指定定时器之前要等待多长时间*timeDuration*可以设置为*timerEventDefinition*的子元素,使用[ISO 8601](http://en.wikipedia.org/wiki/ISO_8601#Durations)规定的格（等待10天）。 

```
<timerEventDefinition>
    <timeDuration>P10D</timeDuration>
</timerEventDefinition>
```

- **timeCycle**

指定重复执行的间隔， 可以用来定期启动流程实例，或为超时时间发送多个提醒。timeCycle元素可以使用两种格式。第一种是		 [ISO 8601](http://en.wikipedia.org/wiki/ISO_8601#Repeating_intervals)标准的格式。示例（重复3次，每次间隔10小时）：

```
<timerEventDefinition>
    <timeCycle>R3/PT10H</timeCycle>
</timerEventDefinition>
```

可以使用cron表达式指定timeCycle，下面的例子是从整点开始，每5分钟执行一次

```
0 0/5 * * * ?
```

**时间段表示法**：

如果要表示某一作为一段时间的时间期间，前面加一大写字母P，但时间段后都要加上相应的代表时间的大写字母。如在一年三个月五天六小时七分三十秒内，可以写成P1Y3M5DT6H7M30S

**重复时间表示法**

前面加上一大写字母R，如要从2004年5月6日北京时间下午1点起重复半年零5天3小时，要重复3次，可以表示为R3/20040506T130000+08/P0Y6M5DT3H0M0S。

#### 信号开始事件:

signal开始事件，可以用来通过一个已命名的信号（signal）来启动一个流程实例。信号可以在流程实例内部使用“中间信号抛出事务”触发，也可以通过API(*runtimService.signalEventReceivedXXX* 方法)触发。两种情况下，所有流程实例中拥有相同名称的signalStartEvent都会启动

