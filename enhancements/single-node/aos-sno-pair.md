---
title: aos-sno-pair
authors:
- @mshitrit
  reviewers:
- TBD
  approvers:
- TBD

creation-date: 2021-07-12
last-updated: 2021-07-12
status: implementable
---

# Atomic OpenShift Single-Node-Pair

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary
This enhancement describes setting up a two nodes cluster with a reduced footprint which will serve edge cases.

## Motivation

Some companies have a need for a highly available container management solution that fits within a reduced footprint.
- The hardware savings are significant for customers deploying many remote sites (kiosks, branch offices, restaurant chains, etc), most notably for edge computing and RAN specifically.
- The physical constraints of some deployments prevents more than two nodes (planes, submarines, satellites, and also RAN).
- Some locations will not have reliable network connections or limited bandwidth (once again submarines, satellites, and disaster areas such as after hurricanes or floods)
  
### Goals

Provide the ability for all-in-one OpenShift clusters to operate as a **predefined** pair in an **active-active** configuration, detect when its peer has died and **automatically** take over its workloads after ensuring it is safe to do so.

### Non-Goals

- Management as a single entity (pod/namespace replication).
- **Replication of application data** between appliances.
- DR for services in clusters with **more than a single node.**
- DR for services running on **two or more all-in-one clusters.**
- Enable services uptime during a **remote cluster upgrade process.**

## Proposal

Assuming workloads that require some form of coordination between the two nodes (either at-most-one semantics or simply to reduce machine load during normal operation):

