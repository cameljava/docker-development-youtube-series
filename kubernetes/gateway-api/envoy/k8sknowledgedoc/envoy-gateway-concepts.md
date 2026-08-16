# Envoy Gateway Concepts

This note explains the core concepts behind the Envoy Gateway sample in this folder.

## The three layers to keep separate

There are three related but different layers in this sample:

1. Gateway API resources
2. Envoy Gateway controller-specific resources
3. generated Envoy data-plane infrastructure

If you blur these together, troubleshooting becomes confusing.

## Gateway API resources

These are the portable resources defined by the Kubernetes Gateway API project.

Examples in this sample:

- `GatewayClass`
- `Gateway`
- `HTTPRoute`

These resources describe intent:

- who owns the gateway implementation
- what listeners should exist
- how requests should be matched and routed

## Envoy Gateway resources

These are custom resources specific to Envoy Gateway.

Examples in this sample:

- `EnvoyProxy`
- `ClientTrafficPolicy`
- `BackendTrafficPolicy`
- `SecurityPolicy`

These resources describe implementation-specific behavior or extensions.

This means:

- `Gateway` is portable Gateway API intent
- `EnvoyProxy` is Envoy Gateway implementation tuning

If you moved to another Gateway API controller, your `Gateway` and `HTTPRoute` might still make sense, but `EnvoyProxy` would not.

## Generated Envoy infrastructure

This is the actual deployment and service created by the controller to serve traffic.

Examples:

- generated Envoy Deployment
- generated Envoy Service such as `envoy-default-gateway-api-<hash>`

These are not authored directly in your sample YAML files. They are produced by the controller after it reconciles the higher-level resources.

## Controller vs data plane

### Control plane

The Envoy Gateway controller runs in `envoy-gateway-system`.

Its job is to:

- watch Kubernetes resources
- validate and reconcile intent
- create and update the managed Envoy infrastructure

### Data plane

The generated Envoy proxy receives real traffic.

The controller does not directly serve application traffic.

So:

- control plane handles reconciliation
- data plane handles requests and responses

## What each main manifest does

### `01-gatewayclass.yaml`

Defines the `GatewayClass` named `envoy`.

This tells the cluster which controller should own Gateways that reference this class.

### `02-gateway.yaml`

Defines the `Gateway` named `gateway-api` in `default`.

This is the logical gateway instance that asks the controller to create entry points on ports `80` and `443`.

### `02.1-gateway-config.yaml`

Updates the `GatewayClass` with a `parametersRef` and creates an `EnvoyProxy` named `gateway-configuration`.

This is how the sample customizes the managed Envoy infrastructure, including:

- desired service name
- replica count
- telemetry settings

## Service naming behavior

One subtle point from this conversation is service naming.

Before the custom `EnvoyProxy` config is applied successfully, Envoy Gateway may create a generated service name such as:

- `envoy-default-gateway-api-<hash>`

After the `EnvoyProxy` configuration from `02.1-gateway-config.yaml` applies successfully, the service may be renamed or created according to the configured name:

- `envoy-gateway-default`

That is why a hard-coded `kubectl port-forward svc/envoy-gateway-default ...` can fail if `02.1` has not successfully reconciled yet.

## HTTPRoute and Host header behavior

Another important concept from this conversation is that a working port-forward does not guarantee a successful route match.

You can reach Envoy and still get `404` if no `HTTPRoute` matches the request.

Common cause in this sample:

- browser request goes to `http://localhost:8080/`
- actual route expects a Host header like `example-app.com`

So `404` can mean:

- traffic reached Envoy successfully
- but no route matched the host/path combination

## CRD sets you must not confuse

There are two different CRD families involved:

### Gateway API CRDs

Examples:

- `gatewayclasses.gateway.networking.k8s.io`
- `gateways.gateway.networking.k8s.io`
- `httproutes.gateway.networking.k8s.io`

### Envoy Gateway CRDs

Examples:

- `envoyproxies.gateway.envoyproxy.io`
- `clienttrafficpolicies.gateway.envoyproxy.io`
- `backendtrafficpolicies.gateway.envoyproxy.io`
- `securitypolicies.gateway.envoyproxy.io`

Having the first set installed does not mean the second set exists.

That was the direct cause of the `EnvoyProxy` apply failure discussed in this session.

## Learning checklist

To understand what is happening in this sample, always distinguish:

- controller vs data plane
- portable Gateway API intent vs Envoy-specific extension config
- authored manifests vs generated infrastructure
- API existence via CRD vs object existence via `kubectl get`