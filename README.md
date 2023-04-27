# Fluent Bit Backlog Bug

This is a docker compose project to demonstrate a Fluent Bit chunk validation failure backlog bug in version 2.1.X.

## Quickstart

```sh
# setup
docker compose up -d

# stop
docker compose stop

# start
docker compose start

# teardown without removing volumes
docker compose down

# teardown and remove volumes
docker compose down -v
```

Tail the v2.1.X container:

```
$ docker logs -f flb-backlog-bug-flb-v2_1-1
Fluent Bit v2.1.2
* Copyright (C) 2015-2022 The Fluent Bit Authors
* Fluent Bit is a CNCF sub-project under the umbrella of Fluentd
* https://fluentbit.io

[2023/04/27 06:09:22] [ info] [fluent bit] version=2.1.2, commit=113c7c00d2, pid=1
[2023/04/27 06:09:22] [ info] [storage] ver=1.4.0, type=memory+filesystem, sync=normal, checksum=off, max_chunks_up=128
[2023/04/27 06:09:22] [ info] [storage] backlog input plugin: storage_backlog.1
[2023/04/27 06:09:22] [ info] [cmetrics] version=0.6.1
[2023/04/27 06:09:22] [ info] [ctraces ] version=0.3.0
[2023/04/27 06:09:22] [ info] [input:tcp:tcp.0] initializing
[2023/04/27 06:09:22] [ info] [input:tcp:tcp.0] storage_strategy='filesystem' (memory + filesystem)
[2023/04/27 06:09:22] [ info] [input:storage_backlog:storage_backlog.1] initializing
[2023/04/27 06:09:22] [ info] [input:storage_backlog:storage_backlog.1] storage_strategy='memory' (memory only)
[2023/04/27 06:09:22] [ info] [input:storage_backlog:storage_backlog.1] queue memory limit: 9.5M
[2023/04/27 06:09:22] [ info] [output:forward:forward.0] worker #0 started
[2023/04/27 06:09:22] [ info] [sp] stream processor started
[2023/04/27 06:09:22] [ info] [output:forward:forward.0] worker #1 started
```

Tail the v2.0.X container:

```
$ docker logs -f flb-backlog-bug-flb-v2_0-1
Fluent Bit v2.0.11
* Copyright (C) 2015-2022 The Fluent Bit Authors
* Fluent Bit is a CNCF sub-project under the umbrella of Fluentd
* https://fluentbit.io

[2023/04/27 06:09:22] [ info] [fluent bit] version=2.0.11, commit=bf22d53a69, pid=1
[2023/04/27 06:09:22] [ info] [storage] ver=1.4.0, type=memory+filesystem, sync=normal, checksum=off, max_chunks_up=128
[2023/04/27 06:09:22] [ info] [storage] backlog input plugin: storage_backlog.1
[2023/04/27 06:09:22] [ info] [cmetrics] version=0.6.0
[2023/04/27 06:09:22] [ info] [ctraces ] version=0.3.0
[2023/04/27 06:09:22] [ info] [input:tcp:tcp.0] initializing
[2023/04/27 06:09:22] [ info] [input:tcp:tcp.0] storage_strategy='filesystem' (memory + filesystem)
[2023/04/27 06:09:22] [ info] [input:storage_backlog:storage_backlog.1] initializing
[2023/04/27 06:09:22] [ info] [input:storage_backlog:storage_backlog.1] storage_strategy='memory' (memory only)
[2023/04/27 06:09:22] [ info] [input:storage_backlog:storage_backlog.1] queue memory limit: 9.5M
[2023/04/27 06:09:22] [ info] [sp] stream processor started
[2023/04/27 06:09:22] [ info] [output:forward:forward.0] worker #0 started
[2023/04/27 06:09:22] [ info] [output:forward:forward.0] worker #1 started
```

## Demonstrate expected behavior in v2.0.X

First, send a message to Fluent Bit v2.1.X using TCP (don't forget to hit `<enter>`):

```
$ docker exec -it flb-backlog-bug-flb-v2_0-1 sh
# nc localhost 5170
hello

```

This will result in a flush failure (expected):

```
[2023/04/27 06:10:28] [ warn] [net] getaddrinfo(host='doesntexist', err=4): Domain name not found
[2023/04/27 06:10:28] [error] [output:forward:forward.0] no upstream connections available
[2023/04/27 06:10:28] [ warn] [engine] failed to flush chunk '1-1682575828.625496799.flb', retry in 7 seconds: task_id=0, input=tcp.0 > output=forward.0 (out_id=0)
```

Next, restart the docker container:

```
$ docker restart flb-backlog-bug-flb-v2_0-1
```

This will add the previously failed chunk to the storage_backlog queue (expected):

```
[2023/04/27 06:10:54] [ info] [input:storage_backlog:storage_backlog.1] register tcp.0/1-1682575828.625496799.flb
[2023/04/27 06:10:54] [ info] [input:storage_backlog:storage_backlog.1] queueing tcp.0:1-1682575828.625496799.flb
```

## Demonstrate unexpected behavior in v2.1.X

First, send a message to Fluent Bit v2.1.X using TCP (don't forget to hit `<enter>`):

```
$ docker exec -it flb-backlog-bug-flb-v2_1-1 sh
# nc localhost 5170
hello

```

This will result in a flush failure (expected):

```
[2023/04/27 06:11:24] [ warn] [net] getaddrinfo(host='doesntexist', err=4): Domain name not found
[2023/04/27 06:11:24] [error] [output:forward:forward.0] no upstream connections available
[2023/04/27 06:11:24] [ warn] [engine] failed to flush chunk '1-1682575884.551412366.flb', retry in 11 seconds: task_id=0, input=tcp.0 > output=forward.0 (out_id=0)
```

Next, restart the docker container:

```sh
$ docker restart flb-backlog-bug-flb-v2_1-1
```

This will result in a validation failure of the previously failed chunk (unexpected):

```
[2023/04/27 06:11:37] [ info] [input:storage_backlog:storage_backlog.1] register tcp.0/1-1682575884.551412366.flb
[2023/04/27 06:11:37] [error] [input:storage_backlog:storage_backlog.1] chunk validation failed, data might be corrupted. No valid records found, the chunk will be discarded.
[2023/04/27 06:11:37] [error] [input:storage_backlog:storage_backlog.1] removing chunk tcp.0:1-1682575884.551412366.flb from the queue
```
