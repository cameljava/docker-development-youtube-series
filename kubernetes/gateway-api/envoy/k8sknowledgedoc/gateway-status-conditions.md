# Gateway Status Conditions

This note explains the status conditions you see when inspecting a `Gateway`.

## Why status conditions matter

When you run:

```bash
kubectl get gateway -n default
kubectl describe gateway gateway-api -n default
kubectl get gateway gateway-api -n default -o yaml
```

you are trying to answer two separate questions:

1. Does the object exist?
2. Has the controller successfully reconciled it?

Existence alone is not enough. A `Gateway` can exist in Kubernetes and still not be usable.

## `Accepted`

This condition usually answers whether the controller accepts ownership of the `Gateway`.

- `Accepted=True` means the controller recognizes the `GatewayClass` and accepts the object for reconciliation.
- `Accepted=False` usually means the class is invalid, the controller does not own it, or the `Gateway` fails a high-level validation step.

## `ResolvedRefs`

This condition usually answers whether object references were resolved successfully.

Common examples:

- TLS certificate secrets
- parameters references
- policy references or other linked resources

In this sample, an HTTPS listener that references `secret-tls` can lead to `ResolvedRefs=False` if the secret is missing or invalid.

## `Programmed`

This condition answers whether the controller successfully translated the desired `Gateway` into actual infrastructure and data-plane programming.

- `Programmed=True` means the `Gateway` is fully realized.
- `Programmed=False` means the object exists, but the controller has not successfully finished making it real.

In this sample, `Programmed=False` can happen when:

- the Envoy Gateway controller is unhealthy
- the generated Envoy proxy Deployment or Service failed
- the listeners are invalid
- TLS configuration is wrong
- a secret such as `secret-tls` is missing or malformed
- reconciliation failed after the `Gateway` was accepted

## Best commands to inspect conditions

Use both a summary view and a detailed view.

```bash
kubectl get gateway -n default
kubectl describe gateway gateway-api -n default
kubectl get gateway gateway-api -n default -o yaml
```

In `describe` output, pay attention to:

- `Conditions`
- `Listeners`
- any reason/message text

In YAML output, inspect:

- `status.conditions`
- `status.listeners`

The reason and message fields are often the shortest path to the root cause.

## Interpreting a `404` versus `Programmed=False`

These mean different things.

### `Programmed=False`

Usually means Envoy Gateway did not successfully finish creating or programming the data plane.

### `404` from Envoy

Usually means Envoy is running and received the request, but no `HTTPRoute` matched the request host and path.

This distinction came up in this conversation:

- a port-forward could work
- the gateway service could exist
- but requests to `localhost` still returned `404`

That was a route-matching problem, not necessarily a `Programmed=False` problem.

## Debugging sequence

1. Confirm the `Gateway` exists.
2. Check `Accepted`.
3. Check `ResolvedRefs`.
4. Check `Programmed`.
5. Check the generated Envoy proxy Pods and Services.
6. Check whether matching `HTTPRoute` resources exist.
7. Check host/path matching for the request you are sending.

Useful commands:

```bash
kubectl get gateway -n default
kubectl describe gateway gateway-api -n default
kubectl get httproute -n default
kubectl -n envoy-gateway-system get pods
kubectl -n envoy-gateway-system get svc
kubectl -n envoy-gateway-system logs -l app.kubernetes.io/instance=envoy-gateway
```