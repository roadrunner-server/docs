# Production Usage

When utilizing RoadRunner in a production environment, it is important to consider various tips and suggestions to
ensure optimal performance and stability.

## State and memory

One crucial aspect to keep in mind is that state and memory are **not shared** between different worker instances, but
they are **shared** for a single worker instance. As a result, it is essential to take precautions such as closing all
descriptors and avoiding state pollution to prevent memory leaks and ensure application stability.

**Here are some tips to keep in mind:**

- Make sure you close all descriptors (especially on fatal exceptions).
- Watch out for memory leaks—you need to be more selective about the components you use. Workers will be restarted in
  case of a memory leak, but it should not be difficult to avoid this problem altogether by designing your application
  properly.
- Avoid state pollution (i.e., caching globals or user data in memory).
 - Database connections and any `pipe`/`socket` are potential points of failure. An easy way to deal with them is to
  close all connections after every iteration. Note that this is not the most performant solution.

{% hint style="warning" %}

Consider calling `gc_collect_cycles` after every execution if you want to keep memory usage low. **However, this will slow down execution a lot. Use with caution**.

{% endhint %}

## Useful tips

- Make sure you are **NOT** listening on `0.0.0.0` in the RPC service (unless in Docker).
- Connect to a worker using `pipes` for better performance (Unix sockets are just a bit slower).
- Adjust your pool timings to the values you like.
- **Number of workers = number of CPU threads** in your system; unless your application is I/O-bound, then choose the
  number heuristically based on the available memory on the server.
- Consider using `max_jobs` for your workers if you experience application stability or memory issues over time.
- RoadRunner has 40% higher performance when using keep-alive connections.
- Set the memory limit at least 10–20% below `max_memory_usage`.
- Since RoadRunner runs workers from the CLI, you need to enable `OPcache` in the CLI with `opcache.enable_cli=1`.
- Make sure to use the [health check endpoint](../lab/health.md) when running `rr` in a cloud environment.
- Use the `user` option in the `server` plugin configuration to start worker processes as the specified user on
  Linux-based systems. Note that in this case RoadRunner should be started as `root` to allow fork-exec processes
  from different users.
- If your application uses mostly I/O (disk, network, etc.), you can allocate as many workers as you have memory for the
  application. Workers are cheap. A hello-world worker uses no more than **~26MB** of RSS memory.
- For CPU-bound operations, monitor average CPU load and choose the number of workers to target **90–95%** CPU utilization. Leave a
  few percent for the Go GC.
- If you have `~const` worker latency, you can calculate the number of workers needed to handle the
  target [load](https://github.com/orgs/roadrunner-server/discussions/799#discussioncomment-1332646).
