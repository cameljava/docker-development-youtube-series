# Introduction to Envoy: Gateway API

<a href="https://youtu.be/me_5W_Q4ZWg" title="envoy"><img src="https://i.ytimg.com/vi/me_5W_Q4ZWg/hqdefault.jpg" width="40%" alt="envoy" /></a>

## Prerequisites

To get started, you will need to follow the [Introduction to Gateway API](../README.md) first. </br>
You'll need an understanding of the Gateway API. </br>

<b>In the introduction guide, you will:</b>
* Create a local Kubernetes cluster
* Install the Gateway API CRDs
* Deploy example apps to our cluster
* Have Domains for our traffic
* Have TLS certificates

This will allow us access to the Gateway API so we can go ahead and deploy a Gateway API controller to use. </br

## What is Envoy

It's important to take a step back and fully understand and appreciate what Envoy is. </br>
Envoy is not only a Gateway API, that's just one of its features. </br>
Let's take a look at the [Official Site](https://www.envoyproxy.io/) and jump to the documentation. </br>
Envoy has a separate web page for the Gateway API feature. </br>

For deeper conceptual background, see the knowledge docs in [`./k8sknowledgedoc/index.md`](./k8sknowledgedoc/index.md). </br>

For live-cluster CRD upgrades, see [`./k8sknowledgedoc/upgrading-crds-on-live-clusters.md`](./k8sknowledgedoc/upgrading-crds-on-live-clusters.md). </br>

## Cheat sheet

Run these commands from `kubernetes/gateway-api/envoy` unless noted otherwise.

### Install Gateway API CRDs

```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.1/standard-install.yaml
kubectl get crd gatewayclasses.gateway.networking.k8s.io
```

### Install Envoy Gateway CRDs and controller

```bash
CHART_VERSION="v1.9.0"

helm template eg-crds oci://docker.io/envoyproxy/gateway-crds-helm \
  --version $CHART_VERSION \
  --set crds.gatewayAPI.enabled=false \
  --set crds.envoyGateway.enabled=true \
  | awk 'BEGIN {print_yaml=0} /^---$/ {print_yaml=1} print_yaml {print}' > /tmp/envoy-gateway-crds.yaml

kubectl apply --server-side -f /tmp/envoy-gateway-crds.yaml

# verify Envoy Gateway CRDs, not just Gateway API CRDs
kubectl get crd envoyproxies.gateway.envoyproxy.io
kubectl api-resources --api-group=gateway.envoyproxy.io

helm install envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
  --version $CHART_VERSION \
  --values values.yaml \
  --namespace envoy-gateway-system \
  --create-namespace \
  --skip-crds

kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
kubectl -n envoy-gateway-system get pods
```

### Apply and verify the class and gateway

```bash
kubectl apply -f 01-gatewayclass.yaml
kubectl get gatewayclass
kubectl describe gatewayclass envoy

kubectl apply -f 02-gateway.yaml
kubectl get gateway -n default
kubectl describe gateway gateway-api -n default
kubectl -n envoy-gateway-system get pods
kubectl -n envoy-gateway-system get svc
```

### Inspect cluster-level resources

```bash
kubectl api-resources --namespaced=false
kubectl get gatewayclass
kubectl get crd | grep -E 'gateway|envoy'
kubectl get clusterrole,clusterrolebinding | grep -E 'envoy|gateway'
kubectl get validatingadmissionpolicy,validatingadmissionpolicybinding
```

### Inspect namespaced resources

```bash
kubectl get gateway -n default
kubectl get httproute -n default
kubectl get pods,svc -n envoy-gateway-system
```

## Envoy: Gateway API controller

Envoy has a `helm` chart specifically for the gateway product. </br>
The `helm` chart relies on the Gateway API CRD's we installed in our cluster already as well as Envoy Proxy. </br>
Because the introduction guide already installs Gateway API CRDs, we only need to install the Envoy Gateway CRDs here and disable CRD management in the main chart. </br>

