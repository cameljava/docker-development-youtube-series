# Traffic Exposure and Hostname Mapping

This note explains a subtle but important concept from this Envoy Gateway sample:

- how traffic physically reaches the gateway
- how hostnames are resolved and matched

These are related, but they are not the same thing.

That distinction is one of the most common sources of confusion when testing Gateway API locally.

## The short version

`EnvoyProxy` is not the same thing as an external load balancer.

`EnvoyProxy` configures how Envoy Gateway should create and manage the Envoy data plane. It does not, by itself, give your laptop or the outside world a working path into the cluster.

You still need a transport path for traffic to reach the generated Kubernetes Service.

Separately, you may also need hostname mapping such as `/etc/hosts` or explicit `Host` headers so your request matches the configured `HTTPRoute` rules.

## Three different layers you should separate mentally

When testing traffic, think in three layers.

### 1. Gateway intent layer

This is where you define resources such as:

- `GatewayClass`
- `Gateway`
- `HTTPRoute`

These describe:

- who owns the gateway implementation
- what listeners should exist
- how incoming requests should be matched and routed

### 2. Infrastructure realization layer

This is where Envoy Gateway creates the actual Envoy proxy Deployment and Service.

Examples from this sample:

- generated Envoy Deployment
- generated Envoy Service such as `envoy-default-gateway-api-<hash>` or `envoy-gateway-default`

This is where `EnvoyProxy` matters.

It can tune:

- the generated Service name
- replica count
- telemetry settings
- implementation-level details

### 3. Traffic reachability layer

This is how packets actually get from your browser or `curl` command to the Envoy Service.

Examples:

- real cloud load balancer
- MetalLB
- cloud-provider-kind
- NodePort
- `kubectl port-forward`

Without this layer, a valid `Gateway` and a healthy Envoy proxy may still be unreachable from your local machine.

## What `LoadBalancer` means in a local kind cluster

In this sample, the generated Envoy Service often looks like this:

```text
TYPE           EXTERNAL-IP
LoadBalancer   <pending>
```

This is normal on a plain `kind` cluster.

Why?

Because `kind` does not include a real cloud load balancer implementation.

So Kubernetes accepts the Service type, but there is nothing in the environment that can assign a real external IP.

That means:

- the Service exists
- the Envoy proxy exists
- the gateway may be programmed successfully
- but there is still no direct external IP address to hit from your browser

## What `EnvoyProxy` changes and what it does not

`EnvoyProxy` in this sample can change things like:

- the generated Service name
- replica count
- access logging

It does **not** automatically solve local traffic exposure.

For example, in this repo, [`../02.1-gateway-config.yaml`](../02.1-gateway-config.yaml) changes the generated service name to `envoy-gateway-default`.

That helps with predictability, but it does not create a real external IP for a plain `kind` cluster.

So this statement is accurate:

- `EnvoyProxy` configures the gateway data plane
- it is not itself the network path from your laptop into the cluster

## A useful Apache and NGINX virtual-host analogy

If you come from Apache or NGINX, a very useful mental model is:

- `Gateway` is somewhat like a virtual host definition
- `HTTPRoute` is somewhat like `location` or request-routing rules
- the external `LoadBalancer` or exposure mechanism owns the reachable IP address

This analogy is not perfect, but it is very useful for learning.

### Why the analogy works

In Apache or NGINX, a virtual host typically defines things like:

- which hostname the server responds to
- which port it listens on
- which certificate it should use
- which request paths should go where

That is very close to the role of a Kubernetes `Gateway` plus attached `HTTPRoute` resources.

In this sample:

- the `Gateway` declares listeners such as HTTP on `80` and HTTPS on `443`
- the `Gateway` can reference TLS certificates for HTTPS termination
- the `HTTPRoute` resources decide how requests are matched and forwarded

### Where the analogy breaks

An Apache `VirtualHost` or NGINX `server` block often feels like a complete, self-contained config unit.

Gateway API is more decomposed.

Instead of one file holding everything, the model is split across multiple resources:

- `GatewayClass` says which controller owns the gateway implementation
- `Gateway` defines listeners and entry-point intent
- `HTTPRoute` defines route matching and backend forwarding
- policies add security, traffic, and implementation-specific behavior

So the better analogy is not "Gateway equals one vhost file" but rather:

- `Gateway` plus `HTTPRoute` plus attached policies together behave like the higher-level idea of a modern virtual host configuration

