---

title: 'Task Markers'
category: 'Tasks'

keywords: 'multi instance boundary event loop compensation'

---

In addition to those various types of tasks, we can mark tasks as loops, multiple instances, or compensations. Markers can be combined with task types. 

## Multiple Instance

A multi-instance activity is a way of defining repetition for a certain step in a business process. In programming concepts, a multi-instance matches the for each construct: it allows to execute a certain step or even a complete subprocess for each item in a given collection, sequentially or in parallel.

A multi-instance is a regular activity that has extra properties defined (so-called `multi-instance characteristics`) which will cause the activity to be executed multiple times at runtime. Following activities can become a multi-instance activity:

* User Task
* Script Task
* Service Task
* Business Rule Task
* Manual Task
* Receive Task
* (Embedded) Sub-Process
* Call Activity

A Gateway or Event can not become multi-instance.

If an activity is multi-instance, this is indicated by three short lines at the bottom of that activity. Three vertical lines indicates that the instances will be executed in <strong>parallel</strong>, while three horizontal lines indicate **sequential** execution.

<div data-bpmn-diagram="implement/multiple-instance"></div>

As required by the spec, each parent execution of the created executions for each instance will have following variables:

* **nrOfInstances**: the total number of instances
* **nrOfActiveInstances**: the number of currently active, i.e. not yet finished, instances. For a sequential multi-instance, this will always be 1
* **nrOfCompletedInstances**: the number of already completed instances

These values can be retrieved by calling the `execution.getVariable(x)` method.

Additionally, each of the created executions will have an execution-local variable (i.e. not visible for the other executions, and not stored on process instance level) :

* **loopCounter**: indicates the index in the for-each loop of that particular instance

To make an activity multi-instance, the activity xml element must have a `multiInstanceLoopCharacteristics` child element.

```xml
<multiInstanceLoopCharacteristics isSequential="false|true">
 ...
</multiInstanceLoopCharacteristics> 
```

The isSequential attribute indicates if the instances of that activity are executed sequentially or parallel.


The number of instances are calculated once, when entering the activity. There are a few ways of configuring this. On way is directly specifying a number, by using the `loopCardinality` child element.

```xml
<multiInstanceLoopCharacteristics isSequential="false|true">
  <loopCardinality>5</loopCardinality>
</multiInstanceLoopCharacteristics> 
```

Expressions that resolve to a positive number are also possible:

```xml
<multiInstanceLoopCharacteristics isSequential="false|true">
  <loopCardinality>${nrOfOrders-nrOfCancellations}</loopCardinality>
</multiInstanceLoopCharacteristics>
```

Another way to define the number of instances, is to specify the name of a process variable which is a collection using the `loopDataInputRef` child element. For each item in the collection, an instance will be created. Optionally, it is possible to set that specific item of the collection for the instance using the inputDataItem child element. This is shown in the following XML example:

```xml
<userTask id="miTasks" name="My Task ${loopCounter}" camunda:assignee="${assignee}">
  <multiInstanceLoopCharacteristics isSequential="false">
    <loopDataInputRef>assigneeList</loopDataInputRef>
    <inputDataItem name="assignee" />
  </multiInstanceLoopCharacteristics>
</userTask> 
```

Suppose the variable assigneeList contains the values [kermit, gonzo, foziee]. In the snippet above, three user tasks will be created in parallel. Each of the executions will have a process variable named assignee containing one value of the collection, which is used to assign the user task in this example.

The downside of the `loopDataInputRef` and `inputDataItem` is that 1) the names are pretty hard to remember and 2) due to the BPMN 2.0 schema restrictions they can't contain expressions. We solve this by offering the collection and elementVariable attributes on the multiInstanceCharacteristics:

```xml
<userTask id="miTasks" name="My Task" camunda:assignee="${assignee}">
  <multiInstanceLoopCharacteristics isSequential="true" 
     camunda:collection="${myService.resolveUsersForTask()}" camunda:elementVariable="assignee" >
  </multiInstanceLoopCharacteristics>
</userTask>
```

A multi-instance activity ends when all instances are finished. However, it is possible to specify an expression that is evaluated every time one instance ends. When this expression evaluates to true, all remaining instances are destroyed and the multi-instance activity ends, continuing the process. Such an expression must be defined in the completionCondition child element.

```xml
<userTask id="miTasks" name="My Task" camunda:assignee="${assignee}">
  <multiInstanceLoopCharacteristics isSequential="false" 
     camunda:collection="assigneeList" camunda:elementVariable="assignee" >
    <completionCondition>${nrOfCompletedInstances/nrOfInstances >= 0.6 }</completionCondition>
  </multiInstanceLoopCharacteristics>
</userTask> 
```

In this example, there will be parallel instances created for each element of the assigneeList collection. However, when 60% of the tasks are completed, the other tasks are deleted and the process continues.

## Boundary events and multi-instance

Since a multi-instance is a regular activity, it is possible to define a boundary event on its boundary. In case of an interrupting boundary event, when the event is caught, all instances that are still active will be destroyed. Take for example following multi-instance subprocess:

<div data-bpmn-diagram="implement/multiple-instance-boundary"></div>

Here, all instances of the subprocess will be destroyed when the timer fires, regardless of how many instances there are or which inner activities are currently not yet completed. 

## Loops

The loop marker is not yet natively supported by the engine. For Multiple Instance the number of repitions is known in advance - which makes it a bad candidate for loops (anyway - since it defines a completion condition that may be already sufficient for some cases).

To get around this limitation the solution is to explicitely model the loop in your bpmn process:

<div data-bpmn-diagram="implement/loop-alternative"></div>

And be assured that we have the loop marker in our backlog to be added to the engine.

## Compensation

If an activity is used for compensating the effects of another activity, it can be declared to be a compensation handler. Compensation handlers are not contained in normal flow and are only executed when a compensation event is thrown.

<div data-bpmn-diagram="implement/compensation-marker"></div>

Notice the compensation handler icon in the bottom canter area of the "cancel hotel reservation" service task

Compensation handlers must not have incoming or outgoing sequence flows.

A compensation handler must be associated with a compensation boundary event using a directed association.

In order to declare an activity to be a compensation handler, we need to set the attribute isForCompensation to true:

```xml
<serviceTask id="undoBookHotel" isForCompensation="true" camunda:class="..." />
```

## Additional Resources

* [Task Markers in the BPMN Tutorial](http://camunda.org/design/reference.html#!/activities/tasks)
* [Transaction Subprocess](ref:#subprocesses-transaction-subprocess)