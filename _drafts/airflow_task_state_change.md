---
layout: post
title: 'Parse Airflow -- Task State Change'
date: 2019-08-20
author: Calvin Wang
cover: '/assets/img/airflow/scheduler/airflow-scheduler-state-change-code.png'
tags: airflow task state
---

> Alo7 数仓分享. Airflow 作为组内的主要调度工具, 整理了状态变化方便源码阅读. (全文基于Airflow 1.10.3 所写)

## 简单地说
![airflow-scheduler-state-change](/assets/img/airflow/scheduler/airflow-scheduler-state-change.png)
如上图在正常的Scheduler流程中一共有 11 个状态组成(Airflow 中一共有 12 个状态, 还有一个SHUTDOWN 没有在本文中介绍).

### ※下图在查看 Airflow 源码时所画流程图, 下文各条都能在图中找到对应源码位置(图片有些大 右键查看)
![image](/assets/img/airflow/scheduler/airflow-scheduler-state-change-code.png)

### ☆SchedulerJob 会在 DafFileProcessorAgent 的帮助下: 
  * 创建 DagRun(下文用DR表示, 非特殊说明DR均为Running状态) 并为其对应的 Task 创建状态为 `NONE` 的 TaskInstance(下文用TI表示; TIS表示 task instances)
  * 获取 DR 下 符合 QUEUE_DEP 依赖的并且状态为[`NONE`, `UP_FOR_RETRY`, `UP_FOR_RESCHEDULED`] 的 TIS 将其状态改变为 `SCHEDULED`
  * 将 DR 中 已经不存在对应 TASK 的, 非 RUNNING 状态的 TI 状态置为 `REMOVED`
  * 将 DR 中状态为 REMOVED 的TI, 在其对应的TASK再次存在时将其状态重置为 `NONE`
  * 通过 TI#are_dependencies_met 来维护[ `SKIPPED`, `UPSTREAM_FAILED`] TI (下文会讲述)

