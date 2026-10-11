# Redis plugin

The Redis plugin exposes Redis commands through RPC. Use it to run atomic, server-side operations such as `INCRBY`, `HSET` or `ZADD` from PHP or from any other goridge client.

The plugin uses the connections of the [KV Redis driver](../kv/redis.md). Each `kv` storage with `driver: redis` is a connection for the `redis.*` RPC methods. The plugin has no config section of its own.

## Configuration

Define a `kv` storage with `driver: redis`. The storage name is the `connection` value in every `redis.*` request.

{% code title=".rr.yaml" %}

```yaml
version: "3"

rpc:
  listen: tcp://127.0.0.1:6001

kv:
  redis-rr:
    driver: redis
    config:
      addrs: ["127.0.0.1:6379"]
  counters:
    driver: redis
    config:
      addrs: ["10.0.0.5:6379"]
      db: 1
```

{% endcode %}

The `config` block and its defaults are described on the [KV Redis driver](../kv/redis.md) page. A storage with another driver, for example `memory`, is not a connection for this plugin.

The KV plugin opens the storages at start. The Redis client connects when it runs the first command, so the plugin does not check the server at start. The KV plugin closes the storages at stop.

## RPC

The plugin registers the RPC service `redis`. Each method has the name `redis.<Command>` with the command name in CamelCase. For example: `redis.IncrBy`, `redis.HGetAll` and `redis.ZRange`.