### Installation 

We can install the helm chart with `helm install` or upgrade it with `helm upgrade`:

Run the following commands from the repository root. </br>
If you are already in `kubernetes/gateway-api/envoy`, change `kubernetes/gateway-api/envoy/values.yaml` to `values.yaml`. </br>

```shell

CHART_VERSION="v1.9.0"

# install Envoy Gateway CRDs only - Gateway API CRDs were installed in the introduction guide
helm template eg-crds oci://docker.io/envoyproxy/gateway-crds-helm \
  --version $CHART_VERSION \
  --set crds.gatewayAPI.enabled=false \
  --set crds.envoyGateway.enabled=true \
  | awk 'BEGIN {print_yaml=0} /^---$/ {print_yaml=1} print_yaml {print}' > /tmp/envoy-gateway-crds.yaml

kubectl apply --server-side -f /tmp/envoy-gateway-crds.yaml

# verify Envoy Gateway CRDs, not just Gateway API CRDs
kubectl get crd envoyproxies.gateway.envoyproxy.io
kubectl api-resources --api-group=gateway.envoyproxy.io

helm show chart oci://docker.io/envoyproxy/gateway-helm --version $CHART_VERSION
helm show values oci://docker.io/envoyproxy/gateway-helm --version $CHART_VERSION > kubernetes/gateway-api/envoy/default-values.yaml

helm install envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
  --version $CHART_VERSION \
  --values kubernetes/gateway-api/envoy/values.yaml \
  --namespace envoy-gateway-system \
  --create-namespace \
  --skip-crds

kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

Upgrade:

```shell
# update Envoy Gateway CRDs first
helm template eg-crds oci://docker.io/envoyproxy/gateway-crds-helm \
  --version $CHART_VERSION \
  --set crds.gatewayAPI.enabled=false \
  --set crds.envoyGateway.enabled=true \
  | awk 'BEGIN {print_yaml=0} /^---$/ {print_yaml=1} print_yaml {print}' > /tmp/envoy-gateway-crds.yaml

kubectl apply --server-side -f /tmp/envoy-gateway-crds.yaml

helm upgrade envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
  --values kubernetes/gateway-api/envoy/values.yaml \
  --version $CHART_VERSION \
  --namespace envoy-gateway-system \
  --skip-crds

kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

### Configuration

Most of the Gateway API controllers are installed using `helm`. </br>
Before we install it, let's take a look at the [default-values.yaml](./default-values.yaml)

### Check Installation

Now we should have the Envoy gateway API controller up and running. </br>
This is not the gateway itself, but the controller that will manage the CRDs we get access to and implement some gateway API CRDs. </br>

```shell
# check the controller pods
kubectl -n envoy-gateway-system get pods

# check the controller pod logs 
kubectl -n envoy-gateway-system logs -l app.kubernetes.io/instance=envoy-gateway

```

## Install an Envoy Gateway Class

```shell
# apply 01
kubectl apply -f kubernetes/gateway-api/envoy/01-gatewayclass.yaml

# verify 01 - GatewayClass is cluster-scoped, so there is no namespace flag
kubectl get gatewayclass
kubectl describe gatewayclass envoy
```

## Install an Envoy Gateway

```shell
# apply 02
kubectl apply -f kubernetes/gateway-api/envoy/02-gateway.yaml

# verify 02 - Gateway is namespaced, in this sample it lives in default
kubectl get gateway -n default
kubectl describe gateway gateway-api -n default
```

The current sample `Gateway` now has three listeners:

* `http` on port `80`
* `https` on port `443` for `example-app.com`
* `https-goexample` on port `443` for `goexample-app.com`

The two HTTPS listeners intentionally share port `443`.
They are separated by hostname and certificate, so a second external port is not required.

When we apply the gateway, we get a new gateway api pod. 

