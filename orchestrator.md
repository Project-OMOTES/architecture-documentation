# Orchestrator component

Options:
- Airflow (Works but heavy architecture)
- Argo (Works only for k8s but seems to do what we want. Arbitrary code/scripts is convoluted it can only schedule containers)
- Celery: Only real options to try out. Have not been able to find any big alternatives except for dramatiq, see below.

Quick non-options:
- Keboola (No selfhosting so not an option. No further analysis)
- Kubeflow (Is meant for ML pipelines, not our use case)
- Crossplane (Is meant to manage cloud resources through a k8s cluster. Does not allow for workflows.)
- Luigi (Is meant for batch processing and submitting jobs using CLI)
- Prefect (Paid rbac)
- Mage (No scheduling)
- Dagster (No REST api to trigger workflows.)
- Kestra (Is for continuous datapipelines and cannot schedule a pod and wait for it to complete)

# Overview
<Insert table of features and characteristics>

# Requirements
- HA or persistency if orchestrator crashes or is updated. Can it resume from the previous state?
- QoS: Allow for configuring priority on jobs. E.g. if a user sneds in 200 jobs and another user only 1, the second user should gain priority. But still, the 200 jobs should not be enqueued for ever
- Job queue: If there are too many jobs, new jobs may be enqueued.
- Keeps overview of lifecycle management of workers. (Which are available, what are they currently doing)
- Is able to connect to broker to send. (Just in case, not necessary for first version of design)
- Can schedule multiple steps after each other.
- Can retrieve arguments for job when a new job is started through the API.
- Lightweight: prefer single component without extra pools of resources.
- Selfhosted
- Can run on both k8s and docker

# Celery evaluation
Link: https://github.com/celery/celery

With Celery you can create chains of tasks to create a workflow. Wireshark analysis shows that the first task in the task is scheduled by the caller, each subsequent task in the chain is scheduled by the worker who completed the previous task.

How to restart running calculations on a system restart? Celery seems to rely on in-memory of the client/app. Reference to task is lost when rebooted although the task is probably executed anyways. => Requires DB backend instead of RPC backend.

Celery seems to contain many bugs and issues due to its flexibility. Have seen multiple reports and own experiences where .get() blocks indefinitely without a clear reason. Canvas calls on rpc backend seems to be broken.
It is actively maintained but the maintainers do not seem to have the capacity to keep up with issues and demand. Application is fast but 'feels complex' due to the large amount of bugs. Clear architecture in code seems to be missing (I mean, it is Python but stil).

	- Warnings / errors van de simulator en optimalisatie (tussentijds) terugkoppelen naar de frontend.

Using DB backend instead of RPC backend allows you to retrieve task state and result. However:
	- DB needs cleaning using `.forget()`!
	- Client app will poll for the result which introduces latency

What can Celery do for us?
- Maintains state of in-progress/scheduled tasks which are flying around
- Out of the box chaining, grouping etc. to allow us to use multi-step workflows. Tasks progress to next steps even when the main app is down.
- Out of the box infrastructure for submitting and waiting for tasks.
- Can easily switch backends & broker types.
- Flower can be used to show state of the cluster. (Flower relies on events and will not show history when restarted)


Downsides:
- Celery does not implement python asyncio syntax and therefore any calls need to happen in asyncio.executor_pool.
- Seemingly redundant infra as celery requires a postgresql databases and maintains state. However, it maintains the state of all
  tasks in its cluster while we maintain state of the running workflows. This is actually not a downside! On restart, we need to know
  the state of a submitted task to know how to proceed.
- Latency due to db backend

Todo:
- How to submit progress reports through Celery? Or do we bypass Celery?
- How does Celery handle threading? It seems to leave the main thread alone so does it spawn a new (set of) thread(s) for AMQP & Celery backend activity?
- How to handle logging from tasks? Perhaps a separate UI for developers to show logs or do we send them as part of the result?

Future: Reevaluate after implementing using Celery if it is required to replace with own async code. Celery can give us the quick start we need but may introduce too many complexities and bugs. Also db backend may prove to introduce too much latency.

What is still required?
- Build a Celery app which maintains the workflow state in its own postgresql and can continue all active tasks on reboot.
  - Communicates with frontend through RabbitMQ
  - Forget results and state when response on rabbitmq has been successfully submitted
- Allow for monitoring & healthchecks of application on top of Celery. Flower does not give this overview.


# Dramatiq evaluation
Link: https://github.com/Bogdanp/dramatiq

Project seems to have quieted down. Community board is barely used and commit history does not show significant progress.
Project seems to use the 'tirant' model where the original author decides what is accepted to the code
base and what isn't.

It misses a number of features that Celery implements which we could implement ourselves:
- SQL integration for backend (only in-memory backens are currently supported).
- Event system to notify client/app of progress reports.

Code base is much smaller with less integrations. This is a big plus as Celery is HUGE.
Lots of stars so it seems that the project has had a big adoption at some point in the past.


# Performance results
Celery+RabbitMQ is super fast! No need to do extensive checking. Do not expect any significant issues with our usage.


# Celery app in the SDK instead of seperate orchestrator?
We could move the celery app to the SDK and communicate through the broker directly to the workers.
However, this would require all possible frontends to speak 'Celery' and have a Celery implementation for all
necessary languages. This is not the case, only the main Celery Python project is actively developed.
Also, this would move requirements such as retries to the frontend/user of the SDK and would complicate the SDK
significantly.

Therefore we decided to utilize a separate orchestrator with the added cost of communicating first through the OMOTES
broker layer before delivering work to the workers.


# Conclusion
We chose Celery for now. Dramatiq project is not active enough.

# Relevant links / literature
