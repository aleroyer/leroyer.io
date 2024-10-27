---
title: "Improving network latency of DaemonSet in Kubernetes with Internal Traffic Policy"
date: 2024-10-27
draft: false
---

At our company, we primarily operate on bare-metal infrastructure, which provides us with very low network
latency within a zone or datacenter. When we began migrating services to Kubernetes (also running on bare-metal),
we needed to ensure that certain daemons running on each bare-metal server were also present on Kubernetes nodes
in a Kubernetes-native manner. To achieve this, we utilized DaemonSets, which are designed for this purpose.

In this article, we will focus on a specific service we use, [mcrouter](https://github.com/facebook/mcrouter),
a proxy that enhances [memcached](https://memcached.org/)'s reliability by facilitating read/write operations to
multiple memcached servers.

> Note: Modern versions of memcached include a similar feature with a
> [built-in proxy](https://docs.memcached.org/features/proxy/) that offers better support. We plan to
> replace mcrouter with this built-in proxy in the future.

## A Brief History: From `127.0.0.1` to Unix Sockets

On our non-Kubernetes servers, mcrouter was configured to listen on `127.0.0.1:11211`. This setup was
advantageous for several reasons:
- Mcrouter was not exposed to internal or external networks
- The configuration for applications using memcached remained consistent across all environments
- There was no latency between the application and mcrouter, as the communication occurred over the
  loopback interface, staying within the host.

However, in 2019, with Kubernetes 1.16, there was no **Internal Traffic Policy** feature available. This
necessitated a change in how mcrouter was exposed to applications while maintaining low latencies. We decided
to switch to a Unix socket with a [`hostPath`](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath) mount.

> Why is low latency important? Memcached access needs to be completed within a few milliseconds; otherwise,
> caching becomes ineffective in a high-traffic environment. Generally, our applications require only 1 or 2
> milliseconds to receive a response from memcached.

Although this approach was not ideal, it was the only way to ensure that a workload on `node A` would use the
mcrouter instance on `node A`. Due to the way Kubernetes Services functioned at the time, it was impossible to
guarantee that an application on `node A` would connect to `node A`'s mcrouter instead of `node B`'s.

We could have employed alternative methods, such as retrieving the DaemonSet's Pod IP and transmitting this
information back to the applications. While this might have been a better solution, we opted to stick with
the Unix socket approach.

While Unix sockets offer good performances, they come with several drawbacks:
- Applications must be reconfigured to use a Unix socket instead of TCP
- Each deployment must mount the same host path as the mcrouter daemonset
- Rolling out the daemonset results in downtime or errors for applications using the socket, as it is challenging to have two sockets existing in parallel

## Kubernetes 1.26, Internal Traffic Policy as stable

In Kubernetes 1.26, Internal and External traffic policies have reached stable status.

According to the [official documentation](https://kubernetes.io/docs/reference/networking/virtual-ips/#traffic-policies) on internal traffic policy:

> You can set the `.spec.internalTrafficPolicy` field to control how traffic from internal sources is routed.
> Valid values are `Cluster` and `Local`. Set the field to `Cluster` to route internal traffic to all ready
> endpoints and `Local` to only route to ready node-local endpoints. If the traffic policy is Local and there
> are no node-local endpoints, traffic is dropped by kube-proxy.

This enhancement allows us to configure Kubernetes to route traffic from `node A` exclusively to
`node A`'s DaemonSet pod, thereby reducing network hops and latency.

Here is an example of how this configuration would look in a service endpoint definition in YAML:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mcrouter
  namespace: core
spec:
  selector:
    app: mcrouter
  ports:
    - protocol: TCP
      port: 11211
      targetPort: 11211
  internalTrafficPolicy: Local
```

With this definition, apps can reach my mcrouter service with this url `mcrouter.core:11211`.

However, there are two important considerations before implementing this:
- **CNI Support**: Your Cluster Network Interface (CNI) must support this specification if you have replaced
  `kube-proxy`. This requirement is not explicitly highlighted in the documentation.
- **Pod Readiness**: If the Pod on a node is in a `NotReady` state, traffic will be dropped for all requests
  from this node.

### Ensuring CNI Support if `kube-proxy` is replaced

Refer to your CNI's documentation to verify support for this feature. In our clusters, we use [Cilium](https://cilium.io/) [without kube-proxy](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#kubernetes-without-kube-proxy).
Fortunately, [Cilium supports this directive](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#internal-traffic-policy):
> Similar to `externalTrafficPolicy` described above, Cilium’s eBPF kube-proxy replacement supports `internalTrafficPolicy`, which translates the above semantics to in-cluster traffic.

### Handling Pod `NotReady` State

A similar issue arises as with our current DaemonSet implementation using Unix sockets. If we restart the pod,
traffic will be dropped, leading to errors in applications.

How can we overcome this?

## Changing the default updateStrategy parameters of your DaemonSet

In Kubernetes, a DaemonSet use the `RollingUpdate` update strategy by default [but with a twist](https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/#daemonset-update-strategy):
> With `RollingUpdate` update strategy, after you update a DaemonSet template, old DaemonSet pods will be killed,
> and new DaemonSet pods will be created automatically, in a controlled fashion. At most one pod of the DaemonSet
> will be running on each node during the whole update process.

This is to prevent two DaemonSet running in parallel on a same host.

In our case, mcrouter serves as a proxy to abstract how clients connect to our cache cluster. Having two daemons running in parallel is not an issue if they are not using Unix sockets. With TCP, this is not a problem at all, as each pod will have a different IP address.

By default, the `updateStrategy` set `.spec.updateStrategy.rollingUpdate.maxUnavailable` to `1`, and `.spec.updateStrategy.rollingUpdate.maxSurge` to `0`. We simply need to inverse the logic here to allow two DaemonSet running on the same host during a rolling update:

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0  # this prevent killing the old pod
      maxSurge: 1        # this create the new pod before killing the old one
```

## Conclusion

With these changes, we achieve the following:
- Establish a consistent service endpoint for all our applications in Kubernetes (`mcrouter.core:11211`)
- Ensure that each application uses its local DaemonSet pod, avoiding unnecessary cluster network traffic
- Enhance the availability of our DaemonSet during restarts

## But wait, there's more

We have seen how to change service traffic routing between `Cluster` and `Local`, which is beneficial for DaemonSets. However, this approach is not suitable for Deployments, as it would require using Affinity to ensure applications are scheduled on the same host as their dependencies, which is not truly Kubernetes-native.

Fortunately, there is a promising new feature, currently in beta in Kubernetes 1.31, called [Traffic Distribution](https://kubernetes.io/docs/reference/networking/virtual-ips/#traffic-distribution). This feature allows you to prefer using a local pod if available, instead of always sending requests across the cluster network. This is particularly useful for improving communication between microservices that require low latencies.