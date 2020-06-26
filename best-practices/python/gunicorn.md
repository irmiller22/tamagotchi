# Python - Gunicorn

## Rule of Thumbs

- For CPU bounded apps, increase workers / cores.
- For I/O bounded apps, use pseudo-threads.

## What is Gunicorn?

It's a Python Web Server Gateway Interface (WSGI) that lives between a reverse proxy (nginx) or load balancer (AWS ELB) and a web application like Django or Flask.

## Architecture

Gunicorn implements what's called a pre-fork web server.

Specifics:

- gunicorn starts a single master process that gets forked, and the resulting child processes are the workers.
- role of the master process is to make sure that the number of workers is the same as the one defined in the settings. So if any of the child processes die, the master process starts another child process by forking itself again.
- role of the child (worker) processes is to handle HTTP requests.
- the "pre" in pre-forked means that the master process creates the workers before handling any HTTP request.
- the OS kernel handles load balancing between requests.

## Performance

### Concurrency

1. 1st means of concurrency - workers, aka UNIX processes

Each worker is a UNIX process that loads the Python app. No shared memory between workers. Suggested number of workers: `(2 * CPU) + 1`. So if there are 2 CPU cores, you want 5 workers.

Establish this via `gunicorn --workers=5 main:app`.

2. 2nd means of concurrency - threads

Gunicorn allows for each worker to have multiple threads. To use threads, you can use the `threads` setting. Whenever the `threads` setting is used, the worker class is set to `gthread` (currently Pillar has this set to `gevent`).

Command invocation: `gunicorn --workers=5 --threads=2 main:app`

Same as: `gunicorn --workers=5 --threads=2 --worker-class=gthread main:app`

Suggested number of concurrent workers is still `(2 * CPU) + 1`. So if you're using 2 CPU cores, you'd still want 5 workers (perhaps 3 workers, 2 of which have 2 threads, and 1 that has 1 thread).

3. 3rd means of concurrency - pseudo-threads

There are several Python libraries that enable concurrency via "pseudo-threads" (which are implemented via co-routines), such as gevent and asyncio.

If we want to run a single-core machine using gevent: `gunicorn --worker-class=gevent --worker-connections=1000 --workers=3 main:app`

`(2*CPU)+1` is still the suggested formula for the number of workers. Based on the configuration above, we would have a maximum of 3000 concurrent requests.

### Concurrency vs Parallelism

- Concurrency is when 2 or more tasks are being performed at the same time, which might mean that only 1 of them is being worked on while the other ones are paused.
- Parallelism is when 2 or more tasks are executing at the same time.

In Python, threads and pseudo-threads are a means of concurrency, but not parallelism; while workers are a means of both concurrency and parallelism.

## Use Cases

1. If the application is I/O bounded, the best performance usually comes from using “pseudo-threads” (gevent or asyncio). As we have seen, Gunicorn supports this programming paradigm by setting the appropriate `worker` class and adjusting the value of workers to `(2*CPU) + 1`.

2. If the application is CPU bounded, it doesn’t matter how many concurrent requests are handled by the application. The only thing that matters is the number of parallel requests. Due to Python’s GIL, threads and “pseudo-threads” cannot run in parallel. The only way to achieve parallelism is to increase workers to the suggested `(2*CPU) + 1`, understanding that the maximum number of parallel requests is the number of cores.

3. If there is a concern about the application memory footprint, using threads and its corresponding gthread worker class in favor of workers yields better performance because the application is loaded once per worker and every thread running on the worker shares some memory, this comes to the expense of some additional CPU consumption.

4. If you don’t know you are doing, start with the simplest configuration, which is only setting workers to (2\*CPU)+1 and don’t worry about threads. From that point, it’s all trial and error with benchmarking. If the bottleneck is memory, start introducing threads. If the bottleneck is I/O, consider a different python programming paradigm. If the bottleneck is CPU, consider using more cores and adjusting the workers value.