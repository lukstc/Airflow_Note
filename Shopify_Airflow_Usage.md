# Note Shopify Airflow Usage Best Practice



## Issue#1 - 对线上DAG文件的访问速度限制了airflow平台的performance

### Pain Point
- 使用GCP平台的[GCPFuse](https://github.com/GoogleCloudPlatform/gcsfuse)去统一存贮和管理所有的DAG，优势是能够统一管理方便访问。
- 但是在实际使用当中，他们的DAG数量多达1w+
- 每个时间点上在运行的tasks超过400个，每个运行的pod都会seperately地去GCP读取dag相关信息。
- 多对一的读取，导致这一步速度很慢。

### Solution

- k8s集群内部使用一个NFS(Network file system) Server
- 多对多读写工作
- user只需要维护GCP上的dag code
- 使用脚本对nfs内部code和GCP上的code进行同步
- 脚本来设定配置哪些部分是需要同步的
- 也可以利用Google Cloud Platform的身份管理来确定user可以上传控制的环境（dev，staging，prod）

## Issue#2 - 累积的Metadata会导致airflow性能下降

### Pain Point

- airflow的dag运行，log等都是存贮在airflow绑定的数据库中
- 当有大量的DAG需要在生成当中使用的时候，每天都会有大量的记录数据塞到数据库中
- 久而久之，这些数据会使我们的数据库变慢，也使得我们的web UI响应迟钝，之后升级迁移也会花费更多的时间

### Solution

- 配置retention policy，清理超过28天的数据
- Shopify团队写了个DAG来做这个: "implemented a simple DAG which uses ORM (object–relational mapping) queries within a PythonOperator to delete rows from any tables containing historical data (DagRuns, TaskInstances, Logs, TaskRetries, etc)."
- 这个retention时间需要结合使用者实际需求来定
- airflow 2.3中会有`db clean`command来做数据库清理工作



## Issue#3 - DAGs难和作者或者团队直接联系起来

### Pain Point

- DAG来源不一，并非从单个repo中fetch过来，可以从不同项目过来或者部署过程中动态的生成新的job
- Airflow中难以把DAG和正确的user或者team联系起来。当需要报错或者联系user的时候，容易找不到origin

### Solution

- 【这部分可能和shopify的k8s以及airflow部署的方式有关】
- user提交DAG的同时，也需要提交一份YAML file，用于register namespace for their DAGs
- 同时文件中也标注了repo的来源和owner



## Issue#4 - DAG Author的权限管理

### Pain Point

dag作者有较大的权限， airflow也是一个中心化的数据平台，连接了不同的系统也有不同的访问权限；

Airflow平台的管理员也不可能在每个DAG投入生产前去逐一review每一个的code；

### Solution

实践DAG Policy，需要读取DAG配置并确保清晰声明了namespace，owner等信息，否则就会报错

- DAG ID 前缀必须为namespace
- DAG中的tasks只能在指定的celery queue中排队
- DAG中的task只能在指定的任务池pool中运行
- DAG中的KubernetesPodOperators只能在指定的POD中启动，避免在别的namespace中运行
- DAG中的tasks，只能在指定external k8s clusters中lunchpod

能够追踪和控制DAG，同时能够增加一些限制控制DAG所能做的事务



## Issue#5 - 难以保证持续稳定的任务负载

### Pain Point

依据User习惯写出来的DAG时间配置，在scale规模化之后往往会有不少其他问题

使用`timedelta=1hour`的写法，虽然在user的角度看来没有什么问题。但是在实际操作当中，当有user使用python 在parse-time generates很多dag，那么所有的DAGRuns都会在同一时间被创建出来。短时间内Airflow Scheduler负载压力很大，外部的资源压力也会很大。同时过了相同的`schedule_interval`之后，所有任务会再次被同时调起，会造成和之前一样的问题。最终会导致更长的执行时间。

但是使用crontab based schedule也有另一种问题。user去写DAG的时候往往会习惯写一种简单明了方便自己读写的方式，因此往往会写成，整点读写，凌晨整点等方式。有些时候业务是业务需要导致，但更多的时候只是因为user希望能够使用一个自己易于读写的方式去表达。

### Solution

一种解决这两个问题的方式，对于所有自动生成的DAG（大多数都是这种）使用一个确定性的(deterministically)随机时间间隔。基于一同一个seed（dag-id的哈希值）

这种随机interval时间的方式，有助于我们Evenly的分发我们的task



## Issue#6 - 其他的bottleneck

### Pools

使用pools有助于解决resource contention 资源争夺的问题。当给定数量的tasks时候，可以用来限制并发数量。这在high traffic的时候很适用。虽然pools能解决一些问题，但是对admin的管理也提出了一些新的要求，因为Airflow admin才有权限处理管理pools

### Celery

tasks可能根据需要放在不同的env中运行，例如不同的权限，依赖，资源等。可以创建更多的queues用于不同的tasks的提交。同时可以配置不同的works去处理不同的任务队列。

一个task也可以指定分配给不同的队列，可以在operator中声明。利用pools，priority-weight，queues可以有效的减少资源挤占情况

## Take Away Msg

- A combination of GCS and NFS allows for both performant and easy to use file management.
- Metadata retention policies can reduce degradation of Airflow performance.
- A centralized metadata repository can be used to track DAG origins and ownership.
- DAG Policies are great for enforcing standards and limitations on jobs.
- Standardized schedule generation can reduce or eliminate bursts in traffic.
- Airflow provides multiple mechanisms for managing resource contention.



## Ref

- Source: [Shopify - Lessons Learned From Running Apache Airflow at Scale](https://shopify.engineering/lessons-learned-apache-airflow-scale)

- Author: Shopify - Megan Parker

- 总结翻译：xukun.liu@outlook.com