### ☆SchedulerJob 本身
  * 在启动时将 DR 中的 orphaned tasks 状态重置为 `NONE`
    * 状态为: [SCHEDULED, QUEUEED]
    * Executor Queue 中不存在的 ti
  * 会通过 DafFileProcessorAgent#harvest_simple_dag 方法来获取合适DAG进行调度
    * 处理异常DR的状态改变.( 存在未完成 TI, 但是 DR 状态已经不是 RUNNING 了)(可能是手动修改 DR 状态)
      * 状态为 UP_FOR_RETRY 的 TI 将重置为 `FAILED`
      * 状态为[QUEUED, SCHEDULED, UP_FOR_RESCHEDULED]的 TI 将重置为 `NONE`
    * 查找满足[pool limits, task concurrency, priority]限制并且状态为SCHEDULED的 ti 将其状态置为`QUEUED`
      并生成对应 ti 的执行命令, 放入 Executor 的队列中 等待执行(SchedulerJob#_enqueue_task_instances_with_queued_state).
    * 对于一条已经置为 QUEUED 的 ti 由于某些原因并没有被 Executor 执行. 则重置为 `SCHEDULED` 状态.
    * 针对Executor 报告为已完成(SUCCESS, FAILED)的 ti 但是其在DB状态为`QUEUED`
      * 如果 TASK 还存在 retry 次数, 则设置为 `up_for_retry`
      * 如果 TASK 不存在 retry 次数, 则设置为 `FAILED`
  
### ☆LocalTaskJob 检查 ti 是否满足 RUN_DEPS 依赖.
  * 不满足: 状态重置为 `None`
  * 满足:  状态置为 `RUNNING` 并触发 任务真实执行.

### ☆TaskInstance#task_runner#start 任务真实执行, 并按照程序结果设置状态.
  * 正常结束:                    `SUCCESS`
  * raise AirflowSkipException: `SKIPPED`
  * raise RESCHEDULE:           `RESCHEDULED`
  * 其他异常, 还存在retry次数:    `UP_FOR_RETRY`
  * 其他异常, 不存在retry次数:    `FAILED`


## 依赖管理(airflow.ti_deps)(源码部分有修改)
### class airflow.ti_deps.deps.BaseTIDep
上下文依赖的抽象类. 定义了依赖检查流程.
```python
# dep检查结果
TIDepStatus = namedtuple('TIDepStatus', ['dep_name', 'passed', 'reason'])

class BaseTIDep(object):
  IGNOREABLE = False # 定义该dep是否可以被忽略
  IS_TASK_DEP = False # 顶盖该dep是否是task dep

  def _failing_status(self, reason=''): # dep检查成功
        return TIDepStatus(self.name, False, reason)

  def _passing_status(self, reason=''): # dep检查失败
      return TIDepStatus(self.name, True, reason)
  
  def _get_dep_statuses(self, ti, session, dep_context=None):
        # 由子类实现具体的依赖检查
        raise NotImplementedErro

  def get_dep_statuses(self, ti, session, dep_context=None):
        # 默认行为处理, 如ignore_task deps. 并调用_get_dep_statuses来完成特定依赖(子类覆写)检查 
        from airflow.ti_deps.dep_context import DepContext

        if dep_context is None:
            dep_context = DepContext()

        if self.IGNOREABLE and dep_context.ignore_all_deps:
            yield self._passing_status(
                reason="Context specified all dependencies should be ignored.")
            return

        if self.IS_TASK_DEP and dep_context.ignore_task_deps:
            yield self._passing_status(
                reason="Context specified all task dependencies should be ignored.")
            return

        for dep_status in self._get_dep_statuses(ti, session, dep_context):
            yield dep_status
```

### BaseTIDep 子类举例
* class airflow.ti_deps.deps.DagUnpausedDep 
```python
class DagUnpausedDep(BaseTIDep): # dag 未暂停
    NAME = "Dag Not Paused"
    IGNOREABLE = True

    def _get_dep_statuses(self, ti, session, dep_context):
        if ti.task.dag.is_paused:
            yield self._failing_status(
                reason="Task's DAG '{0}' is paused.".format(ti.dag_id))
```

* class airflow.ti_deps.deps.TriggerRuleDep [上文的TI.are_dependencies_met处理见此]

当TriggerRuleDep除了返回dep是否满足意外. 当 flag_upstream_failed 为True是会生成task 的upstream_failed 和 skipped 状态.
规则如下:

```python
### 如果任务没有直接上游 或者任务的触发条件为DUMMY, 则执行.

if not ti.task.upstream_list:
    yield self._passing_status(
        reason="The task instance did not have any upstream tasks.")
    return

if ti.task.trigger_rule == TR.DUMMY:
    yield self._passing_status(reason="The task had a dummy trigger rule set.")
    return

### 统计任务上游数据.

import airflow.example_dags.example_branch_operator as brh_op
brh_op.dag.get_task('join').upstream_task_ids
# {'follow_branch_a', 'follow_branch_b', 'follow_branch_d', 'follow_branch_c'}

###
# succuss: 上游任务state = success 的任务个数
# skipped: 上游任务state = skipped 的任务个数
# failed: 上游任务state = failed 的任务个数
# upstream_failed: 上游任务state = upstream_failed 的任务个数
# done: 上游任务的任务个数
# upstream_done: done >= len(upstreams)
###
qry = (
    session
    .query(
        func.coalesce(func.sum(
            case([(TI.state == State.SUCCESS, 1)], else_=0)), 0),
        func.coalesce(func.sum(
            case([(TI.state == State.SKIPPED, 1)], else_=0)), 0),
        func.coalesce(func.sum(
            case([(TI.state == State.FAILED, 1)], else_=0)), 0),
        func.coalesce(func.sum(
            case([(TI.state == State.UPSTREAM_FAILED, 1)], else_=0)), 0),
        func.count(TI.task_id),
    )
    .filter(
        TI.dag_id == ti.dag_id,
        TI.task_id.in_(ti.task.upstream_task_ids),
        TI.execution_date == ti.execution_date,
        TI.state.in_([
            State.SUCCESS, State.FAILED,
            State.UPSTREAM_FAILED, State.SKIPPED]),
    )
)
successes, skipped, failed, upstream_failed, done = qry.first()
```

触发规则|上游状态|状态变更为|备注
-|-|-|-
ALL_SUCCESS|upstream_failed !=0 or failed != 0 | UPSTREAM_FAILED |
|upstream_failed =0 and failed = 0 and skipped !=0| SKIPPED |
ALL_FAILED|successes !=0 or skipped !=0 | SKIPPED |
ONE_SUCCESS | upstream_done ==True and success = 0| SKIPPED |
ONE_FAILED | upstream_done != True and failed ==0 and upstream_failed ==0) | SKIPPED 
NONE_FAILED | upstream_failed !=0 or failed !=0 | UPSTREAM_FAILED | 1.10.2增加
| skipped >= upstream | SKIPPED |
NONE_SKIPPED | skipped!=0 | SKIPPED | 1.10.3增加

