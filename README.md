# Install Istio on a Minkube Kubernetes local cluster #

In the following document you will find several terms like:

- Kubernetes cluster
- Minikube cluster
- Kubernetes local cluster
- Cluster

All these terms refer to the same thing that is the Kubernetes cluster containing one node hosted on a local VirtualBox VM, created and managed with the CLI tool called Minikube.

## Preparation ##

### Virtual Machine resources ###

Istio running on Minikube will need additional resources on top of what is instaled by default with `minikube start` (2CPU and 4GB RAM).

Istio recommends on their [site](https://istio.io/docs/setup/kubernetes/platform-setup/minikube/) a cluster with 4CPU and 16GB RAM but I managed to install it on a Minikube cluster with **2CPU and 8GB RAM** and it worked with one sample application as seen below.

As your Minikube cluster will grow then you will surely need more resources.

## Installation ##

### Download latest Istio package ###

The following command will download and unpack Istio:

```shell
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.2.4 sh -
```

Istio documentation: https://istio.io/docs/setup/kubernetes/

The directory contains several directories. You will use scripts from *install* and *samples*.

```shell
cd istio-1.2.4

$ ls -lA
total 33
drwxr-xr-x+ 1 bond None     0 Jun 19 00:46 bin
drwxr-xr-x+ 1 bond None     0 Jun 19 00:46 install
-rw-r--r--  1 bond None   602 Jun 19 00:46 istio.VERSION
-rw-r--r--  1 bond None 11343 Jun 19 00:46 LICENSE
-rw-r--r--  1 bond None  6220 Jun 19 00:46 README.md
drwxr-xr-x+ 1 bond None     0 Jun 19 00:46 samples
drwxr-xr-x+ 1 bond None     0 Jun 19 00:46 tools
```

### Create dedicated namespace for Istio ###

```shell
kubectl create namespace istio-system
```

### Install Istio components ###

```shell
kubectl apply -f install/kubernetes/istio-demo.yaml
```

### Checks to be made ###

Check that CRDs are deployed.
They should be either 23 or 28 if cert-manager is enabled.

```shell
kubectl get crds | grep 'istio.io\|certmanager.k8s.io' | wc -l
```

Check Istio services:

```shell
kubectl -n istio-system get svc
```

Check Istio pods:

```shell
kubectl -n istio-system get pods
```

## Application deployment ##

You may use a demo application provided by Istio or you may deploy your own application.

### Deploy demo application ###

The application is called **httpbin** and is provided in the directory samples.

#### Create namespace httpbin ####

Istio does not create a dedicated namespace for it but you will do this:

```shell
kubectl create namespace httpbin
```

#### Apply istio-injection label to namespace httpbin ####

Apply `istio-injection` label to the new namespace in order to make Istio aware that it should manage the traffic of the applications residing into this namespace:

```shell
kubectl label namespace httpbin istio-injection=enabled
```

#### Deploy the httpbin application ####

Deploy the application into the dedicated namespace:

```shell
kubectl -n httpbin apply -f samples/httpbin/httpbin.yaml
```

#### Check the service created ####

```shell
$ kubectl -n httpbin describe svc/httpbin

Name:              httpbin
Namespace:         httpbin
Labels:            app=httpbin
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"httpbin"},"name":"httpbin","namespace":"httpbin"},"spec"...
Selector:          app=httpbin
Type:              ClusterIP
IP:                10.109.227.242
Port:              http  8000/TCP
TargetPort:        80/TCP
Endpoints:         172.17.0.33:80
Session Affinity:  None
Events:            <none>
```

You may observe first 2 lines with the service name and the namespace. These 2 items will be modified into the script that creates the Istio gateway and virtual service (see next).

#### Modify the gateway and virtual service script ####

The file [httpbin-gateway.yaml](httpbin-gateway.yaml) from samples should be modified to reflect the deployment into *httpbin* namespace and to use a local domain name for the application.

The destination.host is composed as follows: `servicename.namespace.svc.cluster.local`

In our case it will be **httpbin.httpbin.svc.cluster.local**.

The suffix `.svc.cluster.local` may be skipped as the cluster should resolve it but it is safer to put it there.

#### Deploy the gateway and virtual service ####

Apply the changes from [`httpbin-gateway.yaml`](httpbin-gateway.yaml):

```shell
kubectl apply -f httpbin-gateway.yaml
```

### Check demo application ###

The application may be checked with a CLI tool or with a browser.

#### Check httpbin application with CLI tool ####

```shell
$ curl -I -HHost:httpbin.local http://cluster:31380

HTTP/1.1 200 OK
server: istio-envoy
date: Fri, 21 Jun 2019 01:03:16 GMT
content-type: text/html; charset=utf-8
content-length: 9593
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 30
```

You may observe in the command that we use:

- header: `Host` with the value `httpbin.local`
  - **httpbin.local** is the value in the local domain that we used as host for our application in the script [`httpbin-gateway.yaml`](httpbin-gateway.yaml). This may be replaced with the domain name used for the application (e.g. httpbin.sample.com) in the real life.
- protocol: http
  - we use HTTP in this example for simplicity
- host: cluster
  - this is the name of Minikube cluster that is added to /etc/hosts with the following command:

    ```shell
    echo -e "\n"$(minikube ip)" cluster" | sudo tee -a /etc/hosts
    ```

- port: 31380
  - this is the port exposed by Istio for HTTP as seen below the node port for *http2*:

    ```shell
    $ kubectl -n istio-system describe svc/istio-ingressgateway|grep NodePort

    NodePort:                 status-port  31377/TCP
    NodePort:                 http2  31380/TCP
    NodePort:                 https  31390/TCP
    NodePort:                 tcp  31400/TCP
    NodePort:                 https-kiali  30737/TCP
    NodePort:                 https-prometheus  32649/TCP
    NodePort:                 https-grafana  32058/TCP
    NodePort:                 https-tracing  32597/TCP
    NodePort:                 tls  32722/TCP
    ```

The application *httpbin* is able to reply also with other HTTP status codes.
Examples below:

```shell
$ curl -I -HHost:httpbin.local http://cluster:31380/status/301

HTTP/1.1 301 Moved Permanently
server: istio-envoy
date: Fri, 21 Jun 2019 01:28:44 GMT
location: /redirect/1
access-control-allow-origin: *
access-control-allow-credentials: true
content-length: 0
x-envoy-upstream-service-time: 2
```

```shell
$ curl -I -HHost:httpbin.local http://cluster:31380/status/404

HTTP/1.1 404 Not Found
server: istio-envoy
date: Fri, 21 Jun 2019 01:28:52 GMT
content-type: text/html; charset=utf-8
access-control-allow-origin: *
access-control-allow-credentials: true
content-length: 0
x-envoy-upstream-service-time: 2
```

#### Check httpbin application with browser ####

In order to check the application with a browser you need to send also the header *Host*.
In order to do this you need to install an extension/addon to your browser.

You may use the extension/addon `ModHeader` that is available for Chrome and Firefox and also other browsers using these engines (e.g. Vivaldi, Opera, Brave).

In ModHeader you should set the header `Host` with the value `httpbin.local` and access the page [http://cluster:31380](http://cluster:31380).

Istio documentation: https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/

## Reverse proxy ##

In order to make an easier way to access the web application, we may create a reverse proxy to work together with the Istio gateway. In this way you won't need to forward headers using browser extension/addon as described above.

With the script [reverse-proxy.yaml](reverse-proxy.yaml) you will create the reverse proxy components:

- an Nginx configured to forward all HTTP traffic addressed to any application `*.local` towards Istio HTTP service. The Nginx will forward also the header *Host* that is necessary in Istio to route the traffic to the proper application.
- a service and an ingress that will collect all HTTP trafic to `*.local`

In reverse-proxy.yaml you may want to modify the version of Nginx to the latest or according to your needs.

```shell
$ kubectl apply -f reverse-proxy.yaml

namespace/reverse-proxy created
configmap/nginx-conf created
deployment.extensions/reverse-proxy created
service/nginx-service created
ingress.extensions/nginx-ingress created
```

Make sure you add `httpbin.local` in your /etc/hosts

```shell
echo -e "\n"$(minikube ip)" httpbin.local" | sudo tee -a /etc/hosts
```

Now you are able to access the application *httpbin* in browser: http://httpbin.local

## Install Istio with istioctl ##

### Copy istioctl to a dir in PATH ###

You will find it in `istio-1.x.y/bin/`

The destination path may be `/usr/local/bin/istioctl`

### Auto completion ###

Copy `istio-1.x.y/tools/istioctl.bash` in a location like `$HOME/.local/bin/istioctl.bash`

Add in .bashrc the line:

```shell
source $HOME/.local/bin/istioctl.bash
```

### List available installation profiles ###

```shell
$ istioctl profile list
Istio configuration profiles:
    default
    demo
    empty
    minimal
    openshift
    preview
    remote
```

### Get profile details ###

```shell
istioctl profile dump default
```

### Install profile default ###

```shell
istioctl install --set profile=default
```

More details at:

<https://istio.io/latest/docs/setup/install/istioctl/>

## Upgrade Istio with istioctl ##

Download the `istioctl` target version for the upgrade from https://github.com/istio/istio/releases

>The installed Istio version is no more than one minor version less than the upgrade version. For example, 1.6.0 or higher is required before you start the upgrade process to 1.7.x.

The upgrade commands should be run using the new version of `istioctl`.

Check current installation:

```shell
$ istioctl x precheck 
✔ No issues found when checking the cluster. Istio is safe to install or upgrade!
  To get started, check out https://istio.io/latest/docs/setup/getting-started/
```

>Traffic disruption may occur during the upgrade process. To minimize the disruption, ensure that at least two replicas of `istiod` are running. Also, ensure that `PodDisruptionBudgets` are configured with a minimum availability of 1.

The **in-place upgrade**

You need to provide in command line the same `set` parameters provided at installation.
If in the installation process was used `--set profile=default` then it should be used also in the upgrade.

```shell
$ istioctl upgrade --set profile=default
This will install the Istio 1.15.0 default profile with ["Istio core" "Istiod" "Ingress gateways"] components into the cluster. Proceed? (y/N) y
✔ Istio core installed    
✔ Istiod installed         
✔ Ingress gateways installed
✔ Installation complete
Making this installation the default for injection and validation.
```

More about the upgrade CLI parameters on https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-upgrade

Check after upgrade:

```shell
$ istioctl verify-install
1 Istio control planes detected, checking --revision "default" only
✔ ClusterRole: istiod-istio-system.istio-system checked successfully
✔ ClusterRole: istio-reader-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istio-reader-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istiod-istio-system.istio-system checked successfully
✔ ServiceAccount: istio-reader-service-account.istio-system checked successfully
✔ Role: istiod-istio-system.istio-system checked successfully
✔ RoleBinding: istiod-istio-system.istio-system checked successfully
✔ ServiceAccount: istiod-service-account.istio-system checked successfully
✔ CustomResourceDefinition: wasmplugins.extensions.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: destinationrules.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: envoyfilters.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: gateways.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: proxyconfigs.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: serviceentries.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: sidecars.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: virtualservices.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: workloadentries.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: workloadgroups.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: authorizationpolicies.security.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: peerauthentications.security.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: requestauthentications.security.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: telemetries.telemetry.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: istiooperators.install.istio.io.istio-system checked successfully
✔ HorizontalPodAutoscaler: istiod.istio-system checked successfully
✔ ClusterRole: istiod-clusterrole-istio-system.istio-system checked successfully
✔ ClusterRole: istiod-gateway-controller-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istiod-clusterrole-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istiod-gateway-controller-istio-system.istio-system checked successfully
✔ ConfigMap: istio.istio-system checked successfully
✔ Deployment: istiod.istio-system checked successfully
✔ ConfigMap: istio-sidecar-injector.istio-system checked successfully
✔ MutatingWebhookConfiguration: istio-sidecar-injector.istio-system checked successfully
✔ PodDisruptionBudget: istiod.istio-system checked successfully
✔ ClusterRole: istio-reader-clusterrole-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istio-reader-clusterrole-istio-system.istio-system checked successfully
✔ Role: istiod.istio-system checked successfully
✔ RoleBinding: istiod.istio-system checked successfully
✔ Service: istiod.istio-system checked successfully
✔ ServiceAccount: istiod.istio-system checked successfully
✔ EnvoyFilter: stats-filter-1.13.istio-system checked successfully
✔ EnvoyFilter: tcp-stats-filter-1.13.istio-system checked successfully
✔ EnvoyFilter: stats-filter-1.14.istio-system checked successfully
✔ EnvoyFilter: tcp-stats-filter-1.14.istio-system checked successfully
✔ EnvoyFilter: stats-filter-1.15.istio-system checked successfully
✔ EnvoyFilter: tcp-stats-filter-1.15.istio-system checked successfully
✔ ValidatingWebhookConfiguration: istio-validator-istio-system.istio-system checked successfully
✔ HorizontalPodAutoscaler: istio-ingressgateway.istio-system checked successfully
✔ Deployment: istio-ingressgateway.istio-system checked successfully
✔ PodDisruptionBudget: istio-ingressgateway.istio-system checked successfully
✔ Role: istio-ingressgateway-sds.istio-system checked successfully
✔ RoleBinding: istio-ingressgateway-sds.istio-system checked successfully
✔ Service: istio-ingressgateway.istio-system checked successfully
✔ ServiceAccount: istio-ingressgateway-service-account.istio-system checked successfully
Checked 15 custom resource definitions
Checked 2 Istio Deployments
✔ Istio is installed and verified successfully
```

More about `in-place upgrade` on https://istio.io/latest/docs/setup/upgrade/in-place/

There is also the `canary upgrade` option: https://istio.io/latest/docs/setup/upgrade/canary/

## Uninstall Istio ##

```shell
istioctl x uninstall --purge
```

```shell
kubectl delete namespace istio-system
```

## Istio for ARM64 ##

Starting with Istio version 1.15 (9-aug-2022) there are ARM64 images provided by Istio:

Check https://hub.docker.com/r/istio/proxyv2/tags
