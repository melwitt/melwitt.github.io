## How scheduling works in Nova

Scheduling in [Nova](https://docs.openstack.org/nova) happens in a few steps. I will attempt to explain those steps here.

---

### Pre-filtering with Placement

The first step in scheduling is a call to the [Placement](https://docs.openstack.org/placement) API with the requested vcpus, memory, disk, and traits. Placement will perform an efficient database query to select resource providers (compute hosts) that have capacity to fulfill the request. Placement will return at most [`[scheduler]max_placement_results`](https://docs.openstack.org/nova/latest/configuration/config.html#scheduler.max_placement_results) resource providers from the Nova scheduler configuration. The setting defaults to `1000` results that are in an undefined but determinstic order. If random ordering is desired, set [`[placement]randomize_allocation_candidates`](https://docs.openstack.org/placement/latest/configuration/config.html#placement.randomize_allocation_candidates) to `True` in the Placement service configuration.

### Nova scheduler filters

After the list of compute host candidates is prefiltered and returned from Placement, they are run through the scheduler filters configured by [`[scheduler]enabled_filters`](https://docs.openstack.org/nova/latest/configuration/config.html#filter_scheduler.enabled_filters) in the Nova scheduler configuration. Note that any filters in `[scheduler]enabled_filters` must also be in [`[scheduler]available_filters`](https://docs.openstack.org/nova/latest/configuration/config.html#filter_scheduler.available_filters), which defaults to a list of all the filters that are included in Nova. The filters run in the same order as they appear in `[scheduler]enabled_filters`. So, to get the most efficient performance, it is best to order the filters such that the most restrictive filters are first. This will reduce the candidate set more substantially during the earlier filter runs so that later filters will run on significantly fewer candidates.

### Nova scheduler placement pattern

The scheduler generally defaults to a “pack” pattern where compute hosts are filled to maximum possible capacity, or “depth first”, before moving on to the next compute host. This is evidenced by the Nova scheduler [`[filter_scheduler]host_subset_size`](https://docs.openstack.org/nova/latest/configuration/config.html#filter_scheduler.host_subset_size) default setting of `1`. This means that after the list of candidate compute hosts is determined by the Placement pre-filtering step and the Nova scheduler filtering step, `host_subset_size` compute hosts will be randomly chosen from the candidate list. The order of the candidate list is undefined but deterministic. This includes compute host candidates that all have the same scheduling weight.

In order to achieve more of a “spread” pattern where compute hosts are filled more randomly, or “breadth first”, the `host_subset_size` and [`[filter_scheduler]shuffle_best_same_weighed_hosts`](https://docs.openstack.org/nova/latest/configuration/config.html#filter_scheduler.shuffle_best_same_weighed_hosts) Nova scheduler configuration settings can be adjusted to increase randomness of the compute host candidate list choice.

### Collision handling

“Most” resources are claimed before a request is forwarded to a chosen compute host, but in the case of resources that are evaluated and claimed after landing on a compute host (example: NUMA topology), it is possible for multiple parallel requests to race to the same compute host (recall the default undefined but deterministic ordering of the compute host candidate list). When this occurs, one request will “win” and claim the resources on the compute host while the other requests will be rejected and go back to the scheduler for rescheduling. The number of times rescheduling will be attempted for a given request is determined by the [`[scheduler]max_attempts`](https://docs.openstack.org/nova/latest/configuration/config.html#scheduler.max_attempts) Nova scheduler configuration setting. This defaults to `3` scheduling attempts per request. If you are in a situation where your requests often race to the same compute host and you cannot leverage the randomization configuration settings in the previous section, you can increase your `max_attempts` setting to allow for more rescheduling attempts before `NoValidHost` is raised.

### Troubleshooting NoValidHost

When a server create or move request fails with `NoValidHost`, it means that the scheduling process could not find a compute host that meets the requirements of the request. To get more detail about why scheduling the server failed, you'll want to look at the logs for the `nova-scheduler` service. Usually this file is named `nova-scheduler.log`.

To locate the failed request, you can first search for the UUID of the server that failed to schedule, for example. On that log line, there will also be a [Request ID](https://docs.openstack.org/api-guide/compute/faults.html#tracking-errors-by-request-id) which can be used the trace the request across services. The Request ID will generally be the best way to trace all of the log messages related to the request of interest.

#### Got no allocation candidates from the Placement API

When you find the ERROR, look at log messages immediately before it. If you see the following log message (which is at level INFO):

```console
Got no allocation candidates from the Placement API. This could be due to insufficient resources or a temporary occurence as compute nodes start up.
```

it means that during the Placement pre-filtering step of scheduling, Nova asked Placement for compute hosts meeting the requirements of the request, but Placement did not find any. The next step in this case is to look at the Placement service logs. Usually this file is named `placement-api.log`.

At this point, there is a fair chance you will not be able to find more specific information about the failure if the `nova-scheduler` and `placement-api` services are not configured to log at level DEBUG. It is ideal to temporarily set the log level for these two services to DEBUG and repeat the failed the request to gather more data. You can configure services to log at level DEBUG by setting the [`[DEFAULT]debug`](https://docs.openstack.org/nova/latest/configuration/config.html#DEFAULT.debug) configuration option to `True` in the `nova.conf` and `placement.conf` files. Restart or send the HUP signal to the `nova-scheduler` and `placement-api` services to make them reload their configurations.

Once you have logs for the failed request at level DEBUG, you will see a log message for each step in the filtering process. If you got the "no allocation candidates from the Placement API" message in the `nova-scheduler` log, the Placement service log should now show a DEBUG log message for each resource class it filtered. There should be a message for "found no providers" that shows the resource for which Placement has no capacity.

#### Got allocation candidates from the Placement API

If you did not see the "no allocation candidates" INFO log message in the `nova-scheduler` log, it means that scheduling failed *after* the Placement API returned a list of potential hosts. Now you will want to look further in the `nova-scheduler` log at level DEBUG. For each scheduler filter, there will be a log message showing how many hosts were returned after running that filter. Log messages will look similar to this, they may be a bit different depending on what version of Nova you are running:

```console
Starting with N host(s)
Filter AvailabilityZoneFilter returned 1 host(s)
Filter ComputeFilter returned 1 host(s)
Filter ComputeCapabilitiesFilter returned 1 host(s) 
Filter ImagePropertiesFilter returned 1 host(s)
Filter ServerGroupAntiAffinityFilter returned 1 host(s)
Filter ServerGroupAffinityFilter returned 1 host(s)
Filter SameHostFilter returned 1 host(s) 
Filter DifferentHostFilter returned 1 host(s) 
Filtered [$list_of_compute_hosts_after_filtering]
```

In the case of a failed scheduling request, the last message in the logs for that request will say "returned 0 host(s)", showing which filter eliminated all of the potential compute hosts.
