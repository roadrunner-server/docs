# Invalid message sent to STDOUT

The message `validation failed on the message sent to STDOUT, RR docs: https://docs.roadrunner.dev/docs, invalid message: ...` means that you or some application sent an invalid raw message to `STDOUT`.
The `STDOUT` stream is reserved for RR communication with the PHP process via the `goridge` protocol (`v3`).

How to fix that?  

1. Check your application. All dependencies should send their messages to `STDERR` instead of `STDOUT`. You may see the output after the `invalid message`. It might be truncated on Windows.
2. Check any `dd` or `echo` statements you inserted. PHP workers can redirect `echo` automatically from `STDOUT` to `STDERR`, but only after the worker is fully initialized.
3. A worker from RR v1 was used. To update, see: [link](../integration/migration.md)
4. OPcache is enabled with JIT, but some extensions don't support it, which leads to warnings. Adjust the `error_reporting` configuration (use only errors): [issue](https://github.com/roadrunner-server/roadrunner/issues/1306)
5. If you use a Symfony runtime, add `APP_RUNTIME` to the server environment variables, as described here: https://github.com/php-runtime/roadrunner-symfony-nyholm.
