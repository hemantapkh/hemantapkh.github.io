---
title: "FastAPI and Celery: Prevent This Common Mistake That Crashed Our Service"
description: Are you using Celery tasks in FastAPI? Learn how it can crash your service and how to fix it.
author: hemantapkh
date: 2025-05-11 18:35:00 +0000
categories: [SRE]
tags: [python, fastapi, docsumo]
pin: false
math: false
mermaid: false
image:
  path: https://assets.hemantapkh.com/blog/celery-blocking-fastapi-eventloop/thumbnail.webp
---

Recently, our service on FastAPI crashed unexpectedly. After some investigation, we discovered that the root cause was our Redis service being down. But Redis wasn’t even a core component of the application, So why did our entire service crash just because Redis was unavailable? The answer lies in how we were using Celery tasks.

FastAPI is an asynchronous framework, while Celery is inherently synchronous. Yes, synchronous, you heard it right! Although Celery can run tasks asynchronously, its communication with the broker (like Redis or RabbitMQ) is synchronous. This can become a disaster when the broker is unavailable.

## What Happened in Our Case?

In our application, few of the endpoint relied on Celery to dispatch background tasks. Unfortunately, these endpoints were called frequently, and when Redis (our Celery broker) went down, every request to these endpoints caused Celery to hang while attempting to reconnect. Since FastAPI runs on a single event loop, this blocking behavior stalled the entire application for ~20 seconds per request. The result? Our readiness probes started failing. As the issue propagated across all pods in our Kubernetes cluster, the entire service was marked as unhealthy and taken offline.

## How to Fix This Issue

To prevent Celery from blocking FastAPI’s event loop, you can run Celery tasks in a separate thread. Below, I’ve created a patched version of Celery with custom async methods (`apply_asyncx` and `delayx`) to run the task submissing in a separate thread. The `asyncio.to_thread` function enables us to run synchronous functions in a separate thread while awaiting their results asynchronously in the main event loop.

```python
from celery import Celery

def create_celery_app(broker_url: str, backend_url: str | None = None) -> Celery:
    celery_app = Celery("tasks", broker=broker_url, backend=backend_url or broker_url)

    class PatchedTask(celery_app.Task):  # type: ignore[name-defined]
        def __init__(self, *args, **kwargs):
            super().__init__(*args, **kwargs)

        async def apply_asyncx(self, args=None, kwargs=None, **options):
            result = await asyncio.to_thread(
                super().apply_async, args, kwargs, **options
            )
            return result

        async def delayx(self, *args, **kwargs):
            result = await self.apply_asyncx(args, kwargs)
            return result

    celery_app.Task = PatchedTask  # type: ignore[name-defined]

    return celery_app
```

> I avoided using Starlette’s default thread pool (via run_in_threadpool) as it’s limited to 40 threads by default and shared with other FastAPI tasks. Exhausting this pool blocks the app. Instead, `asyncio.to_thread` utilizes a separate thread pool, avoiding this bottleneck. For greater control over threading, you can also create a custom thread pool using`concurrent.futures.ThreadPoolExecutor` instead of relying on the default thread pool executor provided by `asyncio.to_thread`.
{: .prompt-warning }

With this implementation, we can now use the `apply_asyncx` and `delayx` methods instead of Celery's default `apply_async` and `delay`. These new methods ensure that task dispatching happens in a separate thread, safeguarding FastAPI's event loop from being blocked during broker outages.

```python
from fastapi import FastAPI


app = FastAPI()
celery_tasks = create_celery_app("redis://localhost:6379/0")


@celery_tasks.task(name="app.tasks.requests_post")  # type: ignore[empty-body]
def my_task(*args, **kwargs) -> Tuple[str, bool]:
    pass

@app.post("/trigger-task")
async def trigger_task():
    result = await my_task.delayx("arg1", "arg2")
    return {"message": "Task triggered", "task_id": str(result.id)}
```

## Conclusion

Now, if Redis goes down, only the endpoints depending on Redis are affected, while others continue to function normally. While we’re committed to Celery due to existing infrastructure and dependencies, if you’re starting fresh or planning new implementations, consider exploring other frameworks like [Arq](https://github.com/python-arq/arq) and [TaskIQ](https://github.com/taskiq-python/taskiq).

> For more FastAPI tips, I highly recommend [FastAPI Tips](https://github.com/Kludex/fastapi-tips), a helpful resource maintained by one of the FastAPI maintainers.
{: .prompt-tip }
