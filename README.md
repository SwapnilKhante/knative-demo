# knative



## Getting started ##
[Knative](https://knative.dev/docs/) is an open-source project which adds components for deploying, running, and managing serverless applications on Kubernetes. We can package our services or functions as a container image and hand it over to `Knative`. 

`Knative` then runs the container for a specific service only when it needs to.

The core architecture of `Knative` comprises two broad components - `Serving`, and `Eventing`, In this tutorial we will focus on `Serving`.

We will setup a serverless platform on top of a k3d cluster

## Setting up a K3d cluster on localhost ##

### Prerequisite ###
 -  Windows 10 Pro with `Hyper-V`
 -  Docker Desktop (2.5.x)

1. Enable  `Hyper-V` (If not already enabled - run following command from powershell as admin)

```
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

2. Verify Docker Installation

```
docker run hello-world
```

3. Install [Chocolatey](https://chocolatey.org/why-chocolatey) -

      - [Install chocolatey](https://chocolatey.org/install#individual)
      - verify installation
      ```
      PS C:\Users\jyee\Desktop\k3s-rancher> choco list --local-only
      Chocolatey v0.10.15
    chocolatey 0.10.15
    1 packages installed.

      ```

4. Install `Kubectl` using `chocolatey`

```
choco install kubernetes-cli -y
```

5. Install `k3d` and verify
```
> choco install k3d

> k3d version
k3d version v5.4.1
k3s version v1.22.7-k3s1 (default)

```

## Setup Knative on K3d

- Creating a cluster with `K3d` by default installs [`Traefik`](https://traefik.io/) which is a modern HTTP reverse proxy and load balancer that makes deploying microservice easy.
- For installing `Knative` we will also need to choose a networking layer. Many online articles will make the use of [`Istio`](https://istio.io/) as a de-facto service mesh however `Istio` has quite large resource requirements which might not be suitable for local deployments.
- Here we will use [Contour](https://projectcontour.io/) as our networking layer
- For `Contour` and `Knative` to work locally in k3d we'll need to create our cluster without `traefik`.

1. Create K3d cluster without `traefik`

```
k3d cluster create --port 8082:30080@agent:0 -p 8081:80@loadbalancer --agents 2 --k3s-arg "--disable=traefik@server:0"
```

2. Install the required custom resources by running the command:
```
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.4.0/serving-crds.yaml
```
3. Install the core components of Knative Serving by running the command:

```
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.4.0/serving-core.yaml
```

4. Install a properly configured Contour by running the command:

```
kubectl apply -f https://github.com/knative/net-contour/releases/download/knative-v1.4.0/contour.yaml
```

5. Install the Knative Contour controller by running the command:
```
kubectl apply -f https://github.com/knative/net-contour/releases/download/knative-v1.4.0/net-contour.yaml
```

6. Configure Knative Serving to use Contour by default by running the command:
```
kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress-class":"contour.ingress.networking.knative.dev"}}'
  ```

### Hello From Kubernetes Serverless World

For testing purposes let's do a hello world from the Knative samples. In Knative there's another new resource called Service, not to be mixed up with the Kubernetes resource Service. These Services are used to manage the core Kubernetes resources

```
> kubectl apply -f knative-service.yaml

service.serving.knative.dev/knative-helloworld created

```

We don't have DNS so accessing the application isn't easy. We'll have to set the Host parameter for our requests. Find out the host from:

```
kubectl get ksvc
NAME                 URL                                             LATESTCREATED           LATESTREADY             READY   REASON
knative-helloworld   http://knative-helloworld.default.example.com   knative-helloworld-v1   knative-helloworld-v1   True
```

Now we can see that there are no pods running. There may be one as Knative spins one pod during the creation of the service, wait until no resources are found

```
kubectl get po
No resources found in default namespace.
```

and when we send a request to the application -

```
curl -H "Host: knative-helloworld.default.example.com" http://localhost:8081
```

it works and instantly and new pod is ready.

```
kubectl get po
NAME                                                READY   STATUS    RESTARTS   AGE
knative-helloworld-v1-deployment-597d598958-dfrmz   2/2     Running   0          4s
```

---

