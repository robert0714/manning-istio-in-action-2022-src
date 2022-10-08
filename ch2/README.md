# Chapter 2 Fist steps with Istio
## 2.1 Deploying Istio on Kubernetes

### 2.1.1 Using Docker Desktop for the examples
> **⚠ NOTE:**   It may be worth giving **Docker 8 GB of memory and four CPUs**. You can do that under the advanced settings in Docker Desktop’s preferences.
```shell
kubectl config get-contexts                          # display list of contexts 
kubectl config current-context                       # display the current-context
kubectl config use-context [my-cluster-name]           # set the default context to my-cluster-name


$ kubectl config set-context docker-desktop

$ kubectl get nodes
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   20m   v1.25.0
```
### 2.1.2 Getting the Istio distribution
According [Support status of Istio releases](https://istio.io/latest/docs/releases/supported-releases/#support-status-of-istio-releases)

| Version | Currently Supported  | Release Date      | End of Life              | Supported Kubernetes Versions | Tested, but not supported          |
|---------|----------------------|-------------------|--------------------------|-------------------------------|------------------------------------|
| master  | No, development only |                   |                          |                               |                                    |
| 1.15    | Yes                  | August 31, 2022   | ~March 2023 (Expected)   | 1.22, 1.23, 1.24, 1.25        | 1.16, 1.17, 1.18, 1.19, 1.20, 1.21 |
| 1.14    | Yes                  | May 24, 2022      | ~January 2023 (Expected) | 1.21, 1.22, 1.23, 1.24        | 1.16, 1.17, 1.18, 1.19, 1.20       |
| 1.13    | Yes                  | February 11, 2022 | Oct 12, 2022             | 1.20, 1.21, 1.22, 1.23        | 1.16, 1.17, 1.18, 1.19             |

We use the istioctl command-line tool to install Istio. To do that, download the Istio 1.13.0 distribution from the Istio release page at https://github.com/istio/istio/releases and download the distribution for your operating system. 

```shell
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.15.1 sh -

cd istio-1.15.1/
~/istio-1.15.1$ ls -la
total 48
drwxr-x---  6 robert0714 robert0714  4096  9月 24 06:04 .
drwxr-xr-x 69 robert0714 robert0714  4096 10月  8 11:28 ..
drwxr-x---  2 robert0714 robert0714  4096  9月 24 06:04 bin
-rw-r--r--  1 robert0714 robert0714 11348  9月 24 06:04 LICENSE
drwxr-xr-x  5 robert0714 robert0714  4096  9月 24 06:04 manifests
-rw-r-----  1 robert0714 robert0714   925  9月 24 06:04 manifest.yaml
-rw-r--r--  1 robert0714 robert0714  6016  9月 24 06:04 README.md
drwxr-xr-x 24 robert0714 robert0714  4096  9月 24 06:04 samples
drwxr-xr-x  3 robert0714 robert0714  4096  9月 24 06:04 tools

~/istio-1.13.8$ ./bin/istioctl  version
no running Istio pods in "istio-system"
1.13.8

./bin/istioctl x precheck
✔ No issues found when checking the cluster. Istio is safe to install or upgrade!
  To get started, check out https://istio.io/latest/docs/setup/getting-started/
```
Add the istioctl client to your path (Linux or macOS):
```shell
$ export PATH=$PWD/bin:$PATH
```
### 2.1.3 Installing the Istio components into Kubernetes
The official method for any real installation of Istio is to use **istioctl**, **istio-operator**, or **Helm**. 

For this book, we use **istioctl** and various pre-curated profiles to take a step-by-step, incremental approach to adopting Istio. To perform the demo install, use the **istioctl** CLI tool as shown next:
```shell
$  istioctl install --set profile=demo -y 
```
Uninstall Istio:  
* https://istio.io/v1.13/docs/setup/install/istioctl/ 
* https://istio.io/latest/docs/setup/install/istioctl/
```shell
$  istioctl x uninstall --purge
```
To Verify:
```shell
$ istioctl verify-install
```

Finally, we need to install the control-plane supporting components. These components are not strictly required but should be installed for any real deployment of Istio. The versions of the supporting components we install here are recommended for demo purposes only, not production usage. From the root of the Istio distribution you downloaded, run the following to install the example supporting components:

```shell
$ kubectl apply -f ./samples/addons
```

## 2.3 Deploying your first application in the service mesh
[source code](/services/)

* Creating default namespace `istioinaction` 
```shell
$  kubectl create namespace istioinaction
$  kubectl config set-context $(kubectl config current-context) \
 --namespace=istioinaction
```
* Using Catalog service : [$SRC_BASE/services/catalog/kubernetes/catalog.yaml](/services/catalog/kubernetes/catalog.yaml)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: catalog
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: catalog
  name: catalog
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app: catalog
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: catalog
    version: v1
  name: catalog
spec:
  replicas: 1
  selector:
    matchLabels:
      app: catalog
      version: v1
  template:
    metadata:
      labels:
        app: catalog
        version: v1
    spec: 
      serviceAccountName: catalog
      containers:
      - env:
        - name: KUBERNETES_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: istioinaction/catalog:latest
        imagePullPolicy: IfNotPresent
        name: catalog
        ports:
        - containerPort: 3000
          name: http
          protocol: TCP
        securityContext:
          privileged: false
```
Before we deploy this, however, we want to **inject the Istio service proxy** so that this service can participate in the service mesh. From the root of the source code, run the istioctl command we introduced earlier:
```
$ istioctl kube-inject -f services/catalog/kubernetes/catalog.yaml
```

If you look through the previous command’s output, the YAML now includes a few extra containers as part of the deployment. Most notably, you should see the following:
```yaml
      - args:
        - proxy
        - sidecar
        - --domain
        - $(POD_NAMESPACE).svc.cluster.local
        - --proxyLogLevel=warning
        - --proxyComponentLogLevel=misc:error
        - --log_output_level=default:info
        - --concurrency
        - "2"
        env:
        - name: JWT_POLICY
          value: third-party-jwt
        - name: PILOT_CERT_PROVIDER
          value: istiod
        - name: CA_ADDR
          value: istiod.istio-system.svc:15012
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
...
        image: docker.io/istio/proxyv2:1.15.1
        name: istio-proxy
```
We could deploy the YAML file created by the kube-inject command directly; however, we are going to take advantage of Istio’s ability to automatically inject the sidecar proxy.

To enable **automatic injection**, we label the istioinaction namespace with ``istio-injection=enabled``:
```shell
$ kubectl label namespace istioinaction istio-injection=enabled
```
Now let’s create the catalog deployment:
