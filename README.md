# 源码解析

## 一、基本主入口

### 1. 核心入口

在项目根目录下的`setup.py`文件中我们可以看到定义了很多我们依赖的框架等，当然
最核心的主要是其中的`do_setup`函数中的`scripts`的内容，其中定义了我们为什么
我们可以在系统上下文件直接通过`airflow`启动相关的服务。  

我们根据这个路径的指令打开`airflow/bin/airflow.py`文件，我们可以看到仅仅只有
若干的代码，但是其中就是关键的核心所在，这里依赖`argcomplete`实现了我们通过
命令行进行参数输入时的自动提示补全功能，当然我们需要注意最后这行代码：  

```python
args.func(args)
```

我们直接跟这个`func`会发现无法进行跟踪，但是启动具体的服务的核心就在这里了，所
以我们必须研究出具体如何启动对应的服务，这时候我们可以发现其中有`CLIFactory`是
我们定义的一个类，这个时候我们跟踪进去。当然其中主要还是参数的各类定义和解析，
但是我们可以看到类属性中的`subparsers`中定义了`func`的属性，同时这也是指向了很
多其他启动函数的地方，所以这里是通过对命令函参数的解析组合到这里的命令参数从而
识别到具体需要启动的函数，从而通过`func`直接启动对应服务。  

```python
    subparsers = (
        {
            'func': backfill,
            'help': "Run subsections of a DAG for a specified date range. "
                    "If reset_dag_run option is used,"
                    " backfill will first prompt users whether airflow "
                    "should clear all the previous dag_run and task_instances "
                    "within the backfill date range. "
                    "If rerun_failed_tasks is used, backfill "
                    "will auto re-run the previous failed task instances"
                    " within the backfill date range.",
            'args': (
                'dag_id', 'task_regex', 'start_date', 'end_date',
                'mark_success', 'local', 'donot_pickle', 'yes',
                'bf_ignore_dependencies', 'bf_ignore_first_depends_on_past',
                'subdir', 'pool', 'delay_on_limit', 'dry_run', 'verbose', 'conf',
                'reset_dag_run', 'rerun_failed_tasks', 'run_backwards'
            )
        }
```

而`func`如何被返回则是由类方法中的以下代码生成：  

```python
        for sub in subparser_list:
            sub = cls.subparsers_dict[sub]
            sp = subparsers.add_parser(sub['func'].__name__, help=sub['help'])
            for arg in sub['args']:
                if 'dag_id' in arg and dag_parser:
                    continue
                arg = cls.args[arg]
                kwargs = {
                    f: v
                    for f, v in vars(arg).items() if f != 'flags' and v}
                sp.add_argument(*arg.flags, **kwargs)
            sp.set_defaults(func=sub['func'])
        return parser
```

后续我们就可以根据不同命令寻找对应的启动函数进行进一步的解析了。  

### 2. WebServer入口

这里根据实际服务的搭建过程从上至下逐一进行各类服务的介绍，首先是`WebServer`服务我们在`airflow/bin/cli.py`文件中
查找到`webserver`函数通过其中的解读:  

```python
        run_args = [
            'gunicorn',
            '-w', str(num_workers),
            '-k', str(args.workerclass),
            '-t', str(worker_timeout),
            '-b', args.hostname + ':' + str(args.port),
            '-n', 'airflow-webserver',
            '-p', str(pid),
            '-c', 'python:airflow.www.gunicorn_config',
        ]
```

我们可以发现是通过`gunicorn`进行进行服务的启动的，当然我们还需要寻找实际的主入口类，这里就需要我们接着查看后续的代
码了：  

```python
webserver_module = 'www_rbac' if settings.RBAC else 'www'
run_args += ["airflow." + webserver_module + ".app:cached_app()"]
```

通过这部分我们就可以实际的入口根据实际的配置将会进行选择，这里我们以未开启RBAC，实际的入口将会是`airflow/www/cli.py`
中的`cached_app`函数，此时我们就可以进入到实际的主入口了，后续关于`webserver`更详细的剖析将会在其他章节进行概述。  

### 3. backfill入口