At this stage, Envoy Gateway may create a generated service name such as `envoy-default-gateway-api-<hash>`. </br>
Do not assume the service is called `envoy-gateway-default` until the custom `EnvoyProxy` configuration in `02.1-gateway-config.yaml` has been applied successfully. </br>
When port-forwarding on macOS, avoid local ports `80` and `443` unless you are running with elevated privileges. Use non-privileged local ports such as `8080:80` and `8443:443` instead. </br>

If you want browser testing to be more stable than `kubectl port-forward`, see the detailed walkthrough in [`./k8sknowledgedoc/stable-browser-testing-on-macos-kind.md`](./k8sknowledgedoc/stable-browser-testing-on-macos-kind.md). </br>

```shell
# check the new gateway-api pod
kubectl -n envoy-gateway-system get pods

# we also have a new service
kubectl -n envoy-gateway-system get svc
```

### Additional HTTPS hostname for the Go app

The sample now includes a dedicated HTTPS listener for the Go application hostname `goexample-app.com`.

That listener uses a separate TLS secret:

```shell
kubectl create secret tls secret-goexample-tls -n default \
  --cert /path/to/goexample-app.com.fullchain.pem \
  --key /path/to/goexample-app.com.key.pem
```

Replace the placeholder paths with the actual certificate and key file locations on your machine.

The `Gateway` listener lives in [`./02-gateway.yaml`](./02-gateway.yaml):

```yaml
- name: https-goexample
  hostname: goexample-app.com
  protocol: HTTPS
  port: 443
  tls:
    mode: Terminate
    certificateRefs:
      - name: secret-goexample-tls
        namespace: default
```

The Go application now uses **split routes** in [`../07-httproute-tls.yaml`](../07-httproute-tls.yaml):

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

This split is necessary because `HTTPRoute.hostnames` applies to the whole route.
One `HTTPRoute` cannot use `example-app.com` for its HTTP parent and `goexample-app.com` for its HTTPS parent at the same time.

This is a good pattern when:

* you want standard HTTPS on port `443`
* each hostname has its own certificate
* and you want the listener boundary to be explicit at the `Gateway` level
* you want different hostnames for the HTTP and HTTPS entry points of the same backend

### Gateway Configuration

`EnvoyProxy` is the CRD that allows us to configure each gateway API instance. </br>
For example, we can change the `deployment` or `service` of the instance like so:

```shell
kubectl apply -f kubernetes/gateway-api/envoy/02.1-gateway-config.yaml

# we should see 2 replicas 
kubectl -n envoy-gateway-system get deploy
kubectl -n envoy-gateway-system get pods

# port forward for access
# if the EnvoyProxy config applied successfully, the service should be envoy-gateway-default
# otherwise, use the generated service name shown by kubectl get svc
kubectl -n envoy-gateway-system get svc
kubectl -n envoy-gateway-system port-forward svc/<service-name> 8080:80
```

## HTTP Traffic management

