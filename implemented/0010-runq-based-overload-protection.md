# Run queue based overload protection.

## Changelog
* 2021-03-11: @qzhuyan Initial Draft
* 2021-04-08: @qzhuyan Don't hibernate process when overloaded.

## Abstract

Run queue based overload protection for EMQX.

EMQX needs some mechanism to cool down the node when the node is overloaded.

Runq (Erlang VM run queue) is a performance critical metric.
It is a sign of overloading when the runq number is greater than the number of schedulers for a long period.

This document describes a mechanism how EMQX can cool down itself by monitoring runq.

## Motivation

Some user reports EMQX runs at 100% cpu load and etop trace shows the runq of a 2 cores node is around 100
and lasts more than 5 mins until the user kicks in then restarts the nodes.

Ideally, the runq should not be greater than the number of schedulers. The runq can grow
for a short period of time to handle peak traffic but should not stay high for longer than 1 mins.

Erlang is soft real-time system, a long run queue can cause time-critical processes not getting scheduled on time that may
lead timeout in upper layer's protocol. High CPU usage may also prevent users from login for O&M operations and leave nodes in zombie state.

EMQX needs some mechanism to cool down itself before it runs into the situation above.

And most importantly, operators need to get notified that the system is overloaded to take actions such as scale up the node/cluster.

## Design

### Run queue monitoring and flagging of long runq

An isolated process runs under top supervisor with high scheduling priority to poll the runq value every 3 secs.

If the runq is longer than 10x (number of online schedulers)
for last 5 polls, it should:

1. Raise the overload flag by registering a process name such like '__long_runq__'

1. Monitoring systems should be notified such as logging and metrics counter updates.

1. Alert should be triggered, if the flag is not cleared after [X timer] mins.

### When node need cooldown

Each application of EMQX should decide for itself how to deal with the overload.

The overload flag should be checked in all performance-critical code paths and react on it.

Suggested actions:

- Reject new connections to keep the existing connections alive.

  consequences:

    New Client would retry and may get redirected by load balancer to other nodes in the cluster.

- Defer processing of non-real-time requests such as Web UI requests.

  consequences:

    Web UI would be slow to respond and operator have to wait.

- Stop spawning more parallel workers.

  consequences:
    might hurt latency.

- Do not trigger active GC.

- [Unclear] Redirect requests to other nodes.

  Wonder if on protocol level there is such spec for this.

- Update backend status for polling from load balancer.

  consequences:

    Load balancer could lower the weight of this node in pool
    even remove the node from the pool for new connections.

- Different QoS messages would be handled differently.

  Such as drop low priority messages.
  
- Stop hibernating processes.

  gen_server should prefer not going to hibernate state.

### When runq is back to normal.

'__long_runq__' should be unregistered.


## Configuration Changes

1. disable/enable overload protection

1. The [timer X]

1. Configuration for overload handling in different APPs.

## Backwards Compatibility

N/A

## Document Changes

1. Description of overload protection.

1. Operation

1. Monitoring and Alerting

## Testing Suggestions

Apart from regular performance tests, overload tests should be performed with at least 5X of regular rate.

## Declined Alternatives

N/A
