# About Temporal.IO
Temporal is a distributed, scalable, durable, and highly available orchestration engine used to execute asynchronous 
long-running business logic in a scalable and resilient way.

Read more at [official website](https://docs.temporal.io/).

RoadRunner >= 2.0 includes a plugin to execute Temporal workflows and activities. Make sure to write [temporal worker](/workflow/worker.md).

Activate plugin via config:

```yaml
rpc:
  listen: tcp://127.0.0.1:6001

server:
  command: "php worker.php"

temporal:
  address: "127.0.0.1:7233"
  worker_heartbeat_interval: 10s
  activities:
    num_workers: 10

logs:
  level: debug
  channels:
    temporal.level: error
```

## Worker Heartbeats

The `temporal.worker_heartbeat_interval` option sets how often SDK workers report their state to the Temporal server. Use a duration from `1s` through `60s`. If the option is omitted or set to `0s`, the SDK uses its default of `60s`.

Positive intervals below `1s` are clamped to `1s`. Intervals above `60s` are clamped to `60s`. The plugin logs a warning when it clamps a value.

The plugin supplies host CPU and memory usage and the PHP SDK identity for these heartbeats. This option does not control activity heartbeats or the activity heartbeat timeout.

## Example
Integrated workflow server provides the ability to create very complex, long-running activities.

```php
class SubscriptionWorkflow implements SubscriptionWorkflowInterface
{
    private $account;

    public function __construct()
    {
        $this->account = Workflow::newActivityStub(
            AccountActivityInterface::class,
            ActivityOptions::new()
                ->withScheduleToCloseTimeout(DateInterval::createFromDateString('2 seconds'))
        );
    }

    public function subscribe(string $userID)
    {
        yield $this->account->sendWelcomeEmail($userID);

        try {
            $trialPeriod = true;
            while (true) {
                // Lower period duration to observe workflow behavior
                yield Workflow::timer(DateInterval::createFromDateString('30 days'));
                yield $this->account->chargeMonthlyFee($userID);

                if ($trialPeriod) {
                    yield $this->account->sendEndOfTrialEmail($userID);
                    $trialPeriod = false;
                    continue;
                }

                yield $this->account->sendMonthlyChargeEmail($userID);
            }
        } catch (CanceledFailure $e) {
            yield Workflow::asyncDetached(
                function () use ($userID) {
                    yield $this->account->processSubscriptionCancellation($userID);
                    yield $this->account->sendSorryToSeeYouGoEmail($userID);
                }
            );
        }
    }
}
```

> Read more at [official website](https://docs.temporal.io/develop/php/).