Feel free to quickly run through the basic [traffic management table](../README.md#traffic-management-features--http-routes) for using `HTTPRoute` routing for traffic. </br>

<i>Note: HTTPRoute features are not specific to this controller and should be available to any other gateway API controller that you choose.</i>

## Gateway API Extensions

Envoy Gateway has its own extensions on top of the native Gateway API features we've seen so far, using custom CRD's.

### Client Traffic Policies

#### Connection Limit

Envoy allows us to limit connections to gateways & listeners. </br>
We can use tools like `hey` to demonstrate connection limits.

```shell

hey -c 10 -q 1 -z 10s -host "example-app.com" http://localhost:8080/api/go/status

kubectl apply -f kubernetes/gateway-api/envoy/08-clientpolicy-connectionlimit.yaml

hey -c 10 -q 1 -z 10s -host "example-app.com" http://localhost:8080/api/go/status
```

#### Mutual TLS

Currently we have a listener for TLS (HTTPs). For normal TLS, the client does not have to pass a private key. </br>
Envoy supports mutual TLS

```shell

# tell Envoy about the CA cert 
kubectl create secret generic secret-tls-ca --from-file=rootCA.pem=kubernetes/gateway-api/tls/rootCA.pem

# generate client certs for curl
mkcert -client client-user.com

kubectl apply -f kubernetes/gateway-api/envoy/09-clientpolicy-mtls.yaml

# browser TLS should now be broken "Access to example-app.com was denied"
# This is because a client needs to pass certificates to trust 

curl -v -H "Host: example-app.com" \
--cert kubernetes/gateway-api/tls/client-user.com-client.pem \
--key kubernetes/gateway-api/tls/client-user.com-client-key.pem \
--cacert kubernetes/gateway-api/tls/rootCA.pem \
https://127.0.0.1:8443/api/go/status

```

### Backend Traffic Policies

#### Local and Global Rate limits

We can rate limit each HTTP route using Local Rate Limit:

```shell

kubectl apply -f kubernetes/gateway-api/envoy/10-backendpolicy-ratelimit.yaml

for i in {1..4}; do
curl -I --header "x-user-id: one" http://example-app.com/api/go/status ; sleep 1;
done

```

The current repo also includes a minimal local rate-limit reproduction manifest:

```shell
kubectl apply -f kubernetes/gateway-api/envoy/10.1-backendpolicy-ratelimit-minimal.yaml

for i in {1..4}; do
  curl -I --header "Host: example-app.com" http://127.0.0.1:8080/ratelimit-minimal ; sleep 1;
done
```

This minimal repro isolates local rate limiting from the more complex split-route and TLS examples.

Important note from this session:

* Local rate limiting in this repo was verified working after upgrading Envoy Gateway to `v1.9.0`.
* On `v1.8.0`, the policy could be accepted by Kubernetes but still fail to appear in the generated Envoy xDS for this setup.
* If you need to debug that case, see [`./k8sknowledgedoc/testing-and-troubleshooting.md`](./k8sknowledgedoc/testing-and-troubleshooting.md).

### Security Policies

### CORS 

```shell

# port-forward to set pre-cors
kubectl port-forward svc/web-app 8000:80

kubectl apply -f kubernetes/gateway-api/envoy/11-securitypolicy-cors.yaml

```

### Basic Auth

```shell

#cleanup client policies to avoid conflict
kubectl delete clienttrafficpolicies mtls-policy
kubectl delete SecurityPolicy go-route-cors

# port-forward to local 8443 using the current gateway service name
kubectl -n envoy-gateway-system get svc
kubectl -n envoy-gateway-system port-forward svc/<service-name> 8443:443

#install htpasswd (alpine)
apk add apache2-utils
htpasswd -cbs .htpasswd bob password123

kubectl create secret generic basic-auth --from-file=.htpasswd

kubectl apply -f kubernetes/gateway-api/envoy/12-securitypolicy-basicauth.yaml

```

### API Auth 

```shell

#cleanup previous basic auth policy
kubectl delete SecurityPolicy go-route-basicauth

kubectl apply -f kubernetes/gateway-api/envoy/13-securitypolicy-apiauth.yaml

# port-forward to local 8443 using the current gateway service name
kubectl -n envoy-gateway-system get svc
kubectl -n envoy-gateway-system port-forward svc/<service-name> 8443:443

```

You can see `Client authentication failed.` in the browser. 
We need to pass the header 

```
curl -v -H "Host: example-app.com" \
-H 'x-api-key: secret123' \
--cert kubernetes/gateway-api/tls/client-user.com-client.pem \
--key kubernetes/gateway-api/tls/client-user.com-client-key.pem \
--cacert kubernetes/gateway-api/tls/rootCA.pem \
https://127.0.0.1:8443/api/go/status
```

Logs:

```shell
kubectl -n envoy-gateway-system logs -l app.kubernetes.io/managed-by=envoy-gateway
```