### Mapping Apache/NGINX ideas to Gateway API

Here is a practical mapping:

- public IP on a server or external load balancer -> external IP on a `LoadBalancer` Service or another exposure path
- `VirtualHost` or `server_name` -> `Gateway` listener and hostname intent
- certificate config -> `Gateway` TLS `certificateRefs`
- `location` rules -> `HTTPRoute` matches and filters
- proxy or upstream targets -> `backendRefs`

### The IP address is still a separate concern

This was the core idea from our discussion.

In traditional Apache or NGINX setups, you often mentally combine these ideas:

- the server has an IP address
- the vhost owns a hostname
- the server block owns TLS config

In Kubernetes, those concerns are more clearly separated.

The `Gateway` owns the **behavioral intent**:

- hostname
- listener ports
- TLS termination intent
- route attachment

But the actual reachable IP address is usually owned by the **exposure mechanism**, such as:

- a `LoadBalancer` Service
- a NodePort
- a port-forward
- a local load balancer solution such as MetalLB

That is why this statement is correct:

- Gateway API can describe hostnames and certificates
- but that does not mean the `Gateway` object itself is the external load balancer implementation

### Why hostnames belong in Gateway API anyway

Hostnames and TLS belong in Gateway API because they are part of gateway behavior, not just network exposure.

The gateway is the place where you usually decide:

- which hosts are served
- whether TLS is terminated
- which certificates are used
- which routes are allowed to attach

Those are logical Layer 7 concerns.

The external IP is a Layer 3 and Layer 4 reachability concern.

Gateway API focuses on the Layer 7 gateway intent, while the environment or controller-specific infrastructure provides the actual network exposure.

### A cloud example

In a managed cloud cluster, these concerns often appear unified because the controller may do both jobs:

- provision the real external load balancer
- program hostname, TLS, and route behavior behind it

That can make it feel like the `Gateway` directly owns the IP address.

But conceptually it is still more accurate to say:

- the `Gateway` describes desired entry-point behavior
- the implementation provisions and wires the exposure mechanism

### A kind example

In a plain `kind` cluster, the separation becomes easier to see:

- the `Gateway` can be valid
- the `HTTPRoute` can be valid
- the TLS configuration can be valid
- the generated Service can exist
- but the external IP can still remain `<pending>`

That is because no real external load balancer exists in the environment.

This makes `kind` a very useful learning environment, because it forces you to separate:

- gateway intent
- generated infrastructure
- actual exposure path

### Best mental model to keep

Use this shorthand:

- `Gateway` is like the virtual host and listener definition
- `HTTPRoute` is like the path and request matching rules
- the external IP comes from the exposure mechanism, not from the `Gateway` object itself

That model is simple enough to reason with and accurate enough for real troubleshooting.

## The difference between transport path and hostname mapping

## When to put `hostname` on the `Gateway` listener

This came up directly in this sample once we moved from a single HTTPS hostname to
multiple HTTPS hostnames.

The short version is:

- `HTTPRoute.hostnames` is where request host matching normally lives
- `Gateway.listeners[].hostname` is optional, but useful when a listener itself is intended for a specific host

### What happens if the listener has no hostname

If a `Gateway` listener omits `hostname`, that listener is not restricted to one
host at the listener layer.

In that case:

- the `HTTPRoute.hostnames` field does the hostname matching
- one HTTPS listener can serve multiple hosts if the certificate arrangement supports it

This is often enough when:

- one certificate covers all the needed names
- or the implementation supports the certificate setup you want on a single listener

### What happens if the listener has a hostname

If a listener sets `hostname`, then the effective host matching is the intersection of:

- the listener hostname
- the route hostname list

That means a route only attaches meaningfully when the hostnames line up.

This is especially useful when:

- two HTTPS listeners share the same port
- each listener is meant for a different hostname
- each hostname has its own certificate
- you want the listener boundary to express that intent clearly

## Why the current Envoy sample uses listener hostnames

In the current sample, the `Gateway` has:

- one HTTPS listener for `example-app.com`
- one HTTPS listener for `goexample-app.com`
- both listeners use port `443`
- each listener has its own TLS secret

That design is intentional.

It allows the gateway to use standard HTTPS on the same port while separating
hostname-specific listener intent.

This is the relevant shape from [`../02-gateway.yaml`](../02-gateway.yaml):

