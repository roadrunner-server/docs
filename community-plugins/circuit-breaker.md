# Circuit Breaker [Under Development]

Circuit breaker HTTP middleware is used to detect failures and prevent failures from constantly recurring.
This middleware is still under development.

## Configuration

```yaml
version: '3'

rpc:
  listen: tcp://127.0.0.1:6002

server:
  command: "php php_test_files/php-test-worker.php"
  relay: "pipes"

http:
  address: 127.0.0.1:17876
  max_request_size: 1024
  middleware: ["circuitbreaker"]
  pool:
    num_workers: 1

circuitbreaker:
  max_error_rate: 0.1
  error_codes: [ 500, 503 ]

logs:
  mode: development
  level: debug
```