* 其他dag都是见名知意的, 可以翻翻看.

### class airflow.ti_deps.DepContext
一个Context基类, 用来维护在TI Context 中需要评估的依赖项(deps). 并配置其行为.
```python
class DepContext(object):
    def __init__(
            self,
            deps=None or set(), # 需要被评估的Deps
            flag_upstream_failed=False, # 是否产品 UPSTREAM_FAILED 状态
            ignore_all_deps=False, # 是否忽略所有Deps
            ignore_depends_on_past=False, # 忽略 DAG 的depends_on_past 参数
            ignore_in_retry_period=False, # 忽略 retry 周期
            ignore_in_reschedule_period=False, # 忽略 reschedule 周期
            ignore_task_deps=False, # 忽略 Task 特有依赖, 如 depends_on_past, trigger_rule
            ignore_ti_state=False # 忽略 TI 以往任务状态
    ):
```

## Executor
定义了和工作平台(yarn celery)交互方式
### class airflow.executors.base_executor
Executor 的抽象类
```python
class BaseExecutor(LoggingMixin):
    def __init__(self, parallelism=PARALLELISM):
        self.parallelism = parallelism # 并行task个数
        self.queued_tasks = OrderedDict()
        self.running = {}
        self.event_buffer = {}
    # 启动可以做初始化操作
    def start(self):
      pass

    # 将任务入列
    def queue_command(self, simple_task_instance, command, priority=1, queue=None): 
        self.queued_tasks[key] = (command, priority, queue, simple_task_instance)

    # 同步执行任务
    def sync(self):
      pass
    
    # 异步执行任务
    def execute_async(self,
                      key,
                      command,
                      queue=None,
                      executor_config=None):  # pragma: no cover
        raise NotImplementedError()

    # 由schedulerJob 定期调用
    def heartbeat(self):
      self.execute_async
      self.sync()
```

### class airflow.executors.sequential_executor
```python
class SequentialExecutor(BaseExecutor):
    def __init__(self):
        super(SequentialExecutor, self).__init__()
        self.commands_to_run = []

    def execute_async(self, key, command, queue=None, executor_config=None):
        # one by one 执行不支持并发. 交给同步方法去处理
        self.commands_to_run.append((key, command,))

    def sync(self):
        for key, command in self.commands_to_run:
            self.log.info("Executing command: %s", command)

            try:
                subprocess.check_call(command, close_fds=True)
                self.change_state(key, State.SUCCESS)
            except subprocess.CalledProcessError as e:
                self.change_state(key, State.FAILED)
                self.log.error("Failed to execute task %s.", str(e))
        self.commands_to_run = []

    def end(self):
        self.heartbeat()
```


## JOB