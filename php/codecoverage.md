# Code coverage

RoadRunner usually starts several PHP-CLI processes, and it's not obvious how to calculate code coverage in that case. Here is some advice on how to do that:

1. Do not use [debug](./debugging.md) [pool](./pool.md) mode.
2. Preferably use `num_workers: 1`.
3. If you're using `pcov`, please read this [thread](https://github.com/orgs/roadrunner-server/discussions/1440#discussioncomment-8486186) on how to configure `pcov` to work with RoadRunner.
