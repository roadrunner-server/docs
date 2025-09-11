# Allocate timeout error

RoadRunner allocates workers like regular processes (via `fork` + `exec` syscalls). While RoadRunner is creating a worker (connecting to pipes, TCP, etc.), part of the worker may freeze during the initial handshake. RoadRunner waits `pool.allocate_timeout` for the handshake to complete and returns this error if `pool.allocate_timeout` is exceeded.

## How to fix that?  

1. Check the `pool.allocate_timeout` option. It should be in the form `pool.allocate_timeout: 1s` or `pool.allocate_timeout: 1h`; always specify units. Also note that `1s` might be too small—try increasing it.
2. `xdebug` might freeze the worker spawn while improperly configured. See this tutorial: [link](../php/debugging.md)
3. If you use a server relay other than `pipes`, check [these options](https://github.com/roadrunner-server/roadrunner/blob/master/.rr.yaml#L75). They control the initial handshake timeout between RR and the PHP process when using `sockets` or `TCP`.

## What is `allocate_timeout` pool option?

The `pool.allocate_timeout` controls two things in RoadRunner: first, the timeout to allocate a PHP worker when RoadRunner starts (initializing workers); second, the timeout to take a worker from the internal worker container when a request arrives.
