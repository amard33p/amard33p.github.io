---
title: "Celery: Seed to Sprout"
layout: post
excerpt: "Understanding Python Celery from scratch to build a Django REST Framework API. "
tags:
  - celery
  - django-rest-framework
---
We will learn to create a Celery task queue consisting of a chain of long-running tasks from scratch. Finally, we will build an API using Django REST Framework which accepts task inputs and then runs the task queue.

## Setup

* Celery - `pip install celery[redis]`
* RabbitMQ - `docker run -d -p 5672:5672 rabbitmq`
* Redis - `docker run -d -p 6379:6379 redis`  
* Clone the tutorial repo - <https://github.com/amard33p/celery101>

## Environment

* Ubuntu: 18.04
* Python: 3.6.5
* Celery: 4.3.0
* Redis server: 5.0.7
* RabbitMQ version: 3.8.1
* Docker: 17.12.1-ce

    **Considerations:**

    * Why not use RPC as backend instead of installing an additional component - Redis?  
        Due to [this issue](https://github.com/celery/celery/issues/4084) I found RPC to incorrectly report task status.
    * Why not use Redis as both broker and backend?  
        From [the docs](https://docs.celeryproject.org/en/latest/faq.html#do-i-have-to-use-amqp-rabbitmq),  
        > Redis as a broker won’t perform as well as an AMQP broker, but the combination RabbitMQ as broker and Redis as a result store is commonly used. If you have strict reliability requirements you’re encouraged to use RabbitMQ or another AMQP broker.

## Index
  - [Day-0: Baby Steps](#day-0-baby-steps)
  - [Day-1: Structuring celery projects](#day-1-structuring-celery-projects)
  - [Day-2: Importing multiple tasks](#day-2-importing-multiple-tasks)
  - [Day-3: Structuring tasks in sub directories](#day-3-structuring-tasks-in-sub-directories)
  - [Day-4: Designing workflows](#day-4-designing-workflows)
  - [Day-5: Creating a base class for tasks](#day-5-creating-a-base-class-for-tasks)
  - [Day-6: Getting status of workflow tasks](#day-6-getting-status-of-workflow-tasks)
  - [Day-7: Passing task arguments to a workflow](#day-7-passing-task-arguments-to-a-workflow)
  - [Creating an API for Celery using DRF](#creating-an-api-for-celery-using-drf)

For each section, the supporting scripts are in the dayN directory of the repo.

### Day-0: Baby Steps

A barebones celery task using rabbitmq as broker and rpc as backend.
```bash
cd celery101/day0
cat taskrunner.py
```

```python
import time
from celery import Celery

app = Celery(
    'taskrunner',
    broker='amqp://',
    backend='rpc'
)


@app.task
def add(x, y):
    time.sleep(5)
    return x + y

if __name__ == '__main__':
    app.start()
```
To run this task, open 2 terminals.
* On Term 1 - Start the celery worker process.  
  ```bash
  cd celery101/day0
  celery -A taskrunner worker -l info
  ```
* On Term 2 - Run `python3` 

    ```python
    >>> from taskrunner import add

    >>> result = add.delay(4, 4)
    >>> result.ready() # Is task finished?
    #=> False
    # After 5 seconds
    >>> result.ready()
    #=> True
    >>> result.get() # Task result
    #=> 8

    ```
**Note:** While task.delay() is simpler, [task.apply_async() is much more powerful](https://docs.celeryproject.org/en/latest/userguide/calling.html).  
  ```
  task.delay(arg1, arg2, kwarg1='x', kwarg2='y')
  task.apply_async(args=[arg1, arg2], kwargs={'kwarg1': 'x', 'kwarg2': 'y'})
  ```

### Day-1: Structuring celery projects

The recommended way is to create a module out of your celery project and separate the celery instance and tasks.
```bash
cd celery101/day1
tree
.
└── taskrunner
    ├── celery.py
    ├── __init__.py
    └── tasks.py
```

Normally, celery tasks have 3 states - pending, finished, or waiting to be retried. Since our tasks will be a chain of long-running tasks we need celery to report when a particular task has started. So we need to add `task_track_started=True,` in celery.py.

**Note:** Tasks need to be bound to the app instance. So after importing app we need to decorate tasks with `@app.task`

To run this again open 2 terminals.
* On Term 1 - Start the celery worker process.  
  ```bash
  cd celery101/day0
  celery -A taskrunner worker -l info
  ```
* On Term 2 - Run `python3` 

    ```python
    >>> from taskrunner.tasks import add

    >>> result = add.apply_async((2, 2))
    >>> result.get() # Task result
    #=> 8

    ```

### Day-2: Importing multiple tasks

To import multiple task scripts, simply add it to the include parameter while instantiating the Celery object.  
`include=['taskrunner.task1', 'taskrunner.task2']`

### Day-3: Structuring tasks in sub directories

Now what if we wanted our tasks to be neatly organized in subdirectories? For e.g  

```bash
.
├── celery.py
├── __init__.py
└── tasks
    ├── bar
    │   ├── __init__.py
    │   ├── task1.py
    │   └── task2.py
    ├── foo
    │   ├── __init__.py
    │   ├── task1.py
    │   └── task2.py
    └── __init__.py
```

Instead of modifying celery.py every time we add a new task, we can write a simple function which looks for task scripts under tasks directory and automatically imports them.

```python
def taskDetector(task_root):
    tasks = []
    for root, dirs, files in os.walk(task_root):
        for filename in files:
            if filename != '__init__.py' and filename.endswith('.py'):
                task = os.path.join(root, filename)\
                    .replace('/', '.')\
                    .replace('.py', '')
                tasks.append(task)
    return tasks

app = Celery(APP_NAME,
             broker='amqp://',
             backend='redis://localhost:6379/0',
             include=taskDetector(task_root))
```

### Day-4: Designing workflows

To run multiple tasks serially/in-parallel we need to run the task using the task's signature method.

E.g: To calculate (4 / 2 * 3 + 6 - 2)  
`Workflow1 = (div.s(4, 2) | mul.s(3) | add.s(6) | sub.s(2)).apply_async()`  
OR  
`Workflow1 = (div.s(4, 2) | mul.s(3) | add.s(6) | sub.s(2))()`  
`print(Workflow1.get())`

For parallel execution we will use `group`  
`group(div.s(2, 2) , mul.s(4, 4) , add.s(3, 2) , sub.s(8, 8))()`

**Note:** `.s()` is used when the task accepts args/kwargs and passes result to the next task in the chain. 
`.si()` is used when all required args/kwargs are passed to the task signature directly.

Refer to [day4/wfutil.py](https://github.com/amard33p/celery101/blob/master/day4/wfutil.py) for more examples.  
To run the workflows, start the worker and run `python3 wfutil.py` in another shell.

### Day-5: Creating a base class for tasks

In real world scenario, our tasks will perform repetitive actions. For e.g:
* Logging when task has started
* Logging success/failure
* Doing something while starting/ending the task

It makes sense to have a base class perform these actions and then have our tasks inherit from the base class.

```python
class CustomBaseTask(celery.Task):  # Must extend celery.Task

    def __call__(self, *args, **kwargs):
        # What to do before starting task
        logger.info("Starting to run")
        logger.info('Task-{0!r} with id-{1!r} STARTED with args={2!r} and kwargs={3!r} '.format(self.name, self.request.id, args, kwargs))
        return self.run(*args, **kwargs)    # Note the use of run and not super as suggested in some examples online

    def after_return(self, status, retval, task_id, args, kwargs, einfo):
        #exit point of the task whatever is the state
        logger.info("Ending run")

    def on_success(self, retval, task_id, args, kwargs):
        # What to do if task succeeds
        logger.info('Task-{0!r} with id-{1!r} SUCCEEDED'.format(self.name, task_id))

    def on_failure(self, exc, task_id, args, kwargs, einfo):
        # What to do if task fails
        logger.info('Task-{0!r} with id-{1!r} FAILED with err-{2!r}'.format(self.name, task_id, exc))

# Finally inherit this class in your tasks
@app.task(base=CustomBaseTask)
# task...
```

I have also named the tasks to identify the tasks easily in the logs. This can also be helpful to run tasks by name using [send_task.](https://stackoverflow.com/questions/30308259/how-to-invoke-celery-task-by-name)

### Day-6: Getting status of workflow tasks

* Let's consider the same workflow from Day-4:  
  `Workflow1 = (div.s(4, 2) | mul.s(3) | add.s(6) | sub.s(2))()` 
* Now `Workflow1` has started. Since the workflow contains long running tasks, we need to be able to query the status of each task.
* For this first we need the task_ids of each task in `Workflow1`.  
  `Workflow1` will reference the last task in the chain (sub). We need to traverse up the chain using a generator to access all tasks in the  workflow.  

  ```python
  def nodes(node):
    while node.parent:
        yield node
        node = node.parent
    yield node

  Workflow1_tasklist = [task.id for task in nodes(Workflow1)]
  ```
* Note that the list is in reversed order as the workflow. To query the task status, we create an `AsyncResult` object using the task_ids,
  and then query its state.

  ```python
  from celery.result import AsyncResult

  for task_id in reversed(Workflow_1_tasklist):
      result = AsyncResult(task_id, app=app)
      print({result.id, result.state})
  ```

#### Using Flower

Getting the status of a task is way simpler with Flower.
* `pip install flower`
* Open a new terminal, cd to celery project directory and run  
  `flower -A taskrunner --broker=amqp://guest:guest@localhost:5672//`
* Now simply use Flower REST API to get task status using task id.  
  `curl -X GET http://localhost:5555/api/task/info/TASK_ID`
  ```
  {
    "uuid": "035504e5-89d6-43f4-bc8a-b5e0588be14d",
    "name": "diff-of-two-numbers",
    "state": "SUCCESS",
    "received": 1574678654.7077875,
    "sent": null,
    "started": 1574678654.7103007,
    "rejected": null,
    "succeeded": 1574678659.7206867,
    "failed": null,
    "retried": null,
    "revoked": null,
    "args": "[8, 8]",
    "kwargs": "{}",
    "eta": null,
    "expires": null,
    "retries": 0,
    "worker": "celery@ubuntu-host",
    "result": "0",
    "exception": null,
    "timestamp": 1574678659.7206867,
    "runtime": 5.009667221456766,
    "traceback": null,
    "exchange": null,
    "routing_key": null,
    "clock": 797,
    "client": null,
    "root": "31a9d888-6426-4034-96d7-b31a7a7a5a76",
    "root_id": "31a9d888-6426-4034-96d7-b31a7a7a5a76",
    "parent": "7f69b88f-7531-4e0f-b02d-5931fc2b3588",
    "parent_id": "7f69b88f-7531-4e0f-b02d-5931fc2b3588",
    "children": []
  }
  ```

### Day-7: Passing task arguments to a workflow

* Let's consider the same workflow from Day-6:  
  `Workflow1 = (div.s(4, 2) | mul.s(3) | add.s(6) | sub.s(2))()` 
* However, instead of passing arguments directly to the task signatures, we will supply it at runtime.
* For e.g, this is the argument data that needs to be supplied:  
  `job_data = {'div' : [4, 2], 'mul' : [3], 'add' : [6], 'sub' : [2]}`
* If we pass `job_data` as kwargs to `apply_async` it will only be passed to the first task in the chain. So we will pass the arguments as kwargs to individual task signatures.  
  
  ```python
  from celery import chain

  workflow1_tasklist = [div, mul, add, sub]
  job_data = {'div' : [4, 2], 'mul' : [3], 'add' : [6], 'sub' : [2]}
  
  # Pass job_data as kwargs to each task in task list
  def workflow_generator(task_list, job_data):
      result = tuple(getattr(task, 's')(job_data = job_data) for task in task_list)
      return chain(*result)
  
  Workflow_1 = workflow_generator(workflow1_tasklist, job_data)
  ```
* Now we need to modify our tasks to accept the arguments.  

  ```python
  @app.task(base=CustomBaseTask, name='division-of-two-numbers')
  def div(*args, **kwargs):
      time.sleep(5)
      x, y = kwargs['job_data']['div'][0], kwargs['job_data']['div'][1]
      return x / y
  
  @app.task(base=CustomBaseTask, name='product-of-two-numbers')
  def mul(*args, **kwargs):
      time.sleep(5)
      x = args[0] # return value of div()
      y = kwargs['job_data']['mul'][0]
      return x * y
  ```
* The output of `div` is passed as args to `mul` because in a chain the return value of a task is sent only as args to the next task.  
  Our code would be more readable if the tasks get all their inputs from kwargs only. So we can add logic in our [BaseTask class][1] which checks if args[0] contains a dict and if true, moves it to kwargs.  

  ```python
  def __call__(self, *args, **kwargs):
  if len(args) == 1 and isinstance(args[0], dict):
      kwargs.update(args[0])
      args = ()
      return self.run(*args, **kwargs)
    ```
* Now `div` can `return {'div_result' : x / y}` and `mul` can get `x = kwargs['div_result']`. 
* Note that the reassignment of args to kwargs happens during `__call__` so the task still needs to accept `args=None` as params. To remove this requirement we may use [Celery-Signals.][2]


[1]: https://github.com/amard33p/celery101/blob/master/day7/taskrunner/tasks/baseTask.py#L10-L13
[2]: https://docs.celeryproject.org/en/latest/userguide/signals.html


### Creating an API for Celery using DRF

**Prerequisite Knowledge:** Django REST Framework, Docker

We are in a good shape now to create a simple API around Celery for running workflows.  
First, bring up the docker stack: `docker-compose up -d`  

Our application can run 3 testcases. Here are the workflows:  

    RunTC01Workflow = [div, mul, add, sub]  
    RunTC02Workflow = [div, mul]  
    RunTC03Workflow = [add, sub]

The user chooses the testcase and supplies the testcase data along with it.  
`curl -d 'testcase=ESG-TC02' -d 'job_data={"div" : [4, 2], "mul" : [3]}' -X POST http://localhost:8000`

This will run the workflow and return the task ids in the workflow.

```python
def post(self, request):
    testcase = request.POST.get('testcase')
    job_data = json.loads(request.POST.get('job_data'))
    task_ids = wfutil.run_workflow(testcase, job_data)
    return Response(task_ids)
```

The user can then query the status of the workflow using the task ids.  
`curl -d 'task_list=["e2f970f6-52ed-48fe-ad4c-93c31d312f92","303c37f8-f079-4bc7-a0b4-b311c6fdb65e"]' -X GET http://localhost:8000`

```python
def get(self, request, format=None):
    task_list = request.POST.get('task_list')
    return Response(wfutil.get_wf_status(json.loads(task_list)))
```


---

_References:_  
- <https://docs.celeryproject.org/en/latest/getting-started/first-steps-with-celery.html>  
- <https://docs.celeryproject.org/en/latest/userguide/tasks.html>  
- <https://docs.celeryproject.org/en/latest/userguide/canvas.html>  
- <https://stackoverflow.com/questions/26058156/celery-get-list-of-registered-tasks>  
- <https://stackoverflow.com/questions/7149074/deleting-all-pending-tasks-in-celery-rabbitmq>  
- <https://stackoverflow.com/questions/17702578/how-to-structure-celery-tasks>  
- <https://stackoverflow.com/questions/22078549/celery-discover-tasks-in-files-with-other-filenames>  
- <https://stackoverflow.com/questions/41861882/parallel-and-sequential-execution-of-tasks-using-celery>  
- <https://stackoverflow.com/questions/29547695/celery-access-all-previous-results-in-a-chain>  
- <https://stackoverflow.com/questions/18872854/getting-task-id-inside-a-celery-task>  
- <https://stackoverflow.com/questions/14968265/celery-task-chain-and-accessing-kwargs>  
- <https://stackoverflow.com/questions/54899320/what-is-the-meaning-of-bind-true-keyword-in-celery>  
- <https://stackoverflow.com/questions/9731435/retry-celery-tasks-with-exponential-back-off>  
- <https://www.toptal.com/python/orchestrating-celery-python-background-jobs>  
- <https://www.distributedpython.com/2018/09/04/error-handling-retry/>  
- <https://www.distributedpython.com/2018/08/28/celery-logging/>  
- <https://pawelzny.com/python/celery/2017/08/14/celery-4-tasks-best-practices/>  
- <https://medium.com/@taylorhughes/three-quick-tips-from-two-years-with-celery-c05ff9d7f9eb>  
- <https://github.com/celery/celery/pull/5652>  
- <https://www.distributedpython.com/2018/06/12/celery-django-docker/>  
- <https://coderbook.com/@marcus/best-practices-for-setting-up-celery-with-django/>  
- <https://github.com/xlwings/python-celery-dockerize-celery-django>
