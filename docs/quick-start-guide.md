# 快速指南

Workflower 是一个 PHP 的 [BPMN 2.0](http://www.omg.org/spec/BPMN/2.0/) 工作流引擎，并且是在 [The BSD 2-Clause 协议](https://opensource.org/licenses/BSD-2-Clause) 下的开源产品. Workerflower 的主要用途在 PHP 程序中管理以人为本的商业流程。

这篇文章用来阐述在 Symfony 程序中使用的 Workflower 管理商业流程所需的工作。

<!-- TOC -->

- [Quick Start Guide](#quick-start-guide)
    - [Symfony integration with Workflower using PHPMentorsWorkflowerBundle](#symfony-integration-with-workflower-using-phpmentorsworkflowerbundle)
    - [Installing Workflower and PHPMentorsWorkflowerBundle](#installing-workflower-and-phpmentorsworkflowerbundle)
    - [Configuraring PHPMentorsWorkflowerBundle](#configuraring-phpmentorsworkflowerbundle)
    - [Designing workflows with BPMN](#designing-workflows-with-bpmn)
        - [Workflow elements supported by Workflower](#workflow-elements-supported-by-workflower)
    - [Designing entities that represent instances of workflow](#designing-entities-that-represent-instances-of-workflow)
    - [Designing domain services for managing business processes](#designing-domain-services-for-managing-business-processes)
    - [Managing business processes](#managing-business-processes)
        - [Starting a process](#starting-a-process)
        - [Managing processes with Process Console](#managing-processes-with-process-console)
            - [Process operation views](#process-operation-views)
            - [Process list views](#process-list-views)
        - [Designing the persistent process model](#designing-the-persistent-process-model)
    - [Toward the realization of Generative Programming with BPMS](#toward-the-realization-of-generative-programming-with-bpms)
    - [References](#references)

<!-- /TOC -->

## Symfony 使用 PHPMentorsWorkflowerBundle 来与 Workflower 进行交互

[PHPMentorsWorkflowerBundle](https://github.com/phpmentors-jp/workflower-bundle) 是在 Symfony 程序里使用的 Workflower 的交互层，提供了以下特性：

- 根据工作流自动生成 DI 容器服务并使用 `phpmentors_workflower.process_aware` 标签自动注射服务对象
- 分配任务相关部分，并使用 Symfony 安全系统控制相关部分的准入。
- 使用 [Doctrine ORM](http://www.doctrine-project.org/projects/orm.html) 为数据实体提供透明的序列/反序列
- 支持多工作流上下文 (也就是 BPMN 文件的存储目录)

## 安装 Workflower 和 PHPMentorsWorkflowerBundle

首先，我们将使用 Composer 来安装 Workflower 和 PHPMentorsWorkflowerBundle 作为项目依赖包:

```console
$ composer require phpmentors/workflower "1.3.*"
$ composer require phpmentors/workflower-bundle "1.3.*"
```

第二，修改 `Appkernel` 来开启 `PHPMentorsWorkflowerBundle`:

```php
// app/AppKernel.php

class AppKernel extends Kernel
{
    public function registerBundles()
    {
        $bundles = array(
            // ...
            new PHPMentors\WorkflowerBundle\PHPMentorsWorkflowerBundle(),
        );

        // ...

        return $bundles;
    }
}
```

## 配置 PHPMentorsWorkflowerBundle

接下来，配置 `PHPMentorsWorkflowerBundle`。下边展示一个示例:

```yaml
# app/config/config.yml

# ...

phpmentors_workflower:
    serializer_service: phpmentors_workflower.base64_php_workflow_serializer
    workflow_contexts:
        app:
            definition_dir: "%kernel.root_dir%/../src/AppBundle/Resources/config/workflower"

# ...
```

- `serializer_service` - 设定这个 DI 容器服务的 ID 来序列化工作流的实例 `PHPMentors\Workflower\Workflow\Workflow` 对象。  `PHPMentors\Workflower\Persistence\WorkflowSerializerInterface` 的实例已经为特定的服务所预期。默认是`phpmentors_workflower.php_workflow_serializer`。你也可以使用 `phpmentors_workflower.base64_php_workflow_serializer` 以 MIME base64 来 encode/decode 已序列化的对象。
- `workflow_contexts` - 为每个工作流上下文 ID 设定 `definition_dir` (用来存储 BPMN 文件的目录)。

## 以 BPMN 设计工作流

定义一个工作流来与 Workflower 一起工作，可以使用支持 BPMN 2.0 的编辑器。最初最好定义一个只包含开始事件、任务和结束事件的工作流，然后看到这个工作流从始至终工作正常再设计和定义整个工作流。BPMN 文件 的名称用来作为 `the workflow ID` 。以类似 `LoanRequestProcess.bpmn` 这样 的名称保存它。分支的次序流程的条件表达将使用 [the Symfony ExpressionLanguage 组件](http://symfony.com/doc/3.1/components/expression_language.html). 注意次序流程评估顺序还没定义，所以必须设定与其他分支目的地一致的条件表达式。在条件表达中，你可以使用从 `PHPMentors\Workflower\Process\ProcessContextInterface::getProcessData()`返回的关联数组键。

下边的截图是 [BPMN2 Modeler](https://www.eclipse.org/bpmn2-modeler/), 一个基于 [Eclipse](https://eclipse.org/) 的 BPMN 编辑器:

![Editing a BPMN 2.0 model by BPMN2 Modeler](https://cloud.githubusercontent.com/assets/52985/20619334/57813132-b337-11e6-80dd-36e1c873eb81.png)

### 被 Workflower 支持的工作流组件

Workflower 支持下边的 BPMN 2.0 工作流元素:

- 连接对象
    - 顺序流
- 流对象
    - 活动
        - 任务
        - 服务任务
        - 发送任务
    - 事件
        - 开始事件
        - 结束事件
    - 网关
        - 独家网关
- 泳道
    - 通道

请注意 worflower 不支持的元素将被忽略。

## 设计代表工作流实例的数据实体

设计一个数据实体来保存代表一个特定工作流的实例 (在 Workflower 被称作 `process`) and add it to the application. This entity usually implements `PHPMentors\Workflower\Process\ProcessContextInterface` and `PHPMentors\Workflower\Persistence\WorkflowSerializableInterface`. The associative array returned from `PHPMentors\Workflower\Process\ProcessContextInterface::getProcessData()` is expanded in the conditional expression of the sequence flows in the workflow.

It's also a good idea to provide properties that hold a snapshot of some properties of the `Workflow` object according to the need in the application (e.g. querying the database). For example, if your application needs to search the database for processes that remain in a particular activity, add `$currentActivity` to the entity that represents the current activity. An example is shown below:

```php
// ...

use PHPMentors\Workflower\Persistence\WorkflowSerializableInterface;
use PHPMentors\Workflower\Process\ProcessContextInterface;
use PHPMentors\Workflower\Workflow\Workflow;
// ...

class LoanRequestProcess implements ProcessContextInterface, WorkflowSerializableInterface
{
    // ...

   /**
     * @var Workflow
     */
    private $workflow;

    /**
     * @var string
     *
     * @Column(type="blob", name="serialized_workflow")
     */
    private $serializedWorkflow;
    
    // ...

    /**
     * {@inheritdoc}
     */
    public function getProcessData()
    {
        return array(
            'foo' => $this->foo,
            'bar' => $this->bar,
            // ...
        );
    }

    /**
     * {@inheritdoc}
     */
    public function setWorkflow(Workflow $workflow)
    {
        $this->workflow = $workflow;
    }

    /**
     * {@inheritdoc}
     */
    public function getWorkflow()
    {
        return $this->workflow;
    }

    /**
     * {@inheritdoc}
     */
    public function setSerializedWorkflow($workflow)
    {
        $this->serializedWorkflow = $workflow;
    }

    /**
     * {@inheritdoc}
     */
    public function getSerializedWorkflow()
    {
        if (is_resource($this->serializedWorkflow)) {
            return stream_get_contents($this->serializedWorkflow, -1, 0);
        } else {
            return $this->serializedWorkflow;
        }
    }

    // ...
}
```

## Designing domain services for managing business processes

In order to use Workflower at the production level, you will need domain services for starting processes, allocating/starting/ completing work items. In order to link the processes of a specific workflow with some domain services, implement `PHPMentors\Workflower\Process\ProcessAwareInterface` and tag the DI container services of the domain services with the `phpmentors_workflower.process_aware` tag. An example is shown below:

```php
// ...

use PHPMentors\DomainKata\Entity\EntityInterface;
use PHPMentors\Workflower\Process\Process;
use PHPMentors\Workflower\Process\ProcessAwareInterface;
use PHPMentors\Workflower\Process\WorkItemContextInterface;
// ...

class LoanRequestProcessCompletionUsecase implements ProcessAwareInterface
{
    // ...

    /**
     * @var Process
     */
    private $process;

    // ...
    
    /**
     * {@inheritdoc}
     */
    public function setProcess(Process $process)
    {
        $this->process = $process;
    }

    // ...

    /**
     * {@inheritdoc}
     */
    public function run(EntityInterface $entity)
    {
        assert($entity instanceof WorkItemContextInterface);

        $this->process->completeWorkItem($entity);

        // ...
    }

    // ...
}
```

The service definition according to the above is as follows:

```yaml
# ...

app.loan_request_process_completion_usecase:
    class: "%app.loan_request_process_completion_usecase.class%"
    tags:
        - { name: phpmentors_workflower.process_aware, workflow: LoanRequestProcess, context: app }
    # ...

# ...
```

Implementation of the use case class corresponding to "combination of process and operation" as in this example can be said to be basic, but if you can analyze the commonality and variability of process operations and extract the variability to the outside, you can also combine operations into a single class.

## Managing business processes

Finally, it is necessary to implement clients (controllers, commands, event listeners, etc.) to start processes, allocate/start/complete work items. These clients should also be able to extract variability to the outside.

Once you have done so, you will be able to perform a series of operations on the business process from the web interface or the command line interface (CLI).

### Starting a process

The lifecycle of a workflow process (often called a process instance) starts by triggering a start event from some application event (e.g. an account opening application from a web page). Once the process is started, it will be subject to management.

### Managing processes with Process Console

For process management, one or more sets of a list-operation (generally known as list-detail) views can be used. These views and the features provided by their seem to be called **Process Console** in some products. 

#### Process operation views
 
A process operation view, which is detail or operation view, will consist of the following items:

- The process entity and its related entities
    - Some of these are evaluated as process data in Expression Language expressions when selecting sequence flows after completing an activity
- Buttons that are enabled / disabled according to the state of the activity
    - Allocate (the activity to the authenticated user or specified user)
    - Start
    - Complete
- The activity Log for the process

In addition, in the operation view we've seen the checklist to complete the activity, buttons for other operations related to the activity, the audit log for the process, etc. in the real world.

#### Process list views

A process list view is used to find processes matched with some conditions. In the list view it is useful to group the processes by the current activity ID, the process state, etc.. Their groups can be represented as tabs or nodes in a tree such like Gmail.

### Designing the persistent process model

The design and use of an excellent persistent model is one of the most important things in business systems. Persistence of workflow processes is, in a narrow sense, persistence of `Workflow` objects,
but in an actual system a model of business processes, that is the persistent process model for your system, including` Workflow` object should be considered. In this model, different process data items and search keys for each workflow should also be considered. The models that we consider effective are the following:

1. Individual `ProcessContextInterface` implementations for each workflow
2. A common `ProcessContextInterface` implementation for all workflows
3. A combination of an entity that has common properties for all workflows and individual `ProcessContextInterface` implementations for each workflow
    - with inheritance(*1)
    - with composition

An example of the third model with the *Class Table Composition* (*2) is shown in the following figure.

![An example of third model with the Class Table Composition](https://cloud.githubusercontent.com/assets/52985/25310218/9b1d17d2-281a-11e7-9721-dce3b912d3b5.png)

---

1. See [Single Table Inheritance](https://martinfowler.com/eaaCatalog/singleTableInheritance.html), [Class Table Inheritance](https://martinfowler.com/eaaCatalog/classTableInheritance.html), [Concrete Table Inheritance](https://martinfowler.com/eaaCatalog/concreteTableInheritance.html) described in [Catalog of Patterns of Enterprise Application Architecture](https://www.martinfowler.com/eaaCatalog/).
2. *Class Table Composition*: this is a pattern I had used in the real world over the past two years, similar to PofEAA's Class Table Inheritance, except that it uses composition rather than inheritance.

## Toward the realization of Generative Programming with BPMS

In this article we have seen the work required to manage business processes using Workflower on Symfony applications. Workflower and PHPMentorsWorkflowerBundle will only provide the `Workflow` domain model and basic integration layer corresponding to BPMN 2.0 workflow elements, so further work (and skill) will be required to actually create the workflow system on the application. It is by no means easy. This is because it is the design of a BPMS (Business Process Management System) or BPMS framework suitable for the target domain.

In addition, software development by BPMS can be said to be the practice of [Generative Programming](https://www.amazon.com/dp/0201309777/ref=cm_sw_r_tw_dp_x_Yj8nybT850KQV) in the business process domain. Currently there are few BPMS available in PHP, and only people who try to create it will achieve results.

## References

- [phpmentors-jp/workflower-bundle](https://github.com/phpmentors-jp/workflower-bundle)
- [phpmentors-jp/workflower](https://github.com/phpmentors-jp/workflower)
- [Q-BPM](http://en.q-bpm.org/mediawiki/index.php?title=Main_Page)