- Pacemaker and Corosync from the RHEL-HA cluster stack are deployed [in Pods](https://github.com/beekhof/installer/blob/aio/pod-cluster.yaml.in) on both OCP clusters, and they are configured to form a two node cluster.
- The Pacemaker cluster is configured so that on peer failure the cluster can reliably ensure that the workload on an unhealthy peer is stopped before starting the workload on the healthy peer. Configuration [flexibility](https://drive.google.com/file/d/1aw_78490rTQGsxEFcZytMe8lasZErLMm/view?usp=sharing) is provided here to allow for the anticipated variety of customer requirements.
- One or more identical Kubernetes Deployments are created on both OCP clusters, and a Pacemaker [Resource Agent](https://github.com/beekhof/pcmk/blob/master/k8sDeployment) is used to manage the replica count of these Deployments in order to ensure the singleton workload instance is always running on a healthy peer.
- Each Kubernetes Deployment can be configured to prefer a particular OCP cluster in order to make effective use of the hardware during normal operation.


### User Stories
TBD

### Implementation Details/Notes/Constraints

#### Normal Operation
Two nodes that communicate with each other and co-ordinate such that k8s Deployments are scaled up on at most one peer.
The RHEL-HA cluster performs periodic health checking of Deployments at two levels, frequently checking the number of requested replicas is correct, but also checking the number of active replicas at a lower frequency in order to capture additional soft-failure scenarios where the scheduler is unable to satisfy the replica count.

It is imagined that most Deployments would be scaled between 0 and 1 but the minimum and maximum counts are admin configurable for added flexibility.

Rather than creating a one-size-fits-all 2 node architecture, supportable customer deployments must include a minimum number of [redundancy options](https://drive.google.com/file/d/1aw_78490rTQGsxEFcZytMe8lasZErLMm/view?usp=sharing) based on the constraints of their environment.

#### Network Redundancy
Bonding or Knet can be configured with redundant network paths in environments where this is possible.  This helps ensure that the RHEL-HA cluster does not attempt recovery until all communication paths between the peers have been exhausted.

Care must be taken to ensure that paths are truly independent and avoid single points of failure (SPoF). Crossover cables are often the simplest way of meeting this requirement and are therefore highly encouraged.

#### Fencing
In the event of a peer failure, any **workloads requiring singleton semantics** (such as StatefulSets or kubevirt VMs) require the existing node to be put into a safe state (stopped or otherwise known to be not running the workloads) before they can be started on the surviving peer.  The traditional term for this is “**fencing**” and it is an **optional** part of RHEL-HA clusters.

Customer environments without any singleton workloads may not require fencing, but need to be comfortable with the possibility of workloads being active on both peers at the same time.

For customers that do have singleton workloads, the RHEL-HA stack supports a wide variety of BMCs including those conforming to the IPMI and RedFish standards, as well as SSH and libvirt for testing.

It also supports indirect fencing methods like poison-pill which can optionally make use of shared storage for out-of-band signalling and heartbeating where available.

#### Quorum handling
_The "two node cluster" is a use case that requires special consideration.  With a standard two node cluster, each node with a single vote, there are 2 votes in the cluster. Using the simple majority calculation (50% of the votes + 1)  to calculate quorum, the quorum would be 2.  This means that the both nodes would always have to be alive for the cluster to be quorate and operate._
<br>Source: man votequorum

For environments in which customers will allow it, Corosync also ships a lightweight quorum arbitrator that can be run locally or in the cloud on any non-cluster node.  In such cases, no special quorum handling is required and careful placement of the arbitrator also ensures that the surviving peer is reachable by its intended clients.

Where this is not possible, there are existing capabilities that we can leverage by enabling the _Corosync two_node, last_man_standing, and wait_for_all_ options (See: man votequorum) and configuring a fencing delay in Pacemaker.

In combination, these options provide the following behaviour:
- Both sides retain quorum in the event of a network split.
- Service recovery is normally predicated on successful fencing.
- When the fenced node reboots, it will not be able to obtain quorum until it has seen its peer (preventing an infinite fencing cycle).
- To avoid both sides fencing each other at the same time, fencing is typically configured to occur after a delay on one of the nodes. That node will perform recovery in the event of a true peer failure, but will always lose in the response to a pure network failure.

### Risks and Mitigations
- A two node etcd cluster does not have quorum whenever the communication between the peers is interrupted (by network or peer failure).
- From inside a typical 2 node cluster, network and peer failures look identical.

## Design Details

### Open Questions

- Do we really need active/active, or could active/passive be sufficient?
  - Reasons mentioned in favor:
    - It ensures the secondary node works when you need it. Not only it is working, but the whole list of dependencies it carries with it (DNS...)
    - Performance.
    - Cost. The secondary node is not just sitting there, wasting resources.
    - LESS maintenance procedures. You don't need to failover/failback on upgrades, for example.
- Do we really need automated failover, or is manual confirmation sufficient?
- How much overlap is there between the goals as laid out here and Telco RAN requirements?


### Test Plan

### Graduation Criteria

#### Examples

#### Dev Preview -> Tech Preview

#### Tech Preview -> GA

#### Removing a deprecated feature

### Upgrade / Downgrade Strategy

### Version Skew Strategy

## Implementation History

## Drawbacks

## Alternatives

1. Support an active/passive configuration with 1 master + 1 worker.  Workloads stay running on the worker if the master dies (unless they require rescheduling before the master is recovered), but the master takes over if the worker dies.
2. Run with a single active etcd copy, but make use of [etcd learners](https://github.com/etcd-io/etcd/blob/master/Documentation/learning/design-learner.md) to provide a non-live replica that can be promoted in the event the active copy fails.
   Learners will gain self-promotion abilities for automated failover in etcd 3.5.  Would require additional engineering (either in etcd or some component driving it) for the learner to distinguish between a network and leader failures, and/or ensure the leader is no longer accepting writes before promoting itself.
3. Replicate the logic for safely handling two node failure scenarios (including fencing) in the cluster-etcd-operator so that it can force etcd into a quorate state.  The standard event loop would stall without quorum, so there would be significant engineering and QE effort required to achieve this.
4. Run the existing RHEL-HA cluster stack in a container on both “clusters” but use it to manage etcd directly (instead of the cluster-etcd-operator), using etcd commands to force it into a quorate state.
   Could optionally incorporate [etcd learners](https://github.com/etcd-io/etcd/blob/master/Documentation/learning/design-learner.md) but still would represent a very different management model (RHEL-HA vs. operator) for a core component than is used for every other deployment type.