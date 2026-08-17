# Upgrading CRDs on Live Clusters

This note explains the correct way to upgrade Gateway API and Envoy Gateway CRDs on a live Kubernetes cluster.

The key lesson from this session is simple:

- CRD upgrades are not just file replacements
- controller compatibility matters
- stored versions matter
- field ownership matters
- and controller-specific extension CRDs may need different handling than core Gateway API CRDs

## Two different CRD families

Always separate these two groups mentally.

### 1. Core Gateway API CRDs

These are the upstream Kubernetes Gateway API resources under:

- `gateway.networking.k8s.io`

Examples:

- `GatewayClass`
- `Gateway`
- `HTTPRoute`
- `ReferenceGrant`

These are installed from the Gateway API standard or experimental channel bundle.

### 2. Envoy Gateway extension CRDs

These are the Envoy-specific resources under:

- `gateway.envoyproxy.io`

Examples:

- `EnvoyProxy`
- `BackendTrafficPolicy`
- `ClientTrafficPolicy`
- `SecurityPolicy`

These are installed from the Envoy Gateway CRD chart, not the core Gateway API bundle.

That distinction matters because:

- the core Gateway API release process and compatibility rules are upstream Kubernetes concerns
- the Envoy Gateway extension CRDs follow the Envoy Gateway release lifecycle

## The safe upgrade sequence

For a live cluster, the safest sequence is:

1. inventory current versions
2. read compatibility and upgrade notes
3. back up CRDs and important custom resources
4. upgrade CRDs first
5. upgrade the controller second
6. verify object acceptance and runtime behavior

Do not start by blindly applying a newer CRD YAML.

## Step 1: Inventory the current state

Before touching anything, record what is running.

### Check the Envoy Gateway controller image

```bash
kubectl -n envoy-gateway-system get deployment envoy-gateway \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

### Check served and storage versions for core Gateway API CRDs

```bash
kubectl get crd gatewayclasses.gateway.networking.k8s.io -o jsonpath='{range .spec.versions[*]}{.name}:{.served}:{.storage}{" "}{end}{"\n"}'
kubectl get crd gateways.gateway.networking.k8s.io -o jsonpath='{range .spec.versions[*]}{.name}:{.served}:{.storage}{" "}{end}{"\n"}'
kubectl get crd httproutes.gateway.networking.k8s.io -o jsonpath='{range .spec.versions[*]}{.name}:{.served}:{.storage}{" "}{end}{"\n"}'
```

### Check served and storage versions for Envoy extension CRDs

```bash
for crd in \
  envoyproxies.gateway.envoyproxy.io \
  backendtrafficpolicies.gateway.envoyproxy.io \
  clienttrafficpolicies.gateway.envoyproxy.io \
  securitypolicies.gateway.envoyproxy.io; do
  echo "=== $crd ==="
  kubectl get crd "$crd" -o jsonpath='{range .spec.versions[*]}{.name}:{.served}:{.storage}{" "}{end}{"\n"}'
done
```

### Check stored versions when upgrading core Gateway API CRDs

This is especially important when moving from older Gateway API releases.

```bash
kubectl get crd grpcroutes.gateway.networking.k8s.io -o jsonpath='{.status.storedVersions}{"\n"}'
kubectl get crd referencegrants.gateway.networking.k8s.io -o jsonpath='{.status.storedVersions}{"\n"}'
```

If old storage versions remain, upstream Gateway API upgrade notes may require manual cleanup.

## Step 2: Check compatibility before upgrading

For Envoy Gateway, check the compatibility matrix and release notes first.

Questions to answer:

- which Gateway API version is bundled with the target Envoy Gateway release?
- which Kubernetes versions are supported?
- are there breaking changes in the target release?
- are any fields deprecated or stricter now?

This matters because the controller and the CRDs are version-coupled in practice.

In this session, for example:

- `v1.8.0` bundled Gateway API `v1.5.1`
- `v1.9.0` bundled Gateway API `v1.6.1`

## Step 3: Back up live definitions before changing them

At minimum, back up:

- the CRDs themselves
- the controller Helm values or install inputs
- any Envoy extension resources you rely on

### Back up CRDs

```bash
mkdir -p /tmp/eg-crd-backup

kubectl get crd gatewayclasses.gateway.networking.k8s.io -o yaml > /tmp/eg-crd-backup/gatewayclass-crd.yaml
kubectl get crd gateways.gateway.networking.k8s.io -o yaml > /tmp/eg-crd-backup/gateway-crd.yaml
kubectl get crd httproutes.gateway.networking.k8s.io -o yaml > /tmp/eg-crd-backup/httproute-crd.yaml
kubectl get crd backendtrafficpolicies.gateway.envoyproxy.io -o yaml > /tmp/eg-crd-backup/backendtrafficpolicy-crd.yaml
```

### Back up critical policy objects

```bash
kubectl get backendtrafficpolicy -A -o yaml > /tmp/eg-crd-backup/backendtrafficpolicies.yaml
kubectl get clienttrafficpolicy -A -o yaml > /tmp/eg-crd-backup/clienttrafficpolicies.yaml
kubectl get securitypolicy -A -o yaml > /tmp/eg-crd-backup/securitypolicies.yaml
```

## Step 4: Upgrade CRDs first

This is the correct order for a live cluster.

Why?

- new controllers often expect new schema fields to exist
- old CRDs can cause reconciliation failures or silently skipped resources
- the controller should not be upgraded first if it depends on new CRD definitions

### Upgrading Envoy Gateway CRDs

For the pattern used in this repo:

```bash
CHART_VERSION="v1.9.0"

