# Batch Client Messages

## Summary

Add a new API `BatchCommands` into `Tikv` service, which contains a vector of
raw client requests (e.g. `GetRequest`). It can reduce gRPC messages count
obviously, and according to tests the performance is improved considerably,
especially when TiKV or TiDB's CPU usage is the bottleneck.

## Motivation

In the current implementation the default configuration of `grpc-concurrency`
is `4`, so it could be a bottle-neck of TiKV under heavy load. Although we can
increase the configuration value, or make it adaptable further, we still need
to try best to reduce the gRPC CPU usage. With `BatchCommands` interface,
clients can batch some requests into one request, and send them to TiKV with
less `send` (or `sendmsg`) syscalls, which normally means less gRPC CPU usage.

## Detailed design

### BatchCommands interface

The new interface `BatchCommands` in `Tikv` service looks like this:

```proto
service Tikv {
    // ... some old interfaces.
    rpc BatchCommands(stream BatchCommandsRequest) returns (stream BatchCommandsResponse) {}
}

message BatchCommandsRequest {
    repeated Request requests = 1;
    repeated uint64 request_ids = 2;

    message Request {
        oneof cmd {
            GetRequest Get = 1;
            ScanRequest Scan = 2;
            // ... more other requests.
        }
    }
}

message BatchCommandsResponse {
    repeated Response responses = 1;
    repeated uint64 request_ids = 2;
    // Indicates the TiKV's transport layer is in heavy load or not.
    bool in_heavy_load = 3;

    message Response {
        oneof cmd {
            GetResponse Get = 1;
            ScanResponse Scan = 2;
            // ... more other responses respectively.
        }
    }
}
```

First of all, it's a dual-way streaming interface. It means TiKV is to constuct
responses for batched requests. For example, TiDB can send a
`BatchCommandsRequest` with requests `[Req-1, Req-2, Req-3, Req-4, Req-5]`,
however TiKV can give `BatchCommandsResponse`s back with responses `[Resp-1,
Resp-5, Resp-3]` and `[Resp-2, Resp-4]`. It's unnecessary to wait for any
responses which comes from one `BatchCommandsRequest`; the order just depends
on processing speed of every requests.

`request_ids` in both `BatchCommandsRequest` and `BatchCommandsResponse` is
used to relate requests and responses in the streaming interface. When clients
send a batch request to TiKV, they also need to store their `request_ids`
somewhere, and then dispatch responses (extracted from batch responses)
respectively to them.

And then `oneof` is used to unify requests and responses, instead of a
`message`. Their wired protocols are almost same, but the former's generated
code is much more better.

The last thing is `in_heavy_load` in `BatchCommandsResponse`, which is used for
TiKV can tell clients the load of TiKV is heavy or not. So clients can adjust
their strategy (e.g. add some backoff to avoid little batch) to be more
effective for TiKV.

### Implement in TiKV

The implementation in TiKV is very simple. It just extracts `Request`s from
`BatchCommandsRequest`, dispatches them to engine layer, and then collects the
`Response`s into `BatchCommandsResponse` and sends it to clients.

### How TiKV knows its gRPC threads are busy

TiKV gets CPU usage of gRPC threads by reading
`/proc/<tikv-pid>/tasks/<grpc-tid>`, which is only avaliable on Linux. If gRPC
CPU usage is greater than 80% in last 3 seconds, TiKV can tell clients it's in
heavy load.

## Drawbacks

## Alternatives

## Unresolved questions
