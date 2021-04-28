# Airflow

## Overview

- Airbnb >>> Apache

- 举个例子：有一个工作流，包含任务A/B/C，且有以下要求
  
  - 顺序A > B > C
  - Task#A Timeout 5min
  - Task#B 如果不成功可以重试5次
  - 每天10AM运行整个流程
  - 成功后通知，失败后报警
  
- 一个workflow调度和监控工具、平台
  
- Airflow
  
  - 通过DAG【Directed Acyclic Graph】【有向无环图】来管理调度一个工作流内任务流程
  - 并不处理task本身
  - 关注tasks的管理，控制运行启动时间、tasks之间的依赖关系、顺序、周期、
  - 提供了WebUI来方便的可视化依赖关系，监视任务进度状态等 [Airflow - 201](http://7.40.23.201:31867/admin/)
  
- Airflow架构

  ![](https://pic3.zhimg.com/80/v2-dea00d40725d0588fd67317c4d4e3b82_720w.jpg)

- **理解：**
  - DAG定义了工作流即一组任务tasks和其运行的逻辑
  - Task定义了一个具体的任务

## Concepts

### Core Concept
#### DAG

- DAG (Directed Acyclic Graph) 【有向无循环图】
  - 定义了一组用户想要运行的任务tasks，同时定义了任务之间的关联和依赖关系
  - DAG 的定义语言Python
- DAG并不关心Task的内容，只关注运行的时间，关系，顺序，报错之后如何处理
- DAG files默认储存路径```~/airflow/dags```可以通过修改config文件```dags_folder = /Users/e0465278/airflow/dags```来指明其他路径
- 【Note】 默认只检查dags文件名包含```airflow```和```DAG```可通过修改config文件```dag_discovery_safe_mode = True```
- 【理解】可以吧DAG文件看作一个workflow工作流的config文件

#### Scope

- Airflow会尝试load DAG files中所有的DAG，但是DAG must appears in ```gloabs()```
- e.g. my_function中的DAG就不会被load
- 
  
  ```python
  dag_1 = DAG('this_dag_will_be_discovered')
  def my_function():
      dag_2 = DAG('but_this_dag_will_not')
  my_function()
  ```
  
- 但是可以使用```SubDagOperator```来定义一个subdag inside a function
  
  - 把重复度较高的任务放在subdag这样能使代码和流程更整洁易于维护，参见#Subdag

#### Default Arguments

- 定义默认变量

- will apply them to operator

- 在operator中可以override

- e.g.
```python
default_args = {
    'start_date': datetime(2016, 1, 1),
    'owner': 'airflow'
}

dag = DAG('my_dag', default_args=default_args)
op = DummyOperator(task_id='dummy', dag=dag)
print(op.owner) # Airflow
```

####  Context Manager

- DAGs可以用作上下文管理器，往其中加入新的operator
- e.g.

```python
with DAG('my_dag', start_date=datetime(2016, 1, 1)) as dag:
    op = DummyOperator('op')
op.dag is dag # True
```

#### DAG run

- 是DAG定义的workflow的实例化，包含了在specific的execution_date其中的task run
- 【Note】例如一个dag定义了每月1号运行一组任务，2020-12-01这次运行就是一个dag run
- DAG run 可以通过Airflow scheduler或者人工trigger去创建

#### Execution_date

- 运行日期
- **注意：是逻辑日期时间，不是实际时间**
- e.g. **execution_date**可能是2020-11-30，但是实际上是在2020-12-04运行的
- 【可以理解成模拟的某个时间运行】

#### Tasks

- Task定义的了DAG中的最小任务单位，用python定义，是DAG中的一个node
- 每个task都是个Operator【的实践】； e.g ```PythonOperator，BashOperator```
- Tasks的关系、依赖
  - 定义了```Task1 >> Task2```
  - 运行顺序：load DAG > run task1 > task1 success > run task2

#### Task Instances

- 和DAG run概念类似，是task定义的任务的实例
- task run属于DAG run，both have associated **execution_date**
- 状态：
  - running
  - success
  - failed
  - skipped
  - up-for-retry
- Task Instances之间的关系和依赖
  - DAG ( task1 >> task2 ) 
  - DAG run1 [2020-11-30]  ( task1run [2020-11-30] >> task2 run [2020-11-30]) 
  - DAG run2 [2020-12-04]  ( task1run [2020-12-04] >> task2 run [2020-12-04])
  - DAG run1 [2020-11-30]  与 DAG run2 [2020-12-04] 独立
  - task1run [2020-11-30] 和 task1run [2020-12-04] 独立
  - task2run [2020-11-30] 和 task2run [2020-12-04] 独立

#### Task Life Cycle

- WebUI 里面能很直观展示

  ![](https://airflow.apache.org/docs/stable/_images/task_stages.png)

![Life Cycle](https://airflow.apache.org/docs/stable/_images/task_lifecycle_diagram.png)

#### Operators

- Operators定义了每个task能具体做什么
- 常见operators
  - **[`BashOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/bash_operator/index.html#airflow.operators.bash_operator.BashOperator) - executes a bash command**
  - **[`PythonOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/python_operator/index.html#airflow.operators.python_operator.PythonOperator) - calls an arbitrary Python function**
  - [`EmailOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/email_operator/index.html#airflow.operators.email_operator.EmailOperator) - sends an email
  - [`SimpleHttpOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/http_operator/index.html#airflow.operators.http_operator.SimpleHttpOperator) - sends an HTTP request
  - [`MySqlOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/mysql_operator/index.html#airflow.operators.mysql_operator.MySqlOperator), [`SqliteOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/sqlite_operator/index.html#airflow.operators.sqlite_operator.SqliteOperator), [`PostgresOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/postgres_operator/index.html#airflow.operators.postgres_operator.PostgresOperator), [`MsSqlOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/mssql_operator/index.html#airflow.operators.mssql_operator.MsSqlOperator), [`OracleOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/oracle_operator/index.html#airflow.operators.oracle_operator.OracleOperator), [`JdbcOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/jdbc_operator/index.html#airflow.operators.jdbc_operator.JdbcOperator), etc. - executes a SQL command
  - `Sensor` - an Operator that waits (polls) for a certain time, file, database row, S3 key, etc…
- Operator通常能独立完成工作
- 如果不同的operators【task之间】需要share info 【file name，arg name，small amount of data】
  - 应该考虑合并
  - Airflow提供XCom feature用来在operator之间传递数据

##### DAG Assignment

- 直接Assign - explicitly when the operator is created
- 先定义Operator，之后再assignDAG - through deferred assignment
- 通过link Operator 去定义 - inferred from other operators【link的Operator必须都在同一个DAG下】

##### Bitshift Composition

- Recommended

  ```python
  op1 >> op2 >> op3 << op4
  ```

- equal to

  ```python
  op1.set_downstream(op2)
  op2.set_downstream(op3)
  op3.set_upstream(op4)
  ```
  
- also work with dag

  ```python
  dag >> op1 >> op2
  ```

- other e.g.

  ```python
  op1 >> [op2, op3] >> op4
  ```

##### Cross Downstream

- 针对一些重复度高不好写的关系，可以用```chain` and `cross_downstream```

- e.g.

  ```python
  [op1, op2, op3] >> op4
  [op1, op2, op3] >> op5
  [op1, op2, op3] >> op6
  cross_downstream([op1, op2, op3], [op4, op5, op6])
  
  op1 >> op2 >> op3 >> op4 >> op5
  chain(op1, op2, op3, op4, op5)
  
  chain([DummyOperator(task_id='op' + i, dag=dag) for i in range(1, 6)])
  
  chain(op1, [op2, op3], op4)
  ```

#### Workflow几个核心概念review

- **DAG**: The work (tasks), and the order in which work should take place (dependencies), written in Python.
- **DAG Run**: An instance of a DAG for a particular logical date and time.
- **Operator**: A class that acts as a template for carrying out some work.
- **Task**: Defines work by implementing an operator, written in Python.
- **Task Instance**: An instance of a task - that has been assigned to a DAG and has a state associated with a specific DAG run (i.e for a specific execution_date).
- **execution_date**: The logical date and time for a DAG Run and its Task Instances.

### Additional Functionality

#### Hooks

- 链接数据库

- 参见[List Airflow hooks](https://airflow.apache.org/docs/stable/_api/index.html#pythonapi-hooks)

- ```python
  from airflow.hooks.postgres_hook import PostgresHook
  src = PostgresHook(
    postgres_conn_id = 'source',
    shcema = 'source_shcema'
  )
  src_conn = src.get_conn()
  cursor = src_conn.cursor()
  query = "select * from users;"
  cursor.execute(query)
  src_cursor.close()
  src_conn.close()
  ```

#### Pools

- 处理大量任务时候，能力有限，可以限制任务同时进行数量
- 可先设置pool大小，并把task定义时assign到指定pool
- 可设置priority weight来决定先做什么
- pool满了之后排队任务阻塞，pool有空位之后放行
- 如果不assign pool，默认给default_pool [128 slots]

#### Connections

- Config 数据库链接
- can be managed in the UI (`Menu -> Admin -> Connections`)

#### Queues

- 指定worker



#### XComs

- cross communication 用来在tasks之间传递数据

- **Memo:**

  - XCOM is not meant to exchange large amounts of data between tasks. So, if space considerations is the reason for deleting XCOM values, a better solution would be to avoid using XCOM and write out the data to a shared storage.

- 数据通过键值对K、V储存在db中【不会自动删除，可以考虑使用```on_success_callback=cleanup_xcom```】

- 【note】Any object that can be pickled can be used as an XCom value, so users should make sure to use objects of appropriate size.

- e.g.

  ```python
  # inside a PythonOperator called 'pushing_task'
  def push_function():
      return value
  
  # inside another PythonOperator where provide_context=True
  def pull_function(**context):
      value = context['task_instance'].xcom_pull(task_ids='pushing_task')
  ```

  - 【note】此处```**context```内```context['task_instance']```需要用```'ti' or 'task_instance'```

#### Custom XCom Backend

#### Veriables

- 让user可以自定义变量

- 存贮在airflow中的变量（键值对）

- 【note】虽然大部分值应该定义在code里面，好处是能有部分variables或者是config能通过WebUI方便修改更新

- 可通过WebUI，code, CLI去update

- e.g.

  ```python
  from airflow.models import Variable
  foo = Variable.get("foo")
  bar = Variable.get("bar", deserialize_json=True)
  baz = Variable.get("baz", default_var=None)
  ```



##### Storing Variables in Environment Variables

#### Branching

![](https://airflow.apache.org/docs/stable/_images/branch_note.png)

- 分叉，概念上类似if-else，workflow根据某些条件判断，选择之后的path，没有被选中的path上所有的tasks都skip

- e.g.

  ```python
  def branch_func(**kwargs):
      ti = kwargs['ti']
      xcom_value = int(ti.xcom_pull(task_ids='start_task'))
      if xcom_value >= 5:
          return 'continue_task'
      else:
          return 'stop_task'
  
  start_op = BashOperator(
      task_id='start_task',
      bash_command="echo 5",
      xcom_push=True,
      dag=dag)
  
  branch_op = BranchPythonOperator(
      task_id='branch_task',
      provide_context=True,
      python_callable=branch_func,
      dag=dag)
  
  continue_op = DummyOperator(task_id='continue_task', dag=dag)
  stop_op = DummyOperator(task_id='stop_task', dag=dag)
  
  start_op >> branch_op >> [continue_op, stop_op]
  ```

- 【Note】Note that using tasks with `depends_on_past=True` downstream from `BranchPythonOperator` is logically unsound as `skipped` status will invariably lead to block tasks that depend on their past successes

#### SubDAG

![](https://airflow.apache.org/docs/stable/_images/subdag_before.png)

![](https://airflow.apache.org/docs/stable/_images/subdag_after.png)

- 适用于重复的tasks优化，把重复的tasks打包成一个dag在function中返回

- 参见code, subdag最好单独定义一个文件，用function来**return dag object**，然后在main dag中import，避免subdag被当做单独的dag

  ```python
  from airflow.models import DAG
  from airflow.operators.dummy_operator import DummyOperator
  
  
  def subdag(parent_dag_name, child_dag_name, args):
      dag_subdag = DAG(
          dag_id='%s.%s' % (parent_dag_name, child_dag_name),
          default_args=args,
          schedule_interval="@daily",
      )
  
      for i in range(5):
          DummyOperator(
              task_id='%s-task-%s' % (child_dag_name, i + 1),
              default_args=args,
              dag=dag_subdag,
          )
  
      return dag_subdag
  ```

  ```python
  from airflow.example_dags.subdags.subdag import subdag
  from airflow.models import DAG
  from airflow.operators.dummy_operator import DummyOperator
  from airflow.operators.subdag_operator import SubDagOperator
  from airflow.utils.dates import days_ago
  
  DAG_NAME = 'example_subdag_operator'
  
  args = {
      'owner': 'Airflow',
      'start_date': days_ago(2),
  }
  
  dag = DAG(
      dag_id=DAG_NAME,
      default_args=args,
      schedule_interval="@once",
      tags=['example']
  )
  
  start = DummyOperator(
      task_id='start',
      dag=dag,
  )
  
  section_1 = SubDagOperator(
      task_id='section-1',
      subdag=subdag(DAG_NAME, 'section-1', args),
      dag=dag,
  )
  
  some_other_task = DummyOperator(
      task_id='some-other-task',
      dag=dag,
  )
  
  section_2 = SubDagOperator(
      task_id='section-2',
      subdag=subdag(DAG_NAME, 'section-2', args),
      dag=dag,
  )
  
  end = DummyOperator(
      task_id='end',
      dag=dag,
  )
  
  start >> section_1 >> some_other_task >> section_2 >> end
  ```

#### SLAs

-  Service Level Agreements, or time by which a task or DAG should have succeeded, can be set at a task level as a timedelta

#### Trigger Rules

- 常规trigger方法是，所有直系upstream的tasks都成功
- airflow也支持更复杂的逻辑
  - `all_success`: (default) all parents have succeeded
  - `all_failed`: all parents are in a `failed` or `upstream_failed` state
  - `all_done`: all parents are done with their execution
  - `one_failed`: fires as soon as at least one parent has failed, it does not wait for all parents to be done
  - `one_success`: fires as soon as at least one parent succeeds, it does not wait for all parents to be done
  - `none_failed`: all parents have not failed (`failed` or `upstream_failed`) i.e. all parents have succeeded or been skipped
  - `none_failed_or_skipped`: all parents have not failed (`failed` or `upstream_failed`) and at least one parent has succeeded.
  - `none_skipped`: no parent is in a `skipped` state, i.e. all parents are in a `success`, `failed`, or `upstream_failed` state
  - `dummy`: dependencies are just for show, trigger at will

#### Latest Run Only

- The `LatestOnlyOperator` skips all downstream tasks, if the time right now is not between its `execution_time` and the next scheduled `execution_time`
- 对time敏感的任务

#### Cluster Policy

#### Documentation & Note

#### Jinja Templating

#### Exceptions

#### Packaged DAGS

#### .airflowignore



## Scheduler

- monitoring tasks & DAGS
- 一个监视器+控制器: 监视并启动符合条件的任务，并委派任务给```Executer```去执行

- DAG -- Tasks [Operators]
- DAG[DagRun] -- Task [Instances]
- 启动```airflow shceduler```

## Executer

- 定义了task instances运行的机制
- 支持的executers
  - [Sequential Executor](https://airflow.apache.org/docs/stable/executor/sequential.html): 默认设置，每次只能执行一个任务，不用于生产环境
  - [Debug Executor](https://airflow.apache.org/docs/stable/executor/debug.html)
  - [Local Executor](https://airflow.apache.org/docs/stable/executor/local.html)
  - [Dask Executor](https://airflow.apache.org/docs/stable/executor/dask.html)
  - [Celery Executor](https://airflow.apache.org/docs/stable/executor/celery.html)【生产环境常见】
  - [Kubernetes Executor](https://airflow.apache.org/docs/stable/executor/kubernetes.html)【生产环境常见】
  - [Scaling Out with Mesos (community contributed)](https://airflow.apache.org/docs/stable/executor/mesos.html)

## DAG Runs

- 设定周期，可接受cron expression[str] 或者是datetime.timedelta obj
- [翻译工具: crontab.guru](https://crontab.guru/)
- 【note】run timestamp为2020-01-01的DAG run，trigger时间为soon after 2020-01-01T23:59，
  - 实际上是2020-01-02，但是execution_date仍为2020-01-01

| preset       | meaning                                                      | cron          |
| ------------ | ------------------------------------------------------------ | ------------- |
| `None`       | Don’t schedule, use for exclusively “externally triggered” DAGs |               |
| `@once`      | Schedule once and only once                                  |               |
| `@hourly`    | Run once an hour at the beginning of the hour                | `0 * * * *`   |
| `@daily`     | Run once a day at midnight                                   | `0 0 * * *`   |
| `@weekly`    | Run once a week at midnight on Sunday morning                | `0 0 * * 0`   |
| `@monthly`   | Run once a month at midnight of the first day of the month   | `0 0 1 * *`   |
| `@quarterly` | Run once a quarter at midnight on the first day              | `0 0 1 */3 *` |
| `@yearly`    | Run once a year at midnight of January 1                     | `0 0 1 1 *`   |



## CLI

- Pause a DAG: ```airflow pause [dag_id]```
- Resume a paused DAG ```airflow unpause [dag_id]```
- Trigger a DAG run ```airflow trigger_dag [dag_id]```
- Delete all DB recors realted to the specified DAG ```airflow delete_dag dag_id```
- run a single task instance ``` airflow run dag_id taks_id execution_date```
- Test a task instance, not checking dependencies or recording its state in the database```airflow test dag_id task_id execution_date```
- List all the DAGs ```airflow list_dags```同时也可以检查DAG
- Get the status of a dag run```airflow dag_state dag_id execution_date```
- Get the status of a task instance ```airflow task_state dag_id task_id execution_date```
- Run subsections of a DAG for a specified date range. If reset_dag_run option is used, backfill will first prompt users whether airflow should clear all the previous dag_run and task_instances within the backfill date range. If rerun_failed_tasks is used, backfill will auto re- run the previous failed task instances within the backfill date range.
  - ```airflow backfill dag_id -s start_date -e end_date```
- list the tasks within a DAG ``` airflow list_tasks dag_id```
- clear a set of task instance, as if they never ran ```airflow clear dag_id```



## [Tutorial](https://airflow.apache.org/docs/stable/tutorial.html)

### DAG的定义文件

- Airflow Python Script仅仅是用来声明DAG的结构
- 真正运行的task是在别处，在其他的worker上不同的时间段运行
- DAG声明文件不应该用于不同task之间的沟通数据参数传输【参考使用XCom】
- DAG Script 本身也不应该用于处理任何数据，scheduler需要反复短时运行这个script并记录反映变化，并决定下一步

```python
# ================================================================================================
from datetime import timedelta

# The DAG object; we'll need this to instantiate a DAG
from airflow import DAG
# Operators; we need this to operate!
from airflow.operators.bash_operator import BashOperator
from airflow.utils.dates import days_ago
# ================================================================================================

# ================================================================================================
# These args will get passed on to each operator
# You can override them on a per-task basis during operator initialization
default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': days_ago(2),
    'email': ['airflow@example.com'],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
    # 'queue': 'bash_queue',
    # 'pool': 'backfill',
    # 'priority_weight': 10,
    # 'end_date': datetime(2016, 1, 1),
    # 'wait_for_downstream': False,
    # 'dag': dag,
    # 'sla': timedelta(hours=2),
    # 'execution_timeout': timedelta(seconds=300),
    # 'on_failure_callback': some_function,
    # 'on_success_callback': some_other_function,
    # 'on_retry_callback': another_function,
    # 'sla_miss_callback': yet_another_function,
    # 'trigger_rule': 'all_success'
}
# ================================================================================================

# ================================================================================================
dag = DAG(
    'tutorial',
    default_args=default_args,
    description='A simple tutorial DAG',
    schedule_interval=timedelta(days=1),
)

# t1, t2 and t3 are examples of tasks created by instantiating operators
t1 = BashOperator(
    task_id='print_date',
    bash_command='date',
    dag=dag,
)

t2 = BashOperator(
    task_id='sleep',
    depends_on_past=False,
    bash_command='sleep 5',
    retries=3,
    dag=dag,
)
dag.doc_md = __doc__

t1.doc_md = """\
#### Task Documentation
You can document your task using the attributes `doc_md` (markdown),
`doc` (plain text), `doc_rst`, `doc_json`, `doc_yaml` which gets
rendered in the UI's Task Instance Details page.
![img](http://montcs.bloomu.edu/~bobmon/Semesters/2012-01/491/import%20soul.png)
"""
templated_command = """
{% for i in range(5) %}
    echo "{{ ds }}"
    echo "{{ macros.ds_add(ds, 7)}}"
    echo "{{ params.my_param }}"
{% endfor %}
"""

t3 = BashOperator(
    task_id='templated',
    depends_on_past=False,
    bash_command=templated_command,
    params={'my_param': 'Parameter I passed in'},
    dag=dag,
)
# ================================================================================================

# ================================================================================================
t1 >> [t2, t3]
# ================================================================================================
```



### 所需的Modules

```python
from datetime import timedelta

# The DAG object; we'll need this to instantiate a DAG
from airflow import DAG
# Operators; we need this to operate!
from airflow.operators.bash_operator import BashOperator
# from airflow.operators.python_operator import PythonOperator
from airflow.utils.dates import days_ago
```

- 更具需要引入modules



### Default Arguments

- 可以设置默认值，这样就不需要再在各个task中去详细声明
- task中可以再次声明去override default args

```python
# These args will get passed on to each operator
# You can override them on a per-task basis during operator initialization
default_args = {
    'owner': 'airflow', # dag的owner，会显示在WebUI上
    'depends_on_past': False, #是否依赖以前的DAG，以前的成功了，才能run今天的
    'start_date': days_ago(2),#
    'email': ['airflow@example.com'],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
    # 'queue': 'bash_queue',
    # 'pool': 'backfill',
    # 'priority_weight': 10,
    # 'end_date': datetime(2016, 1, 1),
    # 'wait_for_downstream': False,
    # 'dag': dag,
    # 'sla': timedelta(hours=2),
    # 'execution_timeout': timedelta(seconds=300),
    # 'on_failure_callback': some_function,
    # 'on_success_callback': some_other_function,
    # 'on_retry_callback': another_function,
    # 'sla_miss_callback': yet_another_function,
    # 'trigger_rule': 'all_success'
}
```



### Instantiate a DAG

- String define a unique dag_id
- 传入default_arg，也可以在内部声明并override

```python
dag = DAG(
    'tutorial', # dag_id
    default_args = default_args, #
    description='A simple tutorial DAG',
    schedule_interval=timedelta(days=1), # 定义DAG运行间隔时间
)
```



### Tasks

- 我们可以传入特定args比如```bash_command```或者是通用args比如```retries```
- 通用args比如```retries```可以从BashOperator继承到operator‘s constructor，不再需要专门声明
- Note：task#2中```retries=3```就override了```default_args```中的```retries=1```

```python
t1 = BashOperator(
    task_id='print_date',
    bash_command='date',
    dag=dag,
)

t2 = BashOperator(
    task_id='sleep',
    depends_on_past=False,
    bash_command='sleep 5',
    retries=3,
    dag=dag,
)
```



### Templating with Jinja

- Jinja为用户提供了参数和宏Macros的接口
- 用户也可以自定义参数，宏和模板

```python
templated_command = """
{% for i in range(5) %}
    echo "{{ ds }}"
    echo "{{ macros.ds_add(ds, 7)}}"
    echo "{{ params.my_param }}"
{% endfor %}
"""

t3 = BashOperator(
    task_id='templated',
    depends_on_past=False,
    bash_command=templated_command,
    params={'my_param': 'Parameter I passed in'},
    dag=dag,
)
```

- 可传入文件到bash_command - e.g. ```bash_command = templated_command.sh```
  - 【file location is relative to the directory containing the pipeline file (`tutorial.py` in this case)】
- Recommend - 有利于分离code/逻辑
- ```template_searchpath```可以定义template folder path

### Add DAG & Tasks Docs

```python
dag.doc_md = __doc__

t1.doc_md = """\
#### Task Documentation
You can document your task using the attributes `doc_md` (markdown),
`doc` (plain text), `doc_rst`, `doc_json`, `doc_yaml` which gets
rendered in the UI's Task Instance Details page.
![img](http://montcs.bloomu.edu/~bobmon/Semesters/2012-01/491/import%20soul.png)
"""
```

- DAG doc目前只支持markdown
- Task doc支持：plain text，markdown，reStructuredText，json，yaml

### Set up Dependencies

- 前文中已经声明了Tasks - t1 / t2 / t3；但是并没有声明这几个tasks的关系

```python
t1.set_downstream(t2) # t2 depend on t1, t1成功了才run
t2.set_upstream(t1) # 同上文等价
# The bit shift operator can also be
# used to chain operations:
t1 >> t2
# And the upstream dependency with the
# bit shift operator:
t2 << t1
# ============================================
# Chaining multiple dependencies becomes
# concise with the bit shift operator:
t1 >> t2 >> t3
# ============================================
# A list of tasks can also be set as
# dependencies. These operations
# all have the same effect:
t1.set_downstream([t2, t3])
t1 >> [t2, t3]
[t2, t3] << t1
```

- 如果发现有循环dependencies会报错

### Testing

- running script
  - dag file位置 - airflow.cfg - default - 【~/airflow/dags】

## Local安装与测试

- 安装与初始化

  ```shell
  pip install apache-airflow
  # 用下面这个
  pip install apache-airflow==1.10.12 \
   --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-1.10.12/constraints-3.7.txt"
  pip install apache-airflow==1.10.13 \
   --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-1.10.13/constraints-3.7.txt"
  
  # 初始化数据库
  airflow initdb
  
  # 重置数据库
  # airflow resetdb
  
  # 上面的命令默认在home目录下创建 airflow 文件夹和相关配置文件，可以使用以下命令来指定目录
  # export AIRFLOW_HOME={yourpath}/airflow
  
  # 学习时-几个配置的建议
  # 不要让airflow自动load examples
  # load_examples = False
  # fetch dag的时间可以设置短一点
  # min_serialized_dag_fetch_interval = 3
  # debug
  # fail_fast = Ture
  
  # 配置数据库等
  # vim airflow/airflow.cfg
  # 修改 sql_alchemy_conn
  
  # 守护进程运行webserver, 默认端口为8080，也可以通过`-p`来指定端口
  # airflow webserver -p XXXX
  airflow webserver -D  
  
  # 守护进程运行调度器     
  airflow scheduler -D
  
  ```

## Best Practices

### Writing a DAG

- Creating a Task

  - 把每次airflow task对待成database的transactions一样，不要产生未完成的结果
  - 需要考虑Airflow可以再次尝试运行失败的任务，task应该设计成每次结果都一样
    - 使用UPSERT而不是INSERT【e.g.第一次运行数据注入了一半，第二次又来insert会有重复的row】
    - 不建议使用```now()```这类func，尤其是用于计算时候，因为每次run会产生不同的结果

- Deleting a task:

  - 不要从DAG中delete task【？】会删除history，新建一个Updated DAG

- Communication:

  - 不要store file、config文件在Local filesystem上面

  - 例如：使用K8s或者是Celery Executer, 即使是同一个dag，下一个task也可能是在别的worker上跑的。

  - 如果下一个task依赖于上一个task产生的本地文件，那downstream task很有可能没有access

  - 解决方法

    - 如果用于小量数据传输，建议使用XCom

    - 如果需要pass large data，建议使用remote storage（e.g. S3）

    - 【场景】task1 处理完数据后，存储在S3，XCom把文件路径push出来，task2 pull拿到文件路径，读取数据，继续处理

    - 【note】run完了之后XCom不会被删除，而是保存在db里面，可以根据需要清理

      - ```python
        from airflow.models import DAG
        from airflow.utils.db import provide_session
        from airflow.models import XCom
        
        @provide_session
        def cleanup_xcom(context, session=None):
            dag_id = context['ti']['dag_id']
            session.query(XCom).filter(XCom.dag_id == dag_id).delete()
        
        dag = DAG( ...
            on_success_callback=cleanup_xcom,
        )
        ```

- Variables

  - 尽量在Operator's execute() 或者Jinja Template内使用
  - 使用variable会链接airflow DB 来fetch value，多了会降低parsing DAG的速度，也给DB增加负担
  - Default Airflow Parses DAG每秒，lots of connections。。。

- Code Out Side the tasks

  - 不要写【需要工作的】code在tasks之外
  - Default Airflow 每1秒Parses一次 所有DAG，尽量减轻负担

### Testing a DAG

- deploy DAG之前最好先test 
  - ```python your-dag-file.py```
- 已经deploy的可以用**CLI**```airflow list_dags```如果dag有问题也会报错【不建议】

## Tips & ken

- Error: ```got an unexpected keyword argument 'conf'```
  - 当设置```'provide_context':True```为true之后，func需要设置成take keywords arguments
  - ```def myfunc(**kwargs): pass```

## Reference

- [Airflow Official Doc V.1.10.13](https://airflow.apache.org/docs/stable/)
- [Airflow 使用及原理分析](https://zhuanlan.zhihu.com/p/90282578)
- [数据科学教程：如何使用Airflow调度数据科学工作流](https://segmentfault.com/a/1190000005835242)
- [Using SubDAGs in Airflow](https://www.astronomer.io/guides/subdags)
- [Airflow基础知识](http://longfei.leanote.com/post/airflow-basic)
- [Airflow DAG从编写到部署](http://longfei.leanote.com/post/airflow-basic)
- [Airflow 中文文档](https://www.kancloud.cn/luponu/airflow-doc-zh/content)
- [Airflow中利用xcom实现task间信息传递](https://kiwijia.work/2020/04/06/xcom/)
- [Trigger DAGs in Airflow](https://www.astronomer.io/guides/trigger-dag-operator)

## Airflow 2.0 Update Notes:
- Airflows升级中的一些细节变化：
  - 新增的包：basehook
  - 自带func的名字的变化：sub_dag -> subdag
  
- XCOM支持的数据格式: 
  - 要求数据必须是kv键值对，1.10.13 & 2.0.0有遇到报错：datetime类型value无法serialization，不能xcom_push
  - 默认只支持可json serialization的数据类型，
  - 如果需要用xcom推送dataframe, datetime等类型，需要set `enable_xcom_pickling = True`【Done/Updated】
  - 已更新 - 参见Airflow2.0 -  [DAG: pickle_test](https://airflow.sanofi.com/tree?dag_id=pickle_test)
  
- 一些支持功能的变化：2.0开始不再支持task overwrite【特例，本身这个用法并不常见】
  - for loop中，一个loop内create previous和after，然后set task stream
  - old version中可以：接下一个loop中可以重新定义task，并overwrite之前的task
  - 2.0开始不再允许这么做
  - 解决方案：`chain()`
  
- Context Manager [1.8新增]【并非新功能】

  - automatcilly assign new operator 给DAG [Ref](https://airflow.apache.org/docs/apache-airflow/stable/concepts.html#context-manager)

  - ```python
    with DAG('my_dag', start_date=datetime(2016, 1, 1)) as dag:
        op = DummyOperator('op')
    
    op.dag is dag # True
    ```
    
  - 可以避免漏写并省略一些code
  
- **TaskFlow API [Airflow2.0新功能]**

  - 参考`example_taskflow_api` [Link](https://airflow.sanofi.com/tree?dag_id=example_taskflow_api)
  - 不再需要用户声明task之间的关系，会自动增加task 之间的 dependency，data push using XCOM backend
  - 总结：在线性task flow的情况下，比较好用

- **DAG decorator [Airflow2.0新功能]**

  - Use func to return and create a dag
  - 旧有的写法依旧可用
  - 参考`example_dag_operator` [Link](https://airflow.sanofi.com/code?dag_id=example_dag_decorator)
  - 参考`example_dag_operator2` [Link](https://airflow.sanofi.com/code?dag_id=example_dag_operator2)
  - 参考`example_dag_operator3` [Link](https://airflow.sanofi.com/code?dag_id=example_dag_operator3)

- **Python Task decorator [Airflow2.0]**

  - ```python
    with DAG('my_dag', start_date=datetime(2020, 5, 15)) as dag:
        @dag.task
        def hello_world():
            print('hello world!')
    
    
        # Also...
        from airflow.decorators import task
    
    
        @task
        def hello_name(name: str):
            print(f'hello {name}!')
    
    
        hello_name('Airflow users')
    ```

  - 参考code`example_dag_operator3` [Link](https://airflow.sanofi.com/code?dag_id=example_dag_operator3)

  - Task 装饰器，同时在推送XCOM value时候，会推送到一个single xcom value中，如果希望能够展开，设置`multiple_outputs=True` to unroll dict, list, tuple into separate XCom values

  - 被Task decorator装饰过的func，在同一个DAG中可以多次引用，值得注意的是，会自动生成unique task_id。

  - ```python
    with DAG('my_dag', start_date=datetime(2020, 5, 15)) as dag:
    
      @dag.task
      def update_user(user_id: int):
        ...
    
      # Avoid generating this list dynamically to keep DAG topology stable between DAG runs
      for user_id in user_ids:
        update_user(user_id)
    
      # This will generate an operator for each user_id
    ```

  - `[update_user,update_user__1,update_user__2, ...] `

  - 注意，task_id的生成是seq的，如果已有重复命名的task_id,则命名会顺序后移，会造成confuse，慎重使用

- Access current context

  - ```python
    from airflow.operators.python import task, get_current_context
    
    @task
    def my_task():
        context = get_current_context()
        ti = context["ti"]
    ```

  - 使用method `get_current_context()`能够方便的结合@task decorator拿到current context
  
  - 参考code`example_dag_operator3` [Link](https://airflow.sanofi.com/code?dag_id=example_dag_operator3)
  
- 引用路径的修改：

  - e.g. [BaseHook](https://github.com/apache/airflow/blob/master/UPDATING.md#changes-to-import-paths) & [OracleHook](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/hooks/oracle_hook/index.html) & [PythonOperator](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/operators/python_operator/index.html?highlight=airflow%20operators%20python_operator#module-airflow.operators.python_operator)
  - BaseHook: [airflow.hooks.base_hook.BaseHook] >> [airflow.hooks.base.BaseHook]
  - OracleHook: [airflow.hooks.oracle_hook] >> [airflow.providers.oracle.hooks.oracle]
  - PythonOperator: [from airflow.operators.python_operator import PythonOperator] >> [from airflow.operators.python import PythonOperator]
  - 更新了新的import path， older version 仍然能用，之后版本可能会不再支持
  - 参考`airflow_2.0_db_connection_test` [Link](https://airflow.sanofi.com/tree?dag_id=airflow_2.0_db_connection_test)
  - 参考`airflow_2.0_db_connection_test_v2`[Link](https://airflow.sanofi.com/tree?dag_id=airflow_2.0_db_connection_test_v2)