helm template eg-crds oci://docker.io/envoyproxy/gateway-crds-helm \
  --version "$CHART_VERSION" \
  --set crds.gatewayAPI.enabled=false \
  --set crds.envoyGateway.enabled=true \
  | awk 'BEGIN {print_yaml=0} /^---$/ {print_yaml=1} print_yaml {print}' > /tmp/envoy-gateway-crds.yaml

kubectl apply --server-side -f /tmp/envoy-gateway-crds.yaml
```

### Upgrading core Gateway API CRDs

If you manage Gateway API CRDs separately:

```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.1/standard-install.yaml
```

Or use the experimental channel if you intentionally depend on experimental resources.

## Step 5: Handle server-side apply conflicts correctly

This also came up directly in this session.

We had manually patched a live Envoy Gateway CRD earlier. When the upgraded CRD bundle was applied, Kubernetes reported a server-side apply conflict because a different field manager owned part of `.spec.versions`.

That is expected behavior.

### What the conflict means

It does **not** automatically mean the upgrade is unsafe.

It means:

- some fields were already modified by another manager
- `kubectl apply --server-side` is protecting ownership boundaries

### Correct response

First decide whether the incoming CRD definition is truly the new source of truth.

If yes, then forcing ownership can be correct:

```bash
kubectl apply --server-side --force-conflicts -f /tmp/envoy-gateway-crds.yaml
```

Only do this after confirming:

- the new CRD content is the version you actually want
- any earlier manual patches should be superseded

In this session, that was exactly the right thing to do after moving from the ad hoc CRD patch to the upstream `v1.9.0` CRD definition.

## Step 6: Upgrade the controller after CRDs

Once the CRDs are in place, upgrade the controller.

For the repo pattern here:

```bash
CHART_VERSION="v1.9.0"

helm upgrade envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
  --version "$CHART_VERSION" \
  --values kubernetes/gateway-api/envoy/values.yaml \
  --namespace envoy-gateway-system \
  --skip-crds
```

Then wait for it:

```bash
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
kubectl -n envoy-gateway-system get pods
```

## Step 7: Verify both API-level and runtime-level behavior

Do not stop at “the deployment rolled out”.

You need to verify:

1. the new schema is present
2. the controller is running the expected version
3. resources reconcile successfully
4. the data plane actually enforces the intended behavior

### Check the upgraded CRD schema

Example from this session:

```bash
kubectl get crd backendtrafficpolicies.gateway.envoyproxy.io -o json | jq -r '
  .spec.versions[] | select(.name=="v1alpha1") |
  [
    .schema.openAPIV3Schema.properties.spec.properties.rateLimit.properties.local.properties.rules.items.properties.limit.properties.requests.format,
    .schema.openAPIV3Schema.properties.spec.properties.rateLimit.properties.local.properties.rules.items.properties.limit.properties.requests.maximum
  ] | @tsv'
```

Before upgrade, we had a bad effective combination in the live cluster.
After upgrade to `v1.9.0`, the live schema reported:

- `int64`
- `4294967295`

which made the earlier manual `int32` patch unnecessary.

### Check the controller image

```bash
kubectl -n envoy-gateway-system get deployment envoy-gateway \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

### Check resource acceptance

```bash
kubectl get gateway -A
kubectl get httproute -A
kubectl get backendtrafficpolicy -A
```

### Check data-plane translation when needed

For extension features, Kubernetes acceptance alone is not enough.

In this session, a `BackendTrafficPolicy` local rate-limit object existed on `v1.8.0` but did not reach Envoy xDS.

The useful runtime check was:

```bash
envoy=$(kubectl -n envoy-gateway-system get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | grep '^envoy-default-gateway-api-' | head -n1)

kubectl -n envoy-gateway-system port-forward pod/$envoy 19000:19000
```

Then:

```bash
curl -s http://127.0.0.1:19000/config_dump | grep -n 'token_bucket\|typed_per_filter_config\|local_ratelimit\|go-route'
```

What you want to see:

- route-level `typed_per_filter_config`
- `envoy.filters.http.local_ratelimit`
- `token_bucket`

If those do not appear, the policy is not active in the data plane even if the CR exists.

## Step 8: Re-test the exact behavior that was failing before

A live upgrade is only complete when the previously failing behavior now works.

In this session, that meant rerunning the local rate-limit test after upgrading from `v1.8.0` to `v1.9.0`.

Before upgrade:

- 4 requests returned `200`
- no local rate-limit config was visible in xDS

After upgrade:

- xDS showed `typed_per_filter_config`
- xDS showed `token_bucket`
- request 4 returned `429`

That was the real proof that the upgrade fixed the issue.

## A practical upgrade checklist

Use this every time you upgrade live CRDs:

1. Identify whether you are upgrading core Gateway API CRDs, Envoy Gateway CRDs, or both.
2. Check compatibility matrix and release notes first.
3. Back up the live CRDs and important custom resources.
4. Upgrade CRDs before controllers.
5. Resolve server-side apply conflicts intentionally, not blindly.
6. Verify controller image version after rollout.
7. Verify reconciled object status.
8. Verify actual Envoy xDS when debugging extension features.
9. Re-run the real failing behavior after upgrade.

## The main lesson from this session

The correct live-cluster upgrade process is not:

- upgrade the controller and hope the APIs line up

It is:

- verify versions
- upgrade CRDs deliberately
- handle ownership conflicts carefully
- upgrade the controller
- verify runtime behavior, not just object acceptance
