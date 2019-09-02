---
bg: 'mongodb.jpg'
layout: post
title:  "mongodb复制集"
crawlertitle: "mongodb"
summary: ""
date:   2019-09-03 09:00:00 +0700
categories: posts
tags: 'mongodb'
author: 宋天
---


mongodb复制集相关


#### Why? 为什么要使用复制集
- 高可用性（HA）
- 数据安全
- 分流/分工


#### 复制集结构

- 一个primary,多个secondry
- primary负责写读请求，副节点从主节点复制数据，secondry可以负责读请求，但可能会有数据不一致的情况，因为副节点从主节点复制数据会有一定延迟
- 每个节点之间互相发送心跳请求，默认每隔2秒发送一次，如果10秒则请求超时
- 复制集最多可以有50个节点（因为增加节点会使心跳请求成平方的增加）

- 当primary down掉时，候选节点会发起选举，当投票赞成结果超过半数时，会将候选节点选为主节点。投票节点的评判标准是，查看该候选节点与之前的主节点的同步状态是否超过自身和之前的主节点的同步状态
- 复制集最多有7个投票节点
- 触发选举的事件
    1. 主节点和副节点之间的心跳请求超时
    2. 复制集初始化
    3. 新节点加入复制集
- 投票机节点，没有数据，可以投票，不能成为主节点
- 新节点通过oplog来从主节点复制数据，复制操作记录。


mongo日志
- mongodb复制集中的secondary接管primary时的日志

```
2019-04-16T21:01:00.460+0800 I NETWORK  [LogicalSessionCacheRefresh] Starting new replica set monitor for rs0/192.168.0.105:3717,192.168.0.106:3717,192.168.0.99:3717
2019-04-16T21:01:06.814+0800 I STORAGE  [WT OplogTruncaterThread: local.oplog.rs] WiredTiger record store oplog truncation finished in: 13ms
2019-04-16T21:01:13.279+0800 I STORAGE  [WT OplogTruncaterThread: local.oplog.rs] WiredTiger record store oplog truncation finished in: 51ms
2019-04-16T21:01:18.380+0800 I STORAGE  [WT OplogTruncaterThread: local.oplog.rs] WiredTiger record store oplog truncation finished in: 14ms
2019-04-16T21:01:29.046+0800 I STORAGE  [WT OplogTruncaterThread: local.oplog.rs] WiredTiger record store oplog truncation finished in: 2ms
2019-04-16T21:01:36.566+0800 I STORAGE  [WT OplogTruncaterThread: local.oplog.rs] WiredTiger record store oplog truncation finished in: 1ms
2019-04-16T21:01:43.937+0800 I STORAGE  [WT OplogTruncaterThread: local.oplog.rs] WiredTiger record store oplog truncation finished in: 2ms
2019-04-16T21:01:52.939+0800 I STORAGE  [WT OplogTruncaterThread: local.oplog.rs] WiredTiger record store oplog truncation finished in: 1ms
2019-04-16T21:02:01.244+0800 I STORAGE  [WT OplogTruncaterThread: local.oplog.rs] WiredTiger record store oplog truncation finished in: 1ms
2019-04-16T21:02:08.904+0800 I STORAGE  [WT OplogTruncaterThread: local.oplog.rs] WiredTiger record store oplog truncation finished in: 2ms
2019-04-16T21:02:15.546+0800 I STORAGE  [WT OplogTruncaterThread: local.oplog.rs] WiredTiger record store oplog truncation finished in: 1ms
2019-04-16T21:02:22.607+0800 I STORAGE  [WT OplogTruncaterThread: local.oplog.rs] WiredTiger record store oplog truncation finished in: 2ms
2019-04-16T21:02:29.310+0800 I STORAGE  [WT OplogTruncaterThread: local.oplog.rs] WiredTiger record store oplog truncation finished in: 1ms
2019-04-16T21:02:37.171+0800 I STORAGE  [WT OplogTruncaterThread: local.oplog.rs] WiredTiger record store oplog truncation finished in: 3ms
2019-04-16T21:02:44.165+0800 I STORAGE  [WT OplogTruncaterThread: local.oplog.rs] WiredTiger record store oplog truncation finished in: 39ms
2019-04-16T21:02:46.361+0800 I REPL     [replexec-328] Scheduling catchup takeover at 2019-04-16T21:03:16.361+0800
2019-04-16T21:02:47.753+0800 I REPL     [replexec-327] Canceling catchup takeover callback
2019-04-16T21:02:54.362+0800 I REPL     [replexec-332] Scheduling catchup takeover at 2019-04-16T21:03:24.362+0800
2019-04-16T21:02:55.753+0800 I REPL     [replexec-332] Canceling catchup takeover callback
2019-04-16T21:02:56.418+0800 I ASIO     [RS] Ending connection to host 192.168.0.99:3717 due to bad connection status; 1 connections to that host remain open
2019-04-16T21:02:56.418+0800 I REPL     [replication-204] Restarting oplog query due to error: HostUnreachable: error in fetcher batch callback :: caused by :: Connection was closed. Last fetched optime (with hash): { ts: Timestamp(1555419775, 854), t: 5 }[-6290649860941869225]. Restarts remaining: 1
2019-04-16T21:02:56.418+0800 I REPL     [replication-204] Scheduled new oplog query Fetcher source: 192.168.0.99:3717 database: local query: { find: "oplog.rs", filter: { ts: { $gte: Timestamp(1555419775, 854) } }, tailable: true, oplogReplay: true, awaitData: true, maxTimeMS: 2000, batchSize: 13981010, term: 5, readConcern: { afterClusterTime: Timestamp(1555419775, 854) } } query metadata: { $replData: 1, $oplogQueryData: 1, $readPreference: { mode: "secondaryPreferred" } } active: 1 findNetworkTimeout: 7000ms getMoreNetworkTimeout: 10000ms shutting down?: 0 first: 1 firstCommandScheduler: RemoteCommandRetryScheduler request: RemoteCommand 4558836 -- target:192.168.0.99:3717 db:local cmd:{ find: "oplog.rs", filter: { ts: { $gte: Timestamp(1555419775, 854) } }, tailable: true, oplogReplay: true, awaitData: true, maxTimeMS: 2000, batchSize: 13981010, term: 5, readConcern: { afterClusterTime: Timestamp(1555419775, 854) } } active: 1 callbackHandle.valid: 1 callbackHandle.cancelled: 0 attempt: 1 retryPolicy: RetryPolicyImpl maxAttempts: 1 maxTimeMillis: -1ms
2019-04-16T21:02:56.418+0800 I NETWORK  [conn205] end connection 192.168.0.99:37808 (21 connections now open)
2019-04-16T21:02:56.418+0800 I ASIO     [Replication] Ending connection to host 192.168.0.99:3717 due to bad connection status; 0 connections to that host remain open
2019-04-16T21:02:56.418+0800 I ASIO     [RS] Connecting to 192.168.0.99:3717
2019-04-16T21:02:56.418+0800 I REPL_HB  [replexec-328] Error in heartbeat (requestId: 4558835) to 192.168.0.99:3717, response status: HostUnreachable: Connection was closed
2019-04-16T21:02:56.418+0800 I REPL     [SyncSourceFeedback] SyncSourceFeedback error sending update to 192.168.0.99:3717: HostUnreachable: Connection was closed


```