```yaml
listeners:
  - name: https
    hostname: example-app.com
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
        - name: secret-tls

  - name: https-goexample
    hostname: goexample-app.com
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
        - name: secret-goexample-tls
```

This does **not** require another external port.

The separation works because:

- HTTPS host selection can use SNI
- each listener is constrained by hostname
- each listener can reference a different certificate

## Why a second listener was the better choice here

In theory, a single HTTPS listener can often serve multiple hosts.

But in this specific case, the sample moved to a second listener because the Go
application hostname needed its own certificate and that certificate was not the
same one used for `example-app.com`.

So the practical options were:

1. use one certificate that covers both hostnames
2. or use two HTTPS listeners on the same port with different listener hostnames and different certificate secrets

The sample uses option 2.

## Best-practice rule of thumb

Use this shorthand when designing Gateway API host handling:

1. Put hostname rules on `HTTPRoute` whenever the route itself is host-specific.
2. Add `listener.hostname` when the listener is also host-specific.
3. Strongly prefer `listener.hostname` when multiple HTTPS listeners share port `443` and each uses a different certificate.
4. Do not create a new port just because you add another hostname. A new port is usually unnecessary.

## How the current Go routes fit this design

The current sample uses **two** routes for the Go backend:

- `go-route-http` for HTTP on `example-app.com`
- `go-route` for HTTPS on `goexample-app.com`

See [`../../07-httproute-tls.yaml`](../../07-httproute-tls.yaml):

```yaml
kind: HTTPRoute
metadata:
  name: go-route-http
spec:
  parentRefs:
    - name: gateway-api
      sectionName: http
  hostnames:
    - example-app.com
---
kind: HTTPRoute
metadata:
  name: go-route
spec:
  parentRefs:
    - name: gateway-api
      sectionName: https-goexample
  hostnames:
    - goexample-app.com
```

This split matters because `hostnames` is defined at the `HTTPRoute` level, not per `parentRef`.

So this is **not** possible in a single route:

- attach one parent to `http`
- attach another parent to `https-goexample`
- use `example-app.com` only for the HTTP parent
- use `goexample-app.com` only for the HTTPS parent

If the hostnames need to differ by entry point, separate `HTTPRoute` resources are the correct design.

That is the key difference from the earlier sample shape where a single route could attach to a more general shared listener.

This is the single most important distinction from the conversation.

### Transport path

Transport path answers:

- how does the request physically reach the Envoy Service?

Examples:

- `kubectl port-forward`
- NodePort
- real external load balancer
- local load balancer controller

### Hostname mapping

Hostname mapping answers:

- what IP address does a hostname such as `example-app.com` resolve to on your machine?

Examples:

- `/etc/hosts`
- DNS
- explicit `curl -H "Host: ..."`

### Why they are different

You can have a valid transport path but still fail routing because the Host header does not match any route.

You can also have a valid hostname mapping but still fail because no traffic path reaches the Service.

## Example 1: Port-forward without matching Host header

Suppose you run:

```bash
kubectl -n envoy-gateway-system port-forward svc/envoy-gateway-default 8080:80
```

Now your local machine can reach Envoy at `127.0.0.1:8080`.

That solves the **transport path** problem.

But if you browse to:

```text
http://localhost:8080/
```

the request host is usually `localhost`.

If your `HTTPRoute` expects:

- `example-app.com`
- or `example-app-go.com`
- or `example-app-python.com`

then Envoy will return `404` because the route did not match.

This is why a `404` after port-forwarding can still mean the network path is working correctly.

## Example 2: `/etc/hosts` without a transport path

Suppose you add this to `/etc/hosts`:

```text
127.0.0.1 example-app.com
```

Now your machine resolves `example-app.com` to `127.0.0.1`.

That solves the **hostname mapping** problem.

But if you do not also provide a transport path such as:

- port-forward
- NodePort
- real load balancer

there is still nothing listening locally for that traffic to reach the cluster.

So `/etc/hosts` alone is not enough.

## `curl -H "Host: ..."` versus `--resolve`

This is another subtle point that matters a lot when testing Gateway API locally.

### What `curl` does by default

If you run:

```bash
curl http://127.0.0.1:8080/api/go/status
```

then curl does two things based on that URL:

1. it connects to `127.0.0.1:8080`
2. it automatically sends an HTTP host corresponding to that URL host

