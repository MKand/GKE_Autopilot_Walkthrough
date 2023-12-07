# Autopilot demo
This repo demonstrates some of the capabilities of GKE Autopilot. Please note that this is **not** an official resource, so please refer to the documentation on GKE Autopilot on Google Cloud's official site.

## Ensure Gateway API is installed on the cluster. 
Demo uses the global external loadbalancer class.
First, ensure that Gateway API is enabled on the cluster by following [these](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-gateways#enable-gateway) instructions.

## Deploy simple application
The application consists of the following:
1. An ngnix deployment.
2. A service.
3. Namespaces for the nginx deployment and Gateway resource.
4. A Gateway resource
5. An HttpRoute to bring traffic into the nginx service.

Deploy the application by running

```sh
kubectl apply -f k8s/step1
```

## Verify the nodes in the cluster
```sh
kubectl get nodes
```
Get the node type provisioned
```sh
kubectl get nodes -o json|jq -Cjr '.items[] | .metadata.name," ",.metadata.labels."beta.kubernetes.io/instance-type"," ",.metadata.labels."beta.kubernetes.io/arch", "\n"'|sort -k3 -r
```

## Autopilot resource requests
In autopilot you are billed based on the resources (like CPU, memory and ephemeral storage) your pods request. There are some behaviours with respect to these requests that autopilot introduces. Some of these concepts are introduced using demos. For official documentation around this, please consult this [page](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-resource-requests).
### View resource requests and limits set by default

```sh
kubectl describe deploy/nginx-deployment -n nginx
```
As we did not specify resource limits/requests, a default value is set by GKE Autopilot.
This value will be the minumum requestable value which is 0.5vCPU and 2Gi memory. The values can be found [here](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-resource-requests#compute-class-defaults).
Also note that the required minimum values are [different](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-resource-requests) for *Daemonset pods* and all *other pods* and also vary based on the selected compute class.

### Setting the resource values manually in the deployment spec
In *k8s/step2/deployment.yaml*, you will notice that we are setting resource limits and requests for CPU and memory. Another thing to note is that the limit values (1vCPU, 4Gi) are higher than the request values (0.5vCPU, 2Gi) respectively.

Let us apply this

```sh
kubectl apply -f k8s/step2/deployment.yaml
```
The deployment will be updated with a **warning**.
> WARNING: autopilot-default-resources-mutator:Autopilot updated Deployment nginx/nginx-deployment: adjusted resources to meet requirements for containers [nginx] (see http://g.co/gke/autopilot-resources)
deployment.apps/nginx-deployment configured

Examining the specification of the deployment will show that Autipilot adjusted the limits to match the requests. 
```sh
kubectl describe deploy/nginx-deployment -n nginx
```
```sh
   Limits:
      cpu:                0.5
      memory:             2Gi
    Requests:
      cpu:                0.5
      memory:             2Gi
```
In short, Autopilot doesn't allow *bursting*. You can read more about it [here](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-resource-requests#resource-limits).

### Autopilot resource values require to adhere to a certain ratio between memory and CPU
In *k8s/step3/deployment.yaml*, you will notice that we are setting resource requests for CPU and memory. Here the CPU:memory ratio is 1:10 and is outside the allowed range (1vCPU and 10Gi). Apply this new deployment.
```sh
kubectl apply -f k8s/step3/deployment.yaml
```
The deployment will be updated with a **warning**.
> WARNING: autopilot-default-resources-mutator:Autopilot updated Deployment nginx/nginx-deployment: adjusted resources to meet requirements for containers [nginx] (see http://g.co/gke/autopilot-resources)
deployment.apps/nginx-deployment configured.

The resource requests and limits will be modified and applied as follows:
```sh
   Limits:
      cpu:                1.75
      memory:             10Gi
    Requests:
      cpu:                1.75
      memory:             10Gi
```

The minimum and maximum ratios required for autopilot can be found [here](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-resource-requests#compute-class-min-max)