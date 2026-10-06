# About Temporal.IO
Temporal runs durable workflows and activities for long-running business operations.

Read more at [official website](https://docs.temporal.io/).

RoadRunner includes a plugin to execute Temporal workflows and activities. Configure a [Temporal worker](worker.md).

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
    temporal:
      level: error
```

## Worker Heartbeats

The `temporal.worker_heartbeat_interval` option sets how often SDK workers report their state to the Temporal server. Use a duration from `1s` through `60s`. If the option is omitted or set to `0s`, the SDK uses its default of `60s`.

Positive intervals below `1s` are clamped to `1s`. Intervals above `60s` are clamped to `60s`. The plugin logs a warning when it clamps a value.

The plugin supplies host CPU and memory usage and the PHP SDK identity for these heartbeats. This option does not control activity heartbeats or the activity heartbeat timeout.

## Custom Payload Converters

The plugin collects Go implementations of Temporal's `converter.PayloadConverter` interface from a [custom RoadRunner build](../customization/build.md). Use `data_converters` to select converters by their `Encoding()` value and set their order:

{% code title=".rr.yaml fragment" %}

```yaml
temporal:
  address: "127.0.0.1:7233"
  data_converters:
    - "json/custom"
```

{% endcode %}

This example requires a registered converter with the encoding `json/custom`. An unknown encoding fails initialization. If the list is empty or omitted, the plugin uses all registered converters. Set the list when converter order matters.

The composite converter tries the standard nil, byte, and Protobuf converters first. It then tries the selected custom converters, followed by the standard JSON converter. See the [payload converter example](https://github.com/roadrunner-server/data-converter-example).

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
