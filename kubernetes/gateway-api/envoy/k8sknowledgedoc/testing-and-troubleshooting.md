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

For a more stable browser-testing path on macOS, prefer the `kind` `extraPortMappings` plus fixed NodePort approach described in [`./stable-browser-testing-on-macos-kind.md`](./stable-browser-testing-on-macos-kind.md).

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

## Common failure: signed versus unsigned numeric schema mismatch

This came up directly in this session while applying the Envoy `BackendTrafficPolicy` local rate-limit example.

The symptom looked like this:

```text
The BackendTrafficPolicy "ratelimit-go-httproute" is invalid:
<nil>: Invalid value: "": Maximum boundary value must be of type integer
with format int32 in spec.rateLimit.local.rules[0].limit.requests
```

At first glance, that error looks like the manifest value is wrong.

But in this case, the real problem was the installed CRD schema, not the YAML manifest.

### What was wrong

The installed `BackendTrafficPolicy` CRD declared the field as:

- `type: integer`
- `format: int32`

but also set:

- `maximum: 4294967295`

That maximum is not valid for a signed `int32` field.

Valid signed `int32` range:

```text
-2147483648 to 2147483647
```

But `4294967295` is the maximum value for an unsigned 32-bit integer.

So the schema was internally inconsistent.

### Why this rejects every value

The key point is that Kubernetes validates not only your object, but also the schema constraints used to validate that object.

In this case:

- your manifest used `requests: 3`
- `3` is a valid `int32`
- but the CRD's `maximum` boundary itself was invalid for `int32`

Because the constraint was broken, the API server could not apply the rule correctly, so it rejected the object before it even got to the question of whether `3` was acceptable.

So the failure was effectively:

- not “your value is too large”
- but “the schema's maximum boundary is invalid for the declared type”

### Why this kind of bug is common

This is a very common modeling mistake across:

- Kubernetes CRDs
- OpenAPI or JSON Schema
- application config schemas
- API contracts
- database column definitions

The root pattern is always similar:

- the declared numeric type is signed
- but the max value or examples assume an unsigned range

Examples of the same class of mistake:

- OpenAPI says `int32` but examples use values above `2147483647`
- a database column is signed `INT` but application code assumes unsigned values
- a config field is documented as non-negative `uint32`, but validation is generated as signed `int32`

### How to recognize it quickly

When you see an error complaining about:

- `Maximum boundary value`
- `format int32`
- or a type boundary mismatch

check the schema itself, not just the object you are applying.

Useful commands:

```bash
kubectl explain backendtrafficpolicy.spec.rateLimit.local.rules.limit.requests \
  --api-version=gateway.envoyproxy.io/v1alpha1

kubectl get crd backendtrafficpolicies.gateway.envoyproxy.io -o yaml
```

If the schema says `format: int32`, then any declared `maximum` above `2147483647` is suspicious.

### How this was fixed in the session

The live cluster CRD was patched to replace the invalid unsigned-style maximum with the real signed `int32` maximum for the affected rate-limit fields.

After that, the exact same manifest validated and applied successfully.

Later in the session, after upgrading Envoy Gateway to `v1.9.0`, the installed CRD changed to use an `int64` format with the same maximum, so the manual patch was no longer required.

### Runtime lesson

When debugging CRD validation problems, do not assume the manifest is wrong just because the error points at a field inside your object.

Sometimes the real bug is:

- in the CRD schema
- in generated OpenAPI constraints
- or in a mismatch between signed and unsigned numeric assumptions

## Common failure: policy exists but is not translated into Envoy xDS

This also came up directly in this session while testing `BackendTrafficPolicy` local rate limiting.

The pattern looked like this:

1. the `BackendTrafficPolicy` object existed in Kubernetes
2. the target `HTTPRoute` existed and was accepted
3. requests still returned `200`, including the 4th request that should have been `429`

In other words, the object was accepted by the Kubernetes API, but the policy was not actually being enforced by Envoy.

### Why `kubectl get` is not enough

These commands only prove that the CR exists:

```bash
kubectl get backendtrafficpolicy ratelimit-go-httproute -n default -o yaml
kubectl get httproute go-route -n default -o yaml
```

They do **not** prove that Envoy Gateway translated the policy into active xDS configuration.

### The runtime diagnosis method used in this session

The reliable check was:

1. inspect Envoy admin `config_dump`
2. look for concrete local rate-limit route config
3. send 4 requests and confirm the 4th becomes `429`

For the admin config:

```bash
envoy=$(kubectl -n envoy-gateway-system get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | grep '^envoy-default-gateway-api-' | head -n1)

kubectl -n envoy-gateway-system port-forward pod/$envoy 19000:19000
```

Then in another terminal:

```bash
curl -s http://127.0.0.1:19000/config_dump | grep -n 'token_bucket\|typed_per_filter_config\|local_ratelimit\|go-route'
```

What you want to see is not just the existence of the `envoy.filters.http.local_ratelimit` filter type.

You want concrete per-route configuration such as:

- `typed_per_filter_config`
- `envoy.filters.http.local_ratelimit`
- `token_bucket`

If those do **not** appear for the route, then the policy is not active in the data plane even if the CR exists in Kubernetes.

### What happened in this repo

On Envoy Gateway `v1.8.0`:

- the local rate-limit policy object existed
- the route existed
- but the route had no effective local-rate-limit xDS config
- the 4th request still returned `200`

After upgrading to Envoy Gateway `v1.9.0`:

- the Envoy `config_dump` showed route-level `typed_per_filter_config`
- the `token_bucket` appeared in the xDS config
- the 4th request returned `429`

### Practical lesson

When testing Envoy Gateway extensions, distinguish these layers:

1. Kubernetes accepted the CR
2. Envoy Gateway translated it into xDS
3. Envoy enforced it at runtime

Do not stop at step 1.

### Minimal repro for local rate limiting

This repo now includes a minimal reproduction manifest:

```text
kubernetes/gateway-api/envoy/10.1-backendpolicy-ratelimit-minimal.yaml
```

Use it when you want to isolate local rate-limit behavior from:

- split HTTP and HTTPS routes
- hostname-specific TLS listeners
- more advanced policy interactions

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