为了防止有人对该功能存在疑惑，这里简单解释下该功能就是用于进行任务回填。采用简单的术语描述就是当然既定的任务错失了对
应的执行时间后，我们可以通过该方法指定对应时间使服务重新进行执行，特别适用于我们进行DAG测试开发和服务器出现故障后发生
部分时间期间任务没有进行执行的情况下。  

我们跟踪到`airflow/www/clu.py`中的`backfill`函数，可以看到具体的Dag任务通过如下代码进行查找：  

```python
dag = dag or get_dag(args)
```

这里可以直接根据既定的Dag任务也可以根据Dag_id进行查询获取，如果存在参数`task_regex`将会根据正则表达式进一步的查询具体
需要的task任务，最后则是根据`dry_run`参数采用不同的方式进行对应任务的执行：  

```python
for task in dag.tasks:
    print("Task {0}".format(task.task_id))
    ti = TaskInstance(task, args.start_date)
    ti.dry_run()     
```

如果`dry_run`不为True则执行如下函数：  

```python
dag.run(
    start_date=args.start_date,
    end_date=args.end_date,
    mark_success=args.mark_success,
     local=args.local,
     donot_pickle=(args.donot_pickle or
                   conf.getboolean('core', 'donot_pickle')),
     ignore_first_depends_on_past=args.ignore_first_depends_on_past,
     ignore_task_deps=args.ignore_dependencies,
     pool=args.pool,
     delay_on_limit_secs=args.delay_on_limit,
     verbose=args.verbose,
     conf=run_conf,
     rerun_failed_tasks=args.rerun_failed_tasks,
     run_backwards=args.run_backwards
)
```

后续更具体的则需要深入了解Dag的运行机制了。  

## 二、WebServer剖析  

由于其框架采用了[Flask](https://dormousehole.readthedocs.io/en/latest/)以及插件[Flask-admin](https://flask-admin.readthedocs.io/en/latest/)，故如果希望能够更好的了解其代码的含义，可能
需要读者具备一定的该类框架基础，否则可能会存在很多不理解的地方。  

首先我们根据之前的入口描述找到`airflow/www/app.py`文件，查看对应的方法源码：` 

```python
def cached_app(config=None, testing=False):
    global app
    if not app:
        base_url = urlparse(conf.get('webserver', 'base_url'))[2]
        if not base_url or base_url == '/':
            base_url = ""

        app = create_app(config, testing)
        app = DispatcherMiddleware(root_app, {base_url: app})
    return app
```

可以看到最终是用`create_app`进行创建，通过对函数的剖析我们可以发现其中进行了相关页面视图的注册，同时通过继续往下的阅读
可以发现这么一行代码：  

```python
import airflow.www.api.experimental.endpoints as e
# required for testing purposes otherwise the module retains
# a link to the default_auth
if app.config['TESTING']:
    six.moves.reload_module(e)

app.register_blueprint(e.api_experimental, url_prefix='/api/experimental')
```

可以发现这里将`airflow/www/api/experimental/endpoints.py`加载进来了，并在在后续进行的注册。那么我们打开到这个对应的文件
继续月度就可以发现，实际服务调用的后台都是该文件提供了对应的服务，对于熟悉框架的读者来说就可以通过对其进行修改从而提供扩
展自己需要的接口功能功能或者扩展自己的页面。  

对于`views.py`文件来说，为了保证快速开发其中对于个性化页面通过`Airflow`类进行对应页面，而对于一些通用的列表等界面则需要
通过文件往下继续查看比如以下几个视图：  

* PoolModelView  
* SlaMissModelView  
* ChartModelView

当然还有其他更多的页面，这些页面在`app.py`中都是单独进行手动注册的：  

```python
av(vs.KnownEventView(
    models.KnownEvent,
    Session, name="Known Events", category="Data Profiling"))
av(vs.SlaMissModelView(
    models.SlaMiss,
    Session, name="SLA Misses", category="Browse"))
av(vs.TaskInstanceModelView(models.TaskInstance,
    Session, name="Task Instances", category="Browse"))
av(vs.LogModelView(
    models.Log, Session, name="Logs", category="Browse"))
av(vs.JobModelView(
    jobs.BaseJob, Session, name="Jobs", category="Browse"))
```

还有部分没有截取，但是通过以上我们可以看到通过`admin.add_view`将页面进行加载，其中也传入的对应的数据模型以及会话相关
的对象。  
