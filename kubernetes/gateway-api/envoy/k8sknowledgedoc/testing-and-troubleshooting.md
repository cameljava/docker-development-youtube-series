# Testing and Troubleshooting

This note captures the practical testing and troubleshooting lessons from this conversation.

## Start with the current context

Always confirm which cluster you are talking to:

```bash
kubectl config current-context
```

If you install CRDs into one cluster and apply manifests to another, the results will look inconsistent.

## Verify CRDs separately

Do not assume one CRD family implies another.

### Gateway API CRDs

```bash
kubectl get crd gatewayclasses.gateway.networking.k8s.io
```

### Envoy Gateway CRDs

```bash
kubectl get crd envoyproxies.gateway.envoyproxy.io
kubectl api-resources --api-group=gateway.envoyproxy.io
```

This matters because `GatewayClass` working does not mean `EnvoyProxy` exists.

## Common failure: OCI metadata in rendered YAML

One issue in this conversation was that `helm template` output for the CRD chart included lines such as:

- `Pulled: ...`
- `Digest: ...`

Those are not Kubernetes YAML and can break `kubectl apply`.

That is why the README now strips everything before the first `---` when generating the CRD file.

## Common failure: wrong values file path

If you are already in `kubernetes/gateway-api/envoy`, use:

```bash
--values values.yaml
```

If you are at the repository root, use:

```bash
--values kubernetes/gateway-api/envoy/values.yaml
```

This came up directly in the session when Helm failed to open the values file.

## Common failure: generated service name mismatch

Before the custom `EnvoyProxy` configuration from `02.1-gateway-config.yaml` applies successfully, the Envoy service may still have a generated name such as:

- `envoy-default-gateway-api-<hash>`

Do not assume the service is already named `envoy-gateway-default` until the `EnvoyProxy` change has successfully reconciled.

Always check:

```bash
kubectl -n envoy-gateway-system get svc
```

and use the actual service name that exists.

## Common failure: privileged local ports on macOS

Port-forwarding to local ports `80` and `443` can fail on macOS without elevated privileges.

Use non-privileged local ports instead:

```bash
kubectl -n envoy-gateway-system port-forward svc/<service-name> 8080:80
kubectl -n envoy-gateway-system port-forward svc/<service-name> 8443:443
```

## Common failure: `404` in browser

If you browse to:

```text
http://localhost:8080/
```

and get `404`, that often means Envoy is working but no route matched the request.

In this sample, many routes depend on:

- a specific Host header such as `example-app.com`
- a specific path such as `/api/go`

So `localhost` plus `/` may not match anything.

Example test:

```bash
curl -H "Host: example-app.com" http://127.0.0.1:8080/api/go/status
```

## Common failure: no `HTTPRoute` resources

If you run:

```bash
kubectl get httproute -n default
```

and see no resources, then a `404` from Envoy is expected because no route has been defined.

Install the route manifests from the parent Gateway API examples before testing traffic routing.

## Useful verification sequence

```bash
kubectl get gatewayclass
kubectl get gateway -n default
kubectl get httproute -n default
kubectl get secret -n default
kubectl get envoyproxy -n envoy-gateway-system
kubectl get pods,svc -n envoy-gateway-system
kubectl describe gateway gateway-api -n default
```

## What Kubernetes remembers versus what it does not

Kubernetes stores objects, not the original manifest file path.

So after the fact, the cluster usually cannot tell you:

- which local file path you used
- whether you applied a single file or a whole directory
- the exact shell command you ran

What you can recover instead is:

- the live object
- annotations such as `kubectl.kubernetes.io/last-applied-configuration`
- `managedFields`
- object names you can search for in the repo

This is why searching the repo by object name is often the most practical way to map a live object back to a manifest.