So by default, the URL controls both:

- the network destination
- the default HTTP Host header

### What `-H "Host: ..."` changes

If you run:

```bash
curl -H "Host: example-app.com" http://127.0.0.1:8080/api/go/status
```

then curl still connects to:

- `127.0.0.1:8080`

but the HTTP `Host` header sent in the request becomes:

- `example-app.com`

This is extremely useful for local HTTP testing because it lets you:

- reach a local port-forward or NodePort
- while pretending the request was sent to the route hostname expected by the gateway

### What `-H "Host: ..."` does **not** change

It does not change the actual network destination.

It also does not fully replace the role of the URL hostname in HTTPS scenarios.

For HTTPS, the URL hostname may still be used by curl for things such as:

- DNS resolution
- TLS SNI (Server Name Indication)
- certificate validation

So `-H "Host: ..."` is very useful for HTTP testing, but it is not always enough for HTTPS virtual-host testing.

## Why HTTPS is different

With HTTPS, there are effectively two hostname-sensitive layers:

1. the HTTP host and routing layer
2. the TLS handshake layer

The gateway may route based on hostnames at the HTTP layer, but TLS may also use the hostname before the HTTP request is even processed.

That is why a plain host-header override can be insufficient for HTTPS.

## What `--resolve` does

`curl --resolve` is a very useful testing tool because it lets you keep the desired hostname in the URL while overriding where curl actually connects.

General form:

```bash
curl --resolve <host>:<port>:<ip> <url>
```

### Syntax breakdown

In:

```bash
--resolve example-app.com:8443:127.0.0.1
```

the fields mean:

- `example-app.com` = the hostname this override applies to
- `8443` = the port this override applies to
- `127.0.0.1` = the concrete address curl should connect to instead of doing normal DNS lookup

There is no second target port after the IP address because curl uses the same port already specified in the middle field.

So this:

```bash
curl --resolve example-app.com:8443:127.0.0.1 \
  https://example-app.com:8443/api/go/status
```

effectively means:

- use `example-app.com` as the URL host
- connect to `127.0.0.1:8443`

### Does the target have to be an IP address?

In practice, yes, you should treat the third field as a concrete address, usually:

- an IPv4 address
- or an IPv6 address

Do not think of it as another hostname to be resolved again. The point of `--resolve` is to bypass normal DNS and provide the final address directly.

### Does the port have to match the URL?

Yes.

The override is per host and port pair.

That means these are different mappings:

- `example-app.com:443`
- `example-app.com:8443`

So if your URL is:

```bash
https://example-app.com:8443/...
```

then the matching override is:

```bash
--resolve example-app.com:8443:127.0.0.1
```

not:

```bash
--resolve example-app.com:443:127.0.0.1
```

Example:

```bash
curl --resolve example-app.com:8443:127.0.0.1 \
	https://example-app.com:8443/api/go/status
```

This tells curl:

- treat the URL host as `example-app.com`
- but connect to `127.0.0.1` for port `8443`

That means `--resolve` helps align multiple layers at once:

- the URL host is the real desired hostname
- DNS lookup is bypassed and replaced with your chosen IP
- TLS SNI uses the desired hostname from the URL
- certificate validation can also use that hostname
- the HTTP host logic is consistent with the URL host

This makes `--resolve` the best local HTTPS testing pattern when you need:

- local connectivity to `127.0.0.1`
- route matching for `example-app.com`
- TLS behavior that also expects `example-app.com`

### A practical comparison

#### HTTP case with `-H`

```bash
curl -H "Host: example-app.com" http://127.0.0.1:8080/api/go/status
```

Good for:

- local HTTP tests
- testing route host matching without changing DNS

#### HTTPS case with `-H`

```bash
curl -H "Host: example-app.com" https://127.0.0.1:8443/api/go/status
```

This may still fail or behave unexpectedly because the TLS layer still sees the URL host as `127.0.0.1` unless additional options are used.

#### HTTPS case with `--resolve`

```bash
curl --resolve example-app.com:8443:127.0.0.1 \
	https://example-app.com:8443/api/go/status
```

This is usually the better and cleaner HTTPS test.

## When to use which

### Use `-H "Host: ..."` when:

- you are testing HTTP
- you need a quick route match override
- you are connecting to a local port-forward or NodePort

### Use `--resolve` when:

- you are testing HTTPS
- the gateway expects a specific hostname
- you want TLS SNI and certificate checks to match that hostname
- you want to bypass real DNS while still using the correct URL host

## Examples for this sample

### HTTP with port-forward and Host override

```bash
kubectl -n envoy-gateway-system port-forward svc/<service-name> 8080:80
curl -H "Host: example-app.com" http://127.0.0.1:8080/api/go/status
```

### HTTPS with port-forward and `--resolve`

```bash
kubectl -n envoy-gateway-system port-forward svc/<service-name> 8443:443

curl --resolve example-app.com:8443:127.0.0.1 \
  --cacert kubernetes/gateway-api/tls/rootCA.pem \
  https://example-app.com:8443/api/go/status
```

### HTTPS with client certificate and `--resolve`

```bash
curl --resolve example-app.com:8443:127.0.0.1 \
  --cert kubernetes/gateway-api/tls/client-user.com-client.pem \
  --key kubernetes/gateway-api/tls/client-user.com-client-key.pem \
  --cacert kubernetes/gateway-api/tls/rootCA.pem \
  https://example-app.com:8443/api/go/status
```

This pattern is especially useful once you start testing mTLS and API-key based access in this sample.

## Local testing options for this sample

### Option 1: `kubectl port-forward`

Best for quick local testing.

Examples:

```bash
kubectl -n envoy-gateway-system port-forward svc/<service-name> 8080:80
kubectl -n envoy-gateway-system port-forward svc/<service-name> 8443:443
```

This is the simplest and most reliable option in a plain `kind` setup.

### Option 2: NodePort

If the Service exposes a node port and your environment allows access to it, you may be able to connect using the assigned node port.

Example service output:

```text
80:32755/TCP
```

Then you may test with:

```bash
curl -H "Host: example-app.com" http://127.0.0.1:32755/api/go/status
```

Whether this works depends on your local Docker and `kind` networking setup.

### Option 3: Local load balancer support

If you want a more cloud-like local experience, you can add a local load balancer solution such as:

- MetalLB
- cloud-provider-kind

This can provide a real local address for `LoadBalancer` Services.

### Option 4: Real cloud load balancer

In a managed cloud cluster, `LoadBalancer` Services usually receive a real external IP or DNS name.

In that case the traffic path is handled by the cloud provider.

## Why port 80 and 443 failed locally on macOS

Another lesson from this conversation is that local port selection matters.

On macOS, binding local ports `80` and `443` often requires elevated privileges.

That is why the README examples were updated to use:

- `8080:80`
- `8443:443`

These are non-privileged local ports and are easier to use for development.

## Recommended test patterns

### HTTP with explicit Host header

```bash
kubectl -n envoy-gateway-system port-forward svc/<service-name> 8080:80
curl -H "Host: example-app.com" http://127.0.0.1:8080/api/go/status
```

### HTTPS with explicit Host header

```bash
kubectl -n envoy-gateway-system port-forward svc/<service-name> 8443:443
curl -k -H "Host: example-app.com" https://127.0.0.1:8443/api/go/status
```

### Browser testing with `/etc/hosts`

If you want browser-friendly testing, add a hosts entry such as:

```text
127.0.0.1 example-app.com
```

Then, with a port-forward active, browse to:

```text
http://example-app.com:8080/api/go/status
```

This combines:

- transport path via port-forward
- hostname mapping via `/etc/hosts`

## How this applies to the Envoy sample specifically

In this sample, if you see:

- `LoadBalancer` with `EXTERNAL-IP <pending>`
- working Envoy Service in `envoy-gateway-system`
- working `Gateway`
- but no browser access on plain `localhost`

that usually means one of these is still missing:

1. a transport path from your machine to the Service
2. a matching Host header or hostname mapping
3. a matching `HTTPRoute`

So the right debugging order is:

1. confirm the Service exists
2. confirm the Envoy proxy Pods exist
3. confirm a transport path exists, such as port-forward
4. confirm `HTTPRoute` resources exist
5. confirm the request host/path matches the route rules

## Mental model

Use this short mental model while testing:

- `EnvoyProxy` shapes the data plane
- the generated Service is the in-cluster entry point
- a load balancer or port-forward provides the traffic path
- `/etc/hosts` or Host headers provide hostname matching
- `HTTPRoute` decides whether the request actually matches and forwards

If you separate those concerns clearly, most local Gateway API debugging becomes much easier.