# Optimizing Cloud Foundry

<!-- TOC -->

- [Optimizing Cloud Foundry](#optimizing-cloud-foundry)
    - [Introduction](#introduction)
    - [IaaS performance guidelines for Cloud Foundry](#iaas-performance-guidelines-for-cloud-foundry)
        - [AWS](#aws)
            - [Networks](#networks)
            - [Instance types](#instance-types)
            - [The right resources](#the-right-resources)
    - [Optimizing Cloud Foundry's components](#optimizing-cloud-foundrys-components)
        - [CAPI](#capi)
            - [blobstore](#blobstore)
            - [cc_uploader, cloud_controller_clock, cloud_controller_ng, nsync, stager, tps](#cc_uploader-cloud_controller_clock-cloud_controller_ng-nsync-stager-tps)
        - [Consul](#consul)
            - [consul_agent](#consul_agent)
        - [Diego](#diego)
            - [auctioneer](#auctioneer)
            - [bbs](#bbs)
            - [rep](#rep)
            - [route_emitter](#route_emitter)
        - [Garden-runC](#garden-runc)
        - [Cloud Foundry](#cloud-foundry)
            - [doppler](#doppler)
            - [etcd](#etcd)
            - [gorouter](#gorouter)
            - [uaa](#uaa)
    - [Appendix A](#appendix-a)
    - [Appendix B](#appendix-b)

<!-- /TOC -->

## Introduction

Tuning Cloud Foundry is no small or menial task. Even more, tuning itself is deeply tied to the expected results and projections of the platform growth.

## IaaS performance guidelines for Cloud Foundry

To achieve a high performance platform, it is better to focus first on the right choice of IaaS resources and its configuration.

### AWS

#### Networks

a. **Use placement groups**: Placement groups is a way to group instances together in an AZ providing low latency, high throughput networking.
A common pattern is to deploy the all the possible runtime instances on their own placement group and then use another one for the runner VMs.

b. **Enhanced networking**: To enable a VM to be compatible with placement groups, it has to support "enhanced networking", a characteristic enabled for certain VM types only. More information [here](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html#concepts-placement-groups).

#### Instance types

a. **Choose specialized instances**: generic-type instances are very convenient, but some are not ideal for some Cloud Foundry components. Some jobs can have different hardware requirement and are prone to consume different types of resources.

See [Appendix A](#appendix-a) for a detailed list of CF components and recommended VM types.

#### The right resources

a. **ELB for Load Balancing**: ELB works very well to load balance connections in the DMZ.
> **Important**: ELB is capable of sustaining a great amount of connections over time. However, it is known to have issues with traffic spikes, ramping up new workers slowly.

b. **S3 for file storage**: S3 provides a flexible, highly scalable and well tuned file storage facility. It is a fantastic resource for configuring Cloud Foundry's Blobstore.

## Optimizing Cloud Foundry's components

Cloud Foundry comes with sensible default values for all of its components and jobs, so only a few properties are worth to be analyzed and modified. Many of these properties will also be adjusted according to the business rules and are more related to the behavior of the platform than to performance itself.

Here are the values that are worth considering analyze when deploying Cloud Foundry.

### CAPI

#### blobstore

```
nginx_workers_per_core: 2
```

This parameter specifies how many Nginx worker process per CPU the Blobstore will create.
Adjustment will be depending on the size of the processor of the Blobstore VM. Defaults to *2*.

#### cc_uploader, cloud_controller_clock, cloud_controller_ng, nsync, stager, tps

   For these jobs, see the properties [here](https://bosh.io/jobs/cloud_controller_clock?source=github.com/cloudfoundry/capi-release&version=1.31.0). Since this job is not resource consuming, the tuning has to be done according to the business rules.

   See [Appendix B](#appendix-b) for information on how to rate-limit requests to the CC's API.

### Consul

In Consul, server performance is critical for overall throughput and health of a Consul cluster. Servers are bound by:
* I/O bound for writes due to a sync to disk every time an entry is appended
* CPU bound for reads since they work from a fully in-memory data store.

It is recommended at least 3 instances of Consul, running on **m4.large** types or bigger. If possible, a specialized **c4.large** type is even better due to the optimization of the processor.
To cope with disk I/O, a provisioned IOPS disk can be used.

Based on experience, if the CPU reaches an **89%** of usage for a period of 7 minutes, Consul will start to degrade in performance and the cluster should be scaled.

#### consul_agent

    max_stale: 30s

Limits the time that requests can be stalled. This can be decreased if DNS resolution by Consul causes trouble.

    recursor_timeout: 5s

Specifies the timeout for Consul when querying a recursive upstream server. This value can be increased in case of low latency or slow Consul server.

### Diego

#### auctioneer

    cell_state_timeout: 1s 

This timeout is applied to limit the response time of the Cell State endpoint. If low performance is found, it can be increased to test the response of the Cells.

#### bbs

    max_open_connections: [required to be configured, no default]

This parameter can be increased in case that the platform has low performance in the container management times.

#### rep

    executor.container_inode_limit: 200000

Maximum enforced number of inodes (references to actual files) enforced by a Garden backend.
Raise if a container has to use more files.

    container_max_cpu_shares: 1024

How many CPU shares can a container use. Lower this value if LRPs are very lightweight

#### route_emitter

    sync_interval_in_seconds: 60
    
The period of time between routes synchronization with the GoRouter. If a fastest response in sync is needed, lower this value.

### Garden-runC

    default_container_grace_time: 5m

How long will it take to reap the idle containers. Depending on the applications deployed, can be extended if the application has valid idle periods; or diminished if more Containers per Cell are needed.

    cpu_quota_per_share_in_us: 100000
    
It is possible to request a maximum CPU limit proportional to the existing fair-share CPU limit (see Cell Rep's `container_max_cpu_shares`). The previous CPU limit just ensured that if two containers competed for CPU they would receive it in proportion to their requested CPU share (in the Cloud Foundry use case, this meant in proportion to their memory limit). This meant, for example, if only one container was active on a host it could potentially consume all of the host's resources.

The `cpu_quota_per_share_in_us` property, which has to be set in Cloud Foundry's deployment manifest, applies to all containers created on a host and enforces an additional maximum amount of CPU usage in each quota period (which has a default value of 100ms) in proportion to the existing shares limit.

**Example**: Since CF sets a container's CPU limit based on the memory limit, if `cpu_quota_per_share_in_us` is set to 100 on a single-core system then a 64MB container will be able to use a maximum of 6.4ms (64 * 100us = 6400us = 6.4ms) of CPU every 100ms.

### Cloud Foundry
    
#### doppler

Doppler cluster's instances can be increased as much as needed.
Depending on the drain buffer size configuration, memory will be the main factor to scale up a Doppler instance.  
If the client consuming Doppler's messages is fast enough, then adding new instances will depend on the amount of Cells and application instances running, as well as the types of logs sent.  
As an example, a Java stacktrace log can be hundreds of lines long. Each line will represent a single message, so Doppler's buffer can be filled up very quickly if the consumer is not fast enough to consume those messages.  
There is no recipe of formula to tune your Doppler cluster, but it is more a matter of watching logs for the message "`Dropped n envelopes`", where `n` is the amount of messages dropped by Doppler. The size of the buffer is configured by the following property:

    message_drain_buffer_size: 10000
    
The max amount of messages that Doppler can buffer before start dropping messages. Increasing this limit too much can fill up the VM memory very quickly.

#### etcd

**etcd** is a very disk intensive service. If the disk I/O cannot be satisfied with the usual GP2 disks in AWS, a specialized dedicated provisioned IOPS instance is recommended. If this is not enough and disk I/O is still an issue, it is possible to move **etcd** to a RAM disk. However, it is very likely that a specialized disk I/O type instance is more than enough.

#### gorouter

Dealing with **gorouter** tuning is more about the right amount of instances and the right metrics and threshold to watch to prevent latency and throughput degradation.
According to experience, when the processor of a **c4.large gorouter** instance reaches a **70%** during at least 5 minutes, latency will slightly increase, but throughput will remain steady.  
When the processor reaches **85%** then throughput will be affected as well and latency will increase significantly. To keep the routing performance steady and at a maximum, scale out the gorouter instances when they reach 70% of processor occupancy.

#### uaa

UAA is a very efficient software running in a JVM. To scale out the UAA, the processor and JVM memory consumption has to be watched.  
With a **m4.large** instance, if the processor reaches an **79%**, the latency of the responses will increase significantly.   
If memory of the JVM gets filled up, throughput will be highly degraded.   
So if the processor reaches a **79%** or a **95%** of occupied memory it is recommended to spin up a new UAA instance.

Some properties to have in mind:

    catalina_opts: -Xmx768m -XX:MaxMetaspaceSize=256m

Tomcat's JVM options. Usually is not necessary to modify these parameters. Scaling up the `Xmx` parameter can be useful when dealing with a high amount of requests per second.

    max_connections: 100 

The maximum number of connections in the DB pool. This can be increased in case of a high load, but will consume memory.

## Appendix A

Recommended VM types for a production environment.
> **IMPORTANT**: This is an approximation based on experience.
> You should always deploy the platform, run load tests and then optimize according to the results of those tests.

| Job               | VM Type   | Notes                                         | Placement Group |
| ----------------- | :-------- | :-------------------------------------------- | --------------- |
| api               | m4.large  |                                               | A               |
| consul            | c4.large  | Compute optimized                             | A               |
| nats              | t2.medium |                                               | N/A             |
| etcd              | i3.large  | Disk I/O optimized. Use i3.xlarge if possible | B               |
| nfs               | m4.large  |                                               | A               |
| uaa               | m4.large  |                                               | A               |
| stats             | t2.medium |                                               | N/A             |
| clock_global      | t2.medium |                                               | N/A             |
| api_worker        | t2.medium |                                               | N/A             |
| loggregator       | t2.medium |                                               | N/A             |
| doppler           | m4.large  |                                               | A               |
| trafficcontroller | t2.medium |                                               | N/A             |
| router            | c4.large  | Compute optimized                             | A               |
| diego cell        | r4.xlarge | Memory optimized                              | C               |
| diego brain       | t2.medium |                                               | N/A             |
| cc_bridge         | m4.large  |                                               | A               |
| route_emitter     | t2.medium |                                               | N/A             |
| compilation       | c4.xlarge | Provides extra fast compilation               | A               |

## Appendix B

To enable the CC's API throttling limits, you have to set the following properties:

```
cc.rate_limiter.enabled
cc.rate_limiter.general_limit
cc.rate_limiter.unauthenticated_limit
cc.rate_limiter.reset_interval_in_minutes
```

| Setting key                 | Possible values | Value meaning                                          |
| --------------------------- | --------------- | ------------------------------------------------------ |
| `enabled`                   | true/false      | This will enable/disable the rate_limiting feature     |
| `general_limit`             | Integer         | The number of requests limit for authenticated calls   |
| `unauthenticated_limit`     | Integer         | The number of requests limit for unauthenticated calls |
| `reset_interval_in_minutes` | Integer         | Time to reset the limits in minutes                    |


This will set the following headers in the HTTP response:

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 56
X-RateLimit-Reset: 1372700873
```

| Field name              | Value meaning                                                                                                                                       |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `X-RateLimit-Limit`     | The maximum number of attempts per UAA user, if a user is authenticated. The maximum number of attempts per IP address, if no user is authenticated |
| `X-RateLimit-Remaining` | The number of attempts remaining.                                                                                                                   |
| `X-RateLimit-Reset`     | The time remaining before the rate limit counter resets, in UTC epoch seconds                                                                       |

Once the user (when authenticated) or the IP (when unauthenticated) reaches the limit, the HTTP response will be `429: Too many requests`.
