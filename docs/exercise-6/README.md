# Exercise 6 - Perform traffic management

## Using rules to manage traffic

The core component used for traffic management in Istio is Pilot, which manages and configures all the Envoy proxy instances deployed in a particular Istio service mesh. It lets you specify what rules you want to use to route traffic between Envoy proxies, which run as sidecars to each service in the mesh. Each service consists of any number of instances running on pods, containers, VMs etc. Each service can have any number of versions (a.k.a. subsets). There can be distinct subsets of service instances running different variants of the app binary. These variants are not necessarily different API versions. They could be iterative changes to the same service, deployed in different environments (prod, staging, dev, etc.). Pilot translates high-level rules into low-level configurations and distributes this config to Envoy instances. Pilot uses three types of configuration resources to manage traffic within its service mesh: Virtual Services, Destination Rules, and Service Entries.

### Virtual Services

A [VirtualService](https://istio.io/latest/docs/reference/config/networking/virtual-service/#VirtualService) defines a set of traffic routing rules to apply when a host is addressed. Each routing rule defines matching criteria for traffic of a specific protocol. If the traffic is matched, then it is sent to a named [destination](https://istio.io/latest/docs/reference/config/networking/destination-rule/#DestinationRule) service (or [subset](https://istio.io/latest/docs/reference/config/networking/destination-rule/#Subset) or version of it) defined in the service registry.

### Destination Rules

A [DestinationRule](https://istio.io/latest/docs/reference/config/networking/destination-rule/#DestinationRule) defines policies that apply to traffic intended for a service after routing has occurred. These rules specify configuration for load balancing, connection pool size from the sidecar, and outlier detection settings to detect and evict unhealthy hosts from the load balancing pool. Any destination `host` and `subset` referenced in a `VirtualService` rule must be defined in a corresponding `DestinationRule`.

### Service Entries

A [ServiceEntry](https://istio.io/latest/docs/reference/config/networking/service-entry/#ServiceEntry) configuration enables services within the mesh to access a service not necessarily managed by Istio. The rule describes the endpoints, ports and protocols of a white-listed set of mesh-external domains and IP blocks that services in the mesh are allowed to access.

## The Guestbook app

In the Guestbook app, there is one service: guestbook. The guestbook service has two distinct versions: the base version (version 1) and the modernized version (version 2). Each version of the service has three instances based on the number of replicas in [guestbook-deployment.yaml](https://github.com/odrodrig/guestbook/blob/openshift-service-mesh/v1/guestbook-deployment.yaml) and v2 [guestbook-deployment.yaml](https://github.com/odrodrig/guestbook/blob/openshift-service-mesh/v2/guestbook-deployment.yaml). By default, prior to creating any rules, Istio will route requests equally across version 1 and version 2 of the guestbook service and their respective instances in a round robin manner. However, new versions of a service can easily introduce bugs to the service mesh, so following A/B Testing and Canary Deployments is good practice.

### A/B testing with Istio

A/B testing is a method of performing identical tests against two separate service versions in order to determine which performs better. To prevent Istio from performing the default routing behavior between the original and modernized guestbook service, define the following rules (found in [istio101/docs/plans](https://github.com/IBM/odrodrig/tree/openshift-service-mesh/docs/plans)):

```shell
oc create -f $WORK_DIR/istio101/docs/plans/guestbook-destination.yaml
```

Let's examine the rule:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: destination-rule-guestbook
spec:
  host: guestbook
  subsets:
    - name: v1
      labels:
        version: '1.0'
    - name: v2
      labels:
        version: '2.0'
```

Next, apply the VirtualService

```shell
oc replace -f $WORK_DIR/istio101/docs/plans/virtualservice-all-v1.yaml
```

Let's examine the rule:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: virtual-service-guestbook
spec:
  hosts:
    - '*'
  gateways:
    - guestbook-gateway
  http:
    - route:
        - destination:
            host: guestbook
            subset: v1
```

The `VirtualService` defines a rule that captures all HTTP traffic coming in through the Istio ingress gateway, `guestbook-gateway`, and routes 100% of the traffic to pods of the guestbook service with label "version: v1". A subset or version of a route destination is identified with a reference to a named service subset which must be declared in a corresponding `DestinationRule`. Since there are three instances matching the criteria of hostname `guestbook` and subset `version: v1`, by default Envoy will send traffic to all three instances in a round robin manner.

View the guestbook application using the `$GUESTBOOK_ROUTE` specified in [Exercise 5](../exercise-5/README.md) and enter it as a URL in Firefox or Chrome web browsers. You can use the echo command to get this value, if you don't remember it.

```shell
echo $GUESTBOOK_ROUTE
```

To enable the Istio service mesh for A/B testing against the new service version, modify the original `VirtualService` rule:

```shell
oc replace -f $WORK_DIR/istio101/docs/plans/virtualservice-test.yaml
```

Let's examine the rule:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: virtual-service-guestbook
spec:
  hosts:
    - '*'
  gateways:
    - guestbook-gateway
  http:
    - match:
        - headers:
            user-agent:
              regex: '.*Firefox.*'
      route:
        - destination:
            host: guestbook
            subset: v2
    - route:
        - destination:
            host: guestbook
            subset: v1
```

![guestbook app in chrome and firefox](../README_images/firefoxchrome.png)

In Istio `VirtualService` rules, there can be only one rule for each service and therefore when defining multiple [HTTPRoute](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRoute) blocks, the order in which they are defined in the yaml matters. Hence, the original `VirtualService` rule is modified rather than creating a new rule. With the modified rule, incoming requests originating from `Firefox` browsers will go to the newer version of guestbook. All other requests fall-through to the next block, which routes all traffic to the original version of guestbook.

Try opening up your route in a Firefox browser and then try it again with a different browser such as Chrome.

### Canary deployment

In `Canary Deployments`, newer versions of services are incrementally rolled out to users to minimize the risk and impact of any bugs introduced by the newer version. To begin incrementally routing traffic to the newer version of the guestbook service, modify the original `VirtualService` rule:

```shell
oc replace -f virtualservice-80-20.yaml
```

Let's examine the rule:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: virtual-service-guestbook
spec:
  hosts:
    - '*'
  gateways:
    - guestbook-gateway
  http:
    - route:
        - destination:
            host: guestbook
            subset: v1
          weight: 80
        - destination:
            host: guestbook
            subset: v2
          weight: 20
```

In the modified rule, the routed traffic is split between two different subsets of the guestbook service. In this manner, traffic to the modernized version 2 of guestbook is controlled on a percentage basis to limit the impact of any unforeseen bugs. This rule can be modified over time until eventually all traffic is directed to the newer version of the service.

View the guestbook application using the `$GUESTBOOK_ROUTE` specified in [Exercise 5](../exercise-5/README.md) and enter it as a URL in Firefox or Chrome web browsers. **Ensure that you are using a hard refresh (command + Shift + R on Mac or Ctrl + F5 on windows) to remove any browser caching.** You should notice that the guestbook should swap between V1 or V2 at about the weight you specified.

### Route all traffic to v2

For the following exercises, we'll be working with Guestbook v2. Route all traffic to guestbook v2 with a new VirtualService rule:

  ```bash
  oc apply -f $WORK_DIR/istio101/docs/plans/virtualservice-all-v2.yaml
  ```

  This is what that rule looks like:

  ```yaml
  apiVersion: networking.istio.io/v1beta1
  kind: VirtualService
  metadata:
    name: virtual-service-guestbook
  spec:
    hosts:
      - '*'
    gateways:
      - guestbook-gateway
    http:
      - route:
          - destination:
              host: guestbook
              subset: v2
  ```

### Implementing circuit breakers with destination rules

Istio `DestinationRules` allow users to configure Envoy's implementation of [circuit breakers](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking.html?highlight=circuit). Circuit breakers are critical for defining the behavior for service-to-service communication in the service mesh. In the event of a failure for a particular service, circuit breakers allow users to set global defaults for failure recovery on a per service and/or per service version basis. Users can apply a [traffic policy](https://istio.io/latest/docs/reference/config/networking/destination-rule/#TrafficPolicy) at the top level of the `DestinationRule` to create circuit breaker settings for an entire service, or it can be defined at the subset level to create settings for a particular version of a service.

Depending on whether a service handles [HTTP](https://istio.io/latest/docs/reference/config/networking/destination-rule/#ConnectionPoolSettings-HTTPSettings) requests or [TCP](https://istio.io/latest/docs/reference/config/networking/destination-rule/#ConnectionPoolSettings-TCPSettings) connections, `DestinationRules` expose a number of ways for Envoy to limit traffic to a particular service as well as define failure recovery behavior for services initiating the connection to an unhealthy service.

## Further reading

* [Istio Concept](https://istio.io/latest/docs/concepts/traffic-management/)
* [Istio Rules API](https://istio.io/latest/docs/reference/config/)
* [Istio Proxy Debug Tool](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-proxy-config)
* [Traffic Management](https://blog.openshift.com/istio-traffic-management-diving-deeper/)
* [Circuit Breaking](https://blog.openshift.com/microservices-patterns-envoy-part-i/)
* [Timeouts and Retries](https://blog.openshift.com/microservices-patterns-envoy-proxy-part-ii-timeouts-retries/)

## Questions

1. Where are routing rules defined?  Options: (VirtualService, DestinationRule, ServiceEntry)  Answer: VirtualService
1. Where are service versions (subsets) defined?  Options: (VirtualService, DestinationRule, ServiceEntry)  Answer: DestinationRule
1. Which Istio component is responsible for sending traffic management configurations to Istio sidecars?  Options: (Mixer, Citadel, Pilot, Kubernetes)  Answer: Pilot
1. What is the name of the default proxy that runs in Istio sidecars and routes requests within the service mesh?  Options: (NGINX, Envoy, HAProxy)  Answer: Envoy

## [Continue to Exercise 7 - Security](../exercise-7/README.md)
