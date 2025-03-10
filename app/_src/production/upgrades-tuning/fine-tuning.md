---
title: Performance fine-tuning
content_type: reference
---

## Reachable services

By default, when transparent proxying is used, every data plane proxy follows every other data plane proxy in the mesh.
With large meshes, usually, a data plane proxy connects to just a couple of services in the mesh.
By defining the list of such services, we can dramatically improve the performance of {{site.mesh_product_name}}.

The result is that:
* The control plane has to generate a much smaller XDS configuration (just a couple of Clusters/Listeners etc.) saving CPU and memory
* Smaller config is sent over a wire saving a lot of network bandwidth
* Envoy only has to keep a couple of Clusters/Listeners which means much fewer statistics and lower memory usage.

Follow the {% if_version lte:2.1.x %}[transparent proxying](/docs/{{ page.version }}/networking/transparent-proxying){% endif_version %}{% if_version gte:2.2.x %}[transparent proxying](/docs/{{ page.version }}/production/dp-config/transparent-proxying/){% endif_version %} docs on how to configure it.

{% if_version gte:2.5.x %}
## Config trimming by using MeshTrafficPermission

{% warning %}
1. This feature only works with [MeshTrafficPermission](/docs/{{ page.version }}/policies/meshtrafficpermission),
   if you're using [TrafficPermission](/docs/{{ page.version }}/policies/traffic-permissions) you need to migrate to MeshTrafficPermission,
   otherwise enabling this feature could stop all traffic flow.
2. Due to [a bug](https://github.com/kumahq/kuma/issues/6589) [ExternalServices](/docs/{{ page.version }}/policies/external-services) won't work without Traffic Permissions without [Zone Egress](/docs/{{ page.version }}/production/cp-deployment/zoneegress), if you're using External Services you need to keep associated TrafficPermissions.
   {% endwarning %}

Starting with release 2.5 the problem stated in [reachable services](#reachable-services) section
can be also mitigated by defining [MeshTrafficPermissions](/docs/{{ page.version }}/policies/meshtrafficpermission) and [configuring](/docs/{{ page.version }}/documentation/configuration) a **zone** control plane with `KUMA_EXPERIMENTAL_AUTO_REACHABLE_SERVICES=true`.

Switching on the flag will result in computing a graph of dependencies between the services
and generating XDS configuration that enables communication **only** with services that are allowed to communicate with each other
(their [effective](/docs/{{ page.version }}/policies/targetref/#merging-configuration) action is **not** `deny`).

For example: if a service `b` can be called only by service `a`:

```yaml
apiVersion: kuma.io/v1alpha1
kind: MeshTrafficPermission
metadata:
  namespace: kuma-system
  name: mtp-b
spec:
  targetRef:
    kind: MeshService
    name: b
  from:
    - targetRef:
        kind: MeshService
        name: a
      default:
        action: Allow
```

Then there is no reason to compute and distribute configuration of service `b` to any other services in the Mesh since (even if they wanted)
they wouldn't be able to communicate with it.

{% tip %}
You can combine `autoReachableServices` with [reachable services](#reachable-services), but **reachable services** will take precedence.
{% endtip %}

Sections below highlight the most important aspects of this feature, if you want to dig deeper please take a look at the [MADR](https://github.com/kumahq/kuma/blob/master/docs/madr/decisions/031-automatic-rechable-services.md#automatic-reachable-services).

### Supported targetRef kinds

The following kinds affect the graph generation and performance:
- all levels of `MeshService`
- [top](/docs/{{ page.version }}/policies/targetref/#target-resources) level `MeshSubset` and `MeshServiceSubset` with `k8s.kuma.io/namespace`, `k8s.kuma.io/service`, `k8s.kuma.io/port` tags
- [from](/docs/{{ page.version }}/policies/targetref/#target-resources) level `MeshSubset` and `MeshServiceSubset` with all tags

If you define a MeshTrafficPermission with other kind, like this one:

{% policy_yaml meshtrafficpermission_other_kind %}
```yaml
type: MeshTrafficPermission
mesh: default
name: mtp-mesh-to-mesh
spec:
  targetRef:
    kind: MeshSubset
    tags:
      customTag: true
  from:
    - targetRef:
        kind: Mesh
      default:
        action: Allow
```
{% endpolicy_yaml %}

it **won't** affect performance.

### Changes to the communication between services

Requests from services trying to communicate with services that they don't have access to will now fail with connection closed error like this:

```sh
root@second-test-server:/# curl -v first-test-server:80
*   Trying [IP]:80...
* Connected to first-test-server ([IP]) port 80 (#0)
> GET / HTTP/1.1
> Host: first-test-server
> User-Agent: curl/7.81.0
> Accept: */*
>
* Empty reply from server
* Closing connection 0
curl: (52) Empty reply from server
```

instead of getting a `503` error.

```sh
root@second-test-server:/# curl -v first-test-server:80
*   Trying [IP]:80...
* Connected to first-test-server ([IP]) port 80 (#0)
> GET / HTTP/1.1
> Host: first-test-server
> User-Agent: curl/7.81.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 503 Service Unavailable
< content-length: 118
< content-type: text/plain
< date: Wed, 08 Nov 2023 14:15:24 GMT
< server: envoy
<
* Connection #0 to host first-test-server left intact
upstream connect error or disconnect/reset before headers. retried and the latest reset reason: connection termination/
```

### Migration

A recommended path of migration is to start with a coarse grain `MeshTrafficPermission` targeting a `MeshSubset` with `k8s.kuma.io/namespace` and then drill down to individual services if needed.

{% endif_version %}

## Postgres

If you choose `Postgres` as a configuration store for {{site.mesh_product_name}} on Universal,
please be aware of the following key settings that affect performance of {{site.mesh_product_name}} Control Plane.

* `KUMA_STORE_POSTGRES_CONNECTION_TIMEOUT` : connection timeout to the Postgres database (default: 5s)
* `KUMA_STORE_POSTGRES_MAX_OPEN_CONNECTIONS` : maximum number of open connections to the Postgres database (default: unlimited)

### KUMA_STORE_POSTGRES_CONNECTION_TIMEOUT

The default value will work well in those cases where both `kuma-cp` and Postgres database are deployed in the same datacenter / cloud region.

However, if you're pursuing a more distributed topology, e.g. by hosting `kuma-cp` on premise and using Postgres as a service in the cloud, the default value might no longer be enough.

### KUMA_STORE_POSTGRES_MAX_OPEN_CONNECTIONS

The more dataplanes join your meshes, the more connections to Postgres database {{site.mesh_product_name}} might need to fetch configurations and update statuses.

As of version 1.4.1 the default value is 50.

However, if your Postgres database (e.g., as a service in the cloud) only permits a small number of concurrent connections, you will have to adjust {{site.mesh_product_name}} configuration respectively.

## Snapshot Generation

{% warning %}
This is advanced topic describing {{site.mesh_product_name}} implementation internals
{% endwarning %}

The main task of the control plane is to provide config for dataplanes. When a dataplane connects to the control plane, the CP starts a new goroutine.
This goroutine runs the reconciliation process with given interval (1s by default). During this process, all dataplanes and policies are fetched for matching.
When matching is done, the Envoy config (including policies and available endpoints of services) for given dataplane is generated and sent only if there is an actual change.

* `KUMA_XDS_SERVER_DATAPLANE_CONFIGURATION_REFRESH_INTERVAL` : interval for re-generating configuration for Dataplanes connected to the Control Plane (default: 1s)

This process can be CPU intensive with high number of dataplanes therefore you can control the interval time for a single dataplane.
You can lower the interval scarifying the latency of the new config propagation to avoid overloading the CP. For example,
changing it to 5s means that when you apply a policy (like TrafficPermission) or the new dataplane of the service is up or down, CP will generate and send new config within 5 seconds.

For systems with high traffic, keeping old endpoints for such a long time (5s) may not be acceptable. To solve this, you can use passive or active [health checks](/docs/{{ page.version }}/policies/health-check) provided by {{site.mesh_product_name}}.

Additionally, to avoid overloading the underlying storage there is a cache that shares fetch results between concurrent reconciliation processes for multiple dataplanes.

* `KUMA_STORE_CACHE_EXPIRATION_TIME` : expiration time for elements in cache (1s by default).

You can also change the expiration time, but it should not exceed `KUMA_XDS_SERVER_DATAPLANE_CONFIGURATION_REFRESH_INTERVAL`, otherwise CP will be wasting time building Envoy config with the same data.

## Profiling

{{site.mesh_product_name}}'s control plane ships with [pprof](https://golang.org/pkg/net/http/pprof/) endpoints so you can profile and debug the performance of the `kuma-cp` process.

To enable the debugging endpoints, you can set the `KUMA_DIAGNOSTICS_DEBUG_ENDPOINTS` environment variable to `true` before starting `kuma-cp` and use one of the following methods to retrieve the profiling information:

{% tabs profiling useUrlFragment=false %}
{% tab profiling pprof %}

You can retrieve the profiling information with Golang's `pprof` tool, for example:

```sh
go tool pprof http://<IP of the CP>:5680/debug/pprof/profile?seconds=30
```

{% endtab %}
{% tab profiling curl %}

You can retrieve the profiling information with `curl`, for example:

```sh
curl http://<IP of the CP>:5680/debug/pprof/profile?seconds=30 --output prof.out
```
{% endtab %}
{% endtabs %}

Then, you can analyze the retrieved profiling data using an application like [Speedscope](https://www.speedscope.app/).

{% warning %}
After a successful debugging session, please remember to turn off the debugging endpoints since anybody could execute heap dumps on them potentially exposing sensitive data.
{% endwarning %}

{% if_version lte:2.1.x %}

## Kubernetes

### Kubernetes outbounds in central place

Configure `KUMA_EXPERIMENTAL_KUBE_OUTBOUNDS_AS_VIPS` to `true` to store the list of outbounds in ConfigMap that is used for VIPs of {{site.mesh_product_name}} DNS.
This way we don't repeat this information across all `Dataplane` objects which may improve a performance with a large number of data plane proxies.

You can enable this only after all instances of the control plane are updated to 1.6.0 or later.
This option will be the default behaviour in the next versions of {{site.mesh_product_name}}.
{% endif_version %}

{% if_version gte:2.3.x %}

### Kubernetes client

Kubernetes client uses client level throttling to not overwhelm kube-api server. In larger deployments, bigger than 2000 services in a single kubernetes cluster, number
of resources updates can hit this throttling. In most cases it's safe to increase this limit as kube-api has it's own throttling mechanism. To change client
throttling configuration you need to update config.

```yaml
runtime:
  kubernetes:
    clientConfig:
      qps: ... # Qps defines maximum requests kubernetes client is allowed to make per second.
      burstQps: ... # BurstQps defines maximum burst requests kubernetes client is allowed to make per second
```

### Kubernetes controller manager

{{site.mesh_product_name}} is modifying some Kubernetes resources. Kubernetes calls the process of modification reconciliation. Every resource has its own working queue, and control plane adds reconciliation tasks to that queue. In larger deployments, bigger than 2000 services in a single Kubernetes cluster, size of the work queue for pod reconciliation
can grow and slow down pods updates. In this situation you can change the number of concurrent pod reconciliation tasks, by changing configuration:

```yaml
runtime:
  kubernetes:
    controllersConcurrency:
      podController: ... # PodController defines maximum concurrent reconciliations of Pod resources
```

{% endif_version %}

## Envoy

### Envoy concurrency tunning

Envoy allows configuring the number of [worker threads ](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/intro/threading_model)used for processing requests. Sometimes it might be useful to change the default number of worker threads e.g.: high CPU machine with low traffic. Depending on the type of deployment, there are different mechanisms in `kuma-dp` to change Envoy’s concurrency level.

{% tabs envoy useUrlFragment=false %}
{% tab envoy Kubernetes %}

By default, Envoy runs with a concurrency level based on resource limit. For example, if you’ve started the `kuma-dp` container with CPU resource limit `7000m` then concurrency is going to be set to 7. It's also worth mentioning that concurrency for K8s is set from at least 2 to a maximum of 10 worker threads. In case when higher concurrency level is required it's possible to change the setting by using annotation `kuma.io/sidecar-proxy-concurrency` which allows to change the concurrency level without limits.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
spec:
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
      annotations:
        kuma.io/sidecar-proxy-concurrency: 55
[...]
```
{% endtab %}

{% tab envoy Universal %}

Envoy on Linux, by default, starts with the flag `--cpuset-threads`. In this case, cpuset size is used to determine the number of worker threads on systems. When the value is not present then the number of worker threads is based on the number of hardware threads on the machine. `Kuma-dp` allows tuning that value by providing a `--concurrency` flag with the number of worker threads to create.

```sh
kuma-dp run \
  [..]
  --concurrency=5
```

{% endtab %}
{% endtabs %}
