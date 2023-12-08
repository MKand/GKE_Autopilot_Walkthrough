# Autopilot demo
This repo demonstrates some of the capabilities of GKE Autopilot. Please note that this is **not** an official resource, so please refer to the documentation on GKE Autopilot on Google Cloud's official site.

## Setup your autopilot cluster
Follow the instructions [here](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-an-autopilot-cluster) to set up the autopilot cluster and connect to it from your kubernetes client.

## Ensure Gateway API is installed on the cluster. 
This demo uses the global external loadbalancer class.
First, ensure that Gateway API is enabled on the cluster by following [these](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-gateways#enable-gateway) instructions.


## Verify the nodes in the cluster
```sh
kubectl get nodes
```
Get the node type provisioned
```sh
kubectl get nodes -o json|jq -Cjr '.items[] | .metadata.name," ",.metadata.labels."beta.kubernetes.io/instance-type"," ",.metadata.labels."beta.kubernetes.io/arch", "\n"'|sort -k3 -r
```
Without any workloads deployed, the cluster will likely be a very small one with a node-pool of size 1. By default nodes will be of the smaller *General* compute-type which uses E2 machine types.

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

### Autopilot resources can only change in specific value increments
In *k8s/step4/deployment.yaml*, we will change the total amount of CPU requested to 0.8vCPU. The amount of memory requested has to be an integer (so we will use an integer value of 2Gi).  Apply this new deployment.
```sh
kubectl apply -f k8s/step4/deployment.yaml
```
The deployment will be updated with a **warning**.
> WARNING: autopilot-default-resources-mutator:Autopilot updated Deployment nginx/nginx-deployment: adjusted resources to meet requirements for containers [nginx] (see http://g.co/gke/autopilot-resources)
deployment.apps/nginx-deployment configured.

The resource requests and limits will be modified and applied as follows:
```sh
   Limits:
      cpu:                1
      memory:             2Gi
    Requests:
      cpu:                1
      memory:             2Gi
```
You will notice that autopilot does not accept a value of 0.8vCPU and changes the number to 1vCPU. In fact the same will happen even if you choose a number greater than 0.75 and less 1. This is to illustrate that CPU resources must be increased in increments of 0.25 CPUs. You can read more about this [here](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-resource-requests#min-max-requests). Please note that memory requests will always need to be integers.

## Node-pool behaviour
Autopilot will scale-up machines and add more nodes based on the requirements of your workloads. Scaling operations may take some time when new nodes needs to be added.

### Scaling nodes out and up
We will create more resource requests than a single e2-standard-4 machine type can handle by requesting 2 replicas of 3vCPUs pods. 
```sh
kubectl apply -f k8s/step5/deployment.yaml
```
Immediately after applying this (assuming that you have none or very small workloads running before the apply), you will notice that pods will be in the *pending* state for a few minutes. You can verify this by running
```sh
kubectl get pods -n nginx -w
```
After the pods are in a running state, let us check the nodes in the cluster by running
```sh
kubectl get nodes -o json|jq -Cjr '.items[] | .metadata.name," ",.metadata.labels."beta.kubernetes.io/instance-type"," ",.metadata.labels."beta.kubernetes.io/arch", "\n"'|sort -k3 -r
```
You will notice that there are atleast 2 nodes in the cluster, so autopilot scaled-out the number of instances to be able to provision your workloads. 

Now, we will try scaling up nodes from the e2-standard-4 type. To do this we will ask for  9vCPU per pod. 
```sh
kubectl apply -f k8s/step6/deployment.yaml
```
Again, immediately after applying this, you will notice that new pods will be in the *pending* state for a few minutes. You can verify this by running
```sh
kubectl get pods -n nginx -w
```
Once all the new pods are in the running state (and the old pods are terminated), let us check the nodes again by running 
```sh
kubectl get nodes -o json|jq -Cjr '.items[] | .metadata.name," ",.metadata.labels."beta.kubernetes.io/instance-type"," ",.metadata.labels."beta.kubernetes.io/arch", "\n"'|sort -k3 -r
```
You will notice that there will be atleast 2 new nodes added to the clusters (and the old nodes may be removed). The new nodes will be of a larger machine type that e2-standard-4 (likely will be e2-standard-16). It is important to note that you are only paying for the resources requested by your workload pods and not number of nodes or the underlying machine type of your nodes.

Let us delete the deployment by running
```sh
kubectl delete deploy/nginx-deployment -n nginx
```
You may continue to watch the nodes in your cluster. Autopilot may scale them down after a few minutes. Again, you will not be billed for these nodes that are running in the cluster.

### Using different compute classes
By default, autopilot uses machines from the *General* compute class to run your workloads. However, you can select from 3 different [compute classes](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-compute-classes#when-to-use) (Balanced, General Purpose, Scale out), or [GPU types](https://cloud.google.com/kubernetes-engine/docs/how-to/autopilot-gpus) (nvidia-l4, nvidia-tesla-t4, nvidia-tesla-a100, nvidia-a100-80gb).
The classes can be selected by specifying the type either using a *NodeSelector* or *NodeAffinity* field in the pod spec.
In this step will will select compute class *Balanced* to deploy our workload.
```sh
kubectl apply -f k8s/step7/deployment.yaml
```
Again, it will take a while (approx a minute) for the pods to start *running* as Autopilot will create a node(s) from the required compute class.
Once the pods are up and running, check the nodes that are running in the cluster.
```sh
kubectl get nodes -o json|jq -Cjr '.items[] | .metadata.name," ",.metadata.labels."beta.kubernetes.io/instance-type"," ",.metadata.labels."cloud.google.com/compute-class", "\n"'|sort -k3 -r
```
You should find atleast one node in the cluster of machine type *n2d* and with the label  *cloud.google.com/compute-class* with value *Balanced*. So autopilot automatically creates the node of the required compute class and labels it with the value expected by the workload (as defined by the nodeSelector or nodeAffinity field).

*Note*: the price of resource requests that Autopilot depends on the [compute class](https://cloud.google.com/kubernetes-engine/pricing#autopilot_mode) used.

## Workload seperation
Workload seperation can be performed when you have workloads that perform different roles and shouldn't run on the same underlying machines. In this example, we will be seperating *dev* workloads from *prod* workloads on different nodes.

In the example, we have 2 seperate deployments of the nginx container, one belonging to *dev* and the other prod. We want to ensure that both these sets of pods are deployed onto different nodes. 
In order to do this, we specify the toleration on the pods that will seperate out nodes that allow *dev* pods from those that allow *prod* pods.  Then Autopilot ensures that nodes with the required *taints* are spun up and that only pods with tolerations to those taints are allowed.

Looking at *k8s/step8/deployment_prod.yaml* and *k8s/step8/deployment_dev.yaml* will show you that the *prod* deployment podSpec shows a toleration for nodes with the taint "env=prod:NoSchedule" while the *dev* deployment podSpec shows a toleration for nodes with the taint "env=dev:NoSchedule". On top of this both podSpecs also specify that they must be schedule on nodes that have the labels "env=prod" and "env=dev" respectively, to ensure workload seperation. Without the latter specification, it could happen that both sets of pods are scheduled on an untainted node, should one exist in the cluster.

Let's start by deleting the existing deployment.
```sh
kubectl delete deploy/nginx-deployment -n nginx
```
Let's apply the new deployments.
```sh
kubectl apply -f k8s/step8
```
Wait for all the new pods to spin up.
After they are all in running state, let us examine the taints on the nodes in the cluster by running the following command.
```sh
kubectl get nodes -o json | jq '.items[] | .metadata.name," ", .spec.taints'
```
You should see that there are (alteast) 2 nodes with 2 different taints in the cluster as defined earlier. The output should look something like this, the actual node-names will be different.
```sh
"gk3-demo-autopilot-clust-nap-1l1nplkd-ab97db75-mphh"
[
  {
    "effect": "NoSchedule",
    "key": "env",
    "value": "prod"
  }
]
"gk3-demo-autopilot-clust-nap-fkequtrn-9618d805-vqwc"
[
  {
    "effect": "NoSchedule",
    "key": "env",
    "value": "dev"
  }
]
```
Now let us verify that the pods are running on the right nodes by running.
```sh
kubectl get pods -n nginx -o wide
```
The output show a list of the pods with the names of the nodes they are running on. By cross-checking with the previous output which associated node names with the taints on them, you should be able to verify that workload seperation is indeed achieved.