The request and reply messages are in the proto package `redis.v1`. The file is `roadrunner/api/redis/v1/redis.proto` in the [roadrunner-server/api](https://github.com/roadrunner-server/api) repository. The Go package is `github.com/roadrunner-server/api-go/v6/redis/v1`. The PHP namespace is `RoadRunner\Redis\DTO\V1`.

Each request message has the name `<Command>Request`. The first field of every request is `connection`. It is the name of a `kv` storage with `driver: redis`.

Keys, hash field names and connection names are strings. Values, list elements and set and sorted set members are bytes. Scores are doubles. Counts, indexes and durations are 64-bit integers.

The plugin uses these reply messages:

| Reply                | Fields                                         | Description                                                          |
|----------------------|------------------------------------------------|----------------------------------------------------------------------|
| `Empty`              | none                                           | The command has no result.                                           |
| `IntReply`           | `int64 value`                                  | An integer result.                                                   |
| `FloatReply`         | `double value`                                 | A floating-point result.                                             |
| `BoolReply`          | `bool value`                                   | A boolean result.                                                    |
| `StringReply`        | `string value`                                 | A string result.                                                     |
| `BytesReply`         | `optional bytes value`                         | A value. The field is absent when Redis returns nil.                 |
| `OptionalIntReply`   | `optional int64 value`                         | An integer. The field is absent when Redis returns nil.              |
| `OptionalFloatReply` | `optional double value`                        | A floating-point number. The field is absent when Redis returns nil. |
| `BytesListReply`     | `repeated bytes values`, `repeated bool found` | A list of values. `found` is set for `MGet` and `HMGet`.             |
| `BoolListReply`      | `repeated bool values`                         | A list of booleans.                                                  |
| `MapReply`           | `map<string, bytes> values`                    | A map of hash fields to values.                                      |
| `ScoredMember`       | `bytes member`, `double score`                 | A sorted set member with its score. The `ZAdd` request also uses it. |
| `ScoredMembersReply` | `repeated ScoredMember members`                | A list of sorted set members with scores.                            |
| `ScanReply`          | `uint64 cursor`, `repeated string keys`        | A page of keys and the next cursor.                                  |

### Nil replies

A nil reply from Redis is data. It is never an error.

- `BytesReply.value`, `OptionalIntReply.value` and `OptionalFloatReply.value` are absent when Redis returns nil.
- For `MGet` and `HMGet`, `found[i]` is `false` and `values[i]` is empty when the key or the field does not exist. The positions of the other items do not change.
- A precondition result is data. For example, `Set` with `nx` on an existing key returns `BoolReply` with `false`.

### Errors

The plugin returns these errors before it calls Redis:

- `no such connection: <name>`: the `connection` name is not a `kv` storage with `driver: redis`.
- `no connection provided`: the `connection` field is empty.
- `key can't be empty string`: a key, field, member, source or destination is empty or has only whitespace.
- `should provide at least 1 key`: a list of keys, fields, elements or members is empty.

Empty values are allowed.

Redis errors pass through with the text of the server. Examples are `WRONGTYPE`, `ERR value is not an integer or out of range`, `ERR no such key`, `NOPERM` and connection errors.

The RPC layer does not retry a command. The `max_retries` option of the storage applies.

### Tracing

When the [otel](../lab/otel.md) plugin is enabled, each RPC call creates a span with the name `redis:<command>`. The Redis client adds its own spans. This is the same behavior as in the [KV plugin](../kv/overview-kv.md).

### Strings and counters

| RPC method          | Request                                                                                               | Reply            | Notes                                                                                                                                                                                                                                                                                                                                               |
|---------------------|-------------------------------------------------------------------------------------------------------|------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `redis.Get`         | `key`                                                                                                 | `BytesReply`     |                                                                                                                                                                                                                                                                                                                                                     |
| `redis.Set`         | `key`, `bytes value`, `optional int64 ex`, `optional int64 px`, `bool nx`, `bool xx`, `bool keep_ttl` | `BoolReply`      | `ex` is in seconds. `px` is in milliseconds. The reply is `false` when `nx` or `xx` blocks the write. `nx` and `xx` together return the error `nx and xx are mutually exclusive`. `ex` and `px` together return the error `ex and px are mutually exclusive`. A value of `ex` or `px` that is zero or less returns the error `invalid expire time`. |
| `redis.MGet`        | `repeated string keys`                                                                                | `BytesListReply` | `found` is set. A missing key has `found[i]` `false` and an empty `values[i]`.                                                                                                                                                                                                                                                                      |
| `redis.MSet`        | `map<string, bytes> values`                                                                           | `Empty`          |                                                                                                                                                                                                                                                                                                                                                     |
| `redis.GetDel`      | `key`                                                                                                 | `BytesReply`     |                                                                                                                                                                                                                                                                                                                                                     |
| `redis.Append`      | `key`, `bytes value`                                                                                  | `IntReply`       | The value is the new length of the string.                                                                                                                                                                                                                                                                                                          |
| `redis.StrLen`      | `key`                                                                                                 | `IntReply`       |                                                                                                                                                                                                                                                                                                                                                     |
| `redis.Incr`        | `key`                                                                                                 | `IntReply`       |                                                                                                                                                                                                                                                                                                                                                     |
| `redis.IncrBy`      | `key`, `int64 increment`                                                                              | `IntReply`       |                                                                                                                                                                                                                                                                                                                                                     |
| `redis.IncrByFloat` | `key`, `double increment`                                                                             | `FloatReply`     |                                                                                                                                                                                                                                                                                                                                                     |
| `redis.Decr`        | `key`                                                                                                 | `IntReply`       |                                                                                                                                                                                                                                                                                                                                                     |
| `redis.DecrBy`      | `key`, `int64 decrement`                                                                              | `IntReply`       |                                                                                                                                                                                                                                                                                                                                                     |

### Keys

| RPC method       | Request                                                       | Reply         | Notes                                                                        |
|------------------|---------------------------------------------------------------|---------------|------------------------------------------------------------------------------|
| `redis.Del`      | `repeated string keys`                                        | `IntReply`    |                                                                              |
| `redis.Unlink`   | `repeated string keys`                                        | `IntReply`    |                                                                              |
| `redis.Exists`   | `repeated string keys`                                        | `IntReply`    |                                                                              |
| `redis.Expire`   | `key`, `int64 seconds`                                        | `BoolReply`   |                                                                              |
| `redis.PExpire`  | `key`, `int64 milliseconds`                                   | `BoolReply`   |                                                                              |
| `redis.ExpireAt` | `key`, `int64 unix_seconds`                                   | `BoolReply`   |                                                                              |
| `redis.TTL`      | `key`                                                         | `IntReply`    | The value `-2` means the key does not exist. The value `-1` means no expiry. |
| `redis.PTTL`     | `key`                                                         | `IntReply`    | The same values as `redis.TTL`, in milliseconds.                             |
| `redis.Persist`  | `key`                                                         | `BoolReply`   |                                                                              |
| `redis.Type`     | `key`                                                         | `StringReply` | The value is `none` when the key does not exist.                             |
| `redis.Rename`   | `key`, `new_key`                                              | `BoolReply`   | The reply is always `true`. A missing key is a Redis error.                  |
| `redis.Scan`     | `uint64 cursor`, `string match`, `int64 count`, `string type` | `ScanReply`   |                                                                              |

In cluster mode, `redis.Scan` reads one node per call and does not iterate the whole cluster.

### Hashes

| RPC method           | Request                            | Reply            | Notes                                                                            |
|----------------------|------------------------------------|------------------|----------------------------------------------------------------------------------|
| `redis.HSet`         | `key`, `map<string, bytes> fields` | `IntReply`       | The value is the number of added fields.                                         |
| `redis.HSetNX`       | `key`, `field`, `bytes value`      | `BoolReply`      |                                                                                  |
| `redis.HGet`         | `key`, `field`                     | `BytesReply`     |                                                                                  |
| `redis.HMGet`        | `key`, `repeated string fields`    | `BytesListReply` | `found` is set. A missing field has `found[i]` `false` and an empty `values[i]`. |
| `redis.HGetAll`      | `key`                              | `MapReply`       |                                                                                  |
| `redis.HDel`         | `key`, `repeated string fields`    | `IntReply`       |                                                                                  |
| `redis.HExists`      | `key`, `field`                     | `BoolReply`      |                                                                                  |
| `redis.HLen`         | `key`                              | `IntReply`       |                                                                                  |
| `redis.HKeys`        | `key`                              | `BytesListReply` |                                                                                  |
| `redis.HVals`        | `key`                              | `BytesListReply` |                                                                                  |
| `redis.HIncrBy`      | `key`, `field`, `int64 increment`  | `IntReply`       |                                                                                  |
| `redis.HIncrByFloat` | `key`, `field`, `double increment` | `FloatReply`     |                                                                                  |

### Lists

| RPC method     | Request                                                         | Reply            | Notes                                                                                                                                     |
|----------------|-----------------------------------------------------------------|------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| `redis.LPush`  | `key`, `repeated bytes elements`                                | `IntReply`       | The value is the length of the list.                                                                                                      |
| `redis.RPush`  | `key`, `repeated bytes elements`                                | `IntReply`       | The value is the length of the list.                                                                                                      |
| `redis.LPop`   | `key`, `int64 count`                                            | `BytesListReply` | The list is empty when the key does not exist. The value `0` or a negative value of `count` pops one element. The reply is always a list. |
| `redis.RPop`   | `key`, `int64 count`                                            | `BytesListReply` | The same rules as `redis.LPop`.                                                                                                           |
| `redis.LRange` | `key`, `int64 start`, `int64 stop`                              | `BytesListReply` |                                                                                                                                           |
| `redis.LLen`   | `key`                                                           | `IntReply`       |                                                                                                                                           |
| `redis.LIndex` | `key`, `int64 index`                                            | `BytesReply`     |                                                                                                                                           |
| `redis.LSet`   | `key`, `int64 index`, `bytes element`                           | `BoolReply`      | The reply is `true`. An index out of range is a Redis error.                                                                              |
| `redis.LRem`   | `key`, `int64 count`, `bytes element`                           | `IntReply`       |                                                                                                                                           |
| `redis.LTrim`  | `key`, `int64 start`, `int64 stop`                              | `BoolReply`      |                                                                                                                                           |
| `redis.LMove`  | `source`, `destination`, `string where_from`, `string where_to` | `BytesReply`     | `where_from` and `where_to` are `LEFT` or `RIGHT`.                                                                                        |

### Sets

| RPC method           | Request                         | Reply            | Notes                                      |
|----------------------|---------------------------------|------------------|--------------------------------------------|
| `redis.SAdd`         | `key`, `repeated bytes members` | `IntReply`       |                                            |
| `redis.SRem`         | `key`, `repeated bytes members` | `IntReply`       |                                            |
| `redis.SMembers`     | `key`                           | `BytesListReply` |                                            |
| `redis.SIsMember`    | `key`, `bytes member`           | `BoolReply`      |                                            |
| `redis.SMIsMember`   | `key`, `repeated bytes members` | `BoolListReply`  |                                            |
| `redis.SCard`        | `key`                           | `IntReply`       |                                            |
| `redis.SPop`         | `key`                           | `BytesReply`     | Removes and returns one member.            |
| `redis.SPopN`        | `key`, `int64 count`            | `BytesListReply` | Removes and returns up to `count` members. |
| `redis.SRandMember`  | `key`                           | `BytesReply`     | Returns one member.                        |
| `redis.SRandMemberN` | `key`, `int64 count`            | `BytesListReply` | Returns `count` members.                   |
| `redis.SDiff`        | `repeated string keys`          | `BytesListReply` |                                            |
| `redis.SInter`       | `repeated string keys`          | `BytesListReply` |                                            |
| `redis.SUnion`       | `repeated string keys`          | `BytesListReply` |                                            |

Redis has a one-member form and a `count` form of `SPOP` and `SRANDMEMBER`. The two forms have different reply types. For this reason, each form is a separate RPC method.

### Sorted sets

| RPC method               | Request                                                                                                                             | Reply                | Notes                                                                                     |
|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------|----------------------|-------------------------------------------------------------------------------------------|
| `redis.ZAdd`             | `key`, `repeated ScoredMember members`, `bool nx`, `bool xx`, `bool gt`, `bool lt`, `bool ch`                                       | `IntReply`           |                                                                                           |
| `redis.ZRem`             | `key`, `repeated bytes members`                                                                                                     | `IntReply`           |                                                                                           |
| `redis.ZScore`           | `key`, `bytes member`                                                                                                               | `OptionalFloatReply` | The value is absent when the member does not exist.                                       |
| `redis.ZIncrBy`          | `key`, `double increment`, `bytes member`                                                                                           | `FloatReply`         |                                                                                           |
| `redis.ZCard`            | `key`                                                                                                                               | `IntReply`           |                                                                                           |
| `redis.ZCount`           | `key`, `string min`, `string max`                                                                                                   | `IntReply`           | `min` and `max` are score strings.                                                        |
| `redis.ZRank`            | `key`, `bytes member`                                                                                                               | `OptionalIntReply`   | The value is absent when the member does not exist.                                       |
| `redis.ZRevRank`         | `key`, `bytes member`                                                                                                               | `OptionalIntReply`   | The value is absent when the member does not exist.                                       |
| `redis.ZRange`           | `key`, `string start`, `string stop`, `bool by_score`, `bool by_lex`, `bool rev`, `int64 offset`, `int64 count`, `bool with_scores` | `ScoredMembersReply` | See the rules for `ZRange` below.                                                         |
| `redis.ZPopMin`          | `key`, `int64 count`                                                                                                                | `ScoredMembersReply` | The value `0` or a negative value of `count` pops one member. The reply is always a list. |
| `redis.ZPopMax`          | `key`, `int64 count`                                                                                                                | `ScoredMembersReply` | The same rules as `redis.ZPopMin`.                                                        |
| `redis.ZRemRangeByRank`  | `key`, `int64 start`, `int64 stop`                                                                                                  | `IntReply`           |                                                                                           |
| `redis.ZRemRangeByScore` | `key`, `string min`, `string max`                                                                                                   | `IntReply`           | `min` and `max` are score strings.                                                        |

Rules for `redis.ZRange`:

- By default, `start` and `stop` are index strings.
- With `by_score`, `start` and `stop` are score strings. Examples are `-inf`, `(1.5` and `10`.
- With `by_lex`, `start` and `stop` are lex strings. Examples are `-`, `+`, `[a` and `(b`.
- `by_score` and `by_lex` together return the error `by_score and by_lex are mutually exclusive`.
- The plugin sends `LIMIT offset count` when `offset` or `count` is not `0`. Redis accepts `LIMIT` only with `by_score` or `by_lex`.
- `score` is `0` in the reply when `with_scores` is `false`.

## Metrics

The RPC methods use the clients of the KV storages, so the KV Redis driver metrics apply. The metric names start with `rr_redis_pool_`. See the [KV Redis driver](../kv/redis.md) page and the [metrics](../lab/metrics.md) plugin.

## Example

This Go client calls `redis.IncrBy` on the storage `redis-rr`. The RPC address is from the `rpc` section of the config.

```go
package main

import (
    "fmt"
    "net"
    "net/rpc"

    redisV1 "github.com/roadrunner-server/api-go/v6/redis/v1"
    goridgeRpc "github.com/roadrunner-server/goridge/v4/pkg/rpc"
)

func main() {
    conn, err := net.Dial("tcp", "127.0.0.1:6001")
    if err != nil {
        panic(err)
    }

    client := rpc.NewClientWithCodec(goridgeRpc.NewClientCodec(conn))

    reply := &redisV1.IntReply{}
    err = client.Call("redis.IncrBy", &redisV1.IncrByRequest{
        Connection: "redis-rr",
        Key:        "visits",
        Increment:  1,
    }, reply)
    if err != nil {
        panic(err)
    }

    fmt.Println(reply.GetValue())
}
```

The PHP DTO classes are in the namespace `RoadRunner\Redis\DTO\V1`. Call them through `Spiral\Goridge\RPC\RPC` with the `ProtobufCodec`, in the same way as the KV DTOs.
