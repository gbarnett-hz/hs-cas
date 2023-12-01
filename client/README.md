---
title: Client Conneciton Preference; Slow Member
---

# Setup

- 5 CP Member Group
- Each CP member is hosted on distinct c5.4xlarge
- Client hosted on distinct c5.4xlarge
- Each JVM Xms and Xmx 8G; Java 17, rest are defaults
- Hazelcast 5.3.7-SNAPSHOT (latest of 5.3.z + Vassilis client filter)
- All VMs in `cluster` placement group
- At most we put a 10ms network latency on one member, i.e. the 'slow' member

# Operation

An _op_ as referred below is:

```java
String observed = atomicReference.get();
String newValue;
do {
    newValue = observed;
} while (!atomicReference.compareAndSet(observed, newValue));
```

The `compareAndSet` always succeeds; the value of `atomicReference` never changes. The reason for
this is so that you compute only the cost of the CP operations rather than any other auxiliary
computation. Before the run begins there's a 10k warmup cycle of that op; the test itself is 500k
ops.

# Leadership

One member is given leadership priority. Before any test is run we wait for the member with
leadership priority to assume leadership of the `default` CP Group. Smart routing is disabled so we
can target a specific member. When smart routing isn't disabled we are showing semantics as-is today
without taking this additional step, exception is use of revised load balancer logic.

# Scenarios

![](topology.svg)

| Scenario                          | Network | Duration(s) | Op/s        | Description                                                                            |
| --------------------------------- | ------- | ----------- | ----------- | -------------------------------------------------------------------------------------- |
| client-leader                     | (a)     | 385         | ~1299       | client talks directly with leader of CP group                                          |
| client-follower                   | (a)     | 541         | ~924        | client talks to follower of CP group                                                   |
| client-leader-1follower10ms       | (b)     | 390         | ~1282       | client talks directly with leader of CP group; one follower has a 10ms network latency |
| client-follower-1follower10ms     | (b)     | 545         | ~917        | client talks to a single follower; one follower has a 10ms network latency             |
| client-follower10ms-1follower10ms | (b)     | FAIL        | FAIL        | client talks to a single follower; that follower has a 10ms network latency            |
| client-lb                         | (a)     | 478         | ~1046       | client lb with all members whitelisted; should be same default RR                      |
| client-lb-10ms                    | (b)     | 456         | ~1096       | client lb with all members whitelisted but the member with 10ms network latency        |
| client-smart-10ms                 | (b)     | in-progress | in-progress | client using smart routing                                                             |

## Observations

- _client-follower10ms-1follower10ms_ this is the worse case scenario, it took around an hour and
  then the client gave up.
- _client-follower10ms-1follower10ms_ the op/s is the very minimum: it could be much more. The test
  failed to complete.
- The _client-lb_ scenario is faster than _client-follower_ because 1/5 requests (I assume) go to
  the leader which is fastest; by contrast to _client-follower_ which never utilises the leader.
  _client-lb-10ms_ is faster than _client-lb_ because now 1/4 requests go to leader.
