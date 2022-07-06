# kubectl-101

## kubectl basics

### Explore your cluster

To see the nodes that make up your cluster, use:

```
kubectl get nodes
```

You should see something like:
```
NAME                                              STATUS   ROLES    AGE   VERSION
ip-192-168-11-255.eu-central-1.compute.internal   Ready    <none>   8d    v1.21.12-eks-5308cf7
ip-192-168-51-107.eu-central-1.compute.internal   Ready    <none>   8d    v1.21.12-eks-5308cf7
ip-192-168-79-95.eu-central-1.compute.internal    Ready    <none>   8d    v1.21.12-eks-5308cf7
```
**Note**: If you are using Amazon EKS, you will only see details for your worker nodes. If you are running standard (self-managed) Kubernetes then you will see both workewr nodes and control plane nodes.

To get a bit more info about nodes, try:
```
kubectl get nodes -o wide
```

To get detailed info about a specific node, use:
```
kubectl get node <node-name> -o yaml
```

To see what pods are running on your cluster, use:

```
kubectl get pods -A
```

The output looks something like:
```
NAMESPACE          NAME                                           READY   STATUS    RESTARTS   AGE
kube-system        aws-node-9zjrl                                 1/1     Running   0          8d
kube-system        aws-node-rrpmd                                 1/1     Running   0          8d
kube-system        aws-node-wgr84                                 1/1     Running   0          8d
kube-system        coredns-745979c988-nbfxz                       1/1     Running   0          8d
kube-system        coredns-745979c988-sfwwj                       1/1     Running   0          8d
kube-system        kube-proxy-cf8bn                               1/1     Running   0          8d
kube-system        kube-proxy-rjvlb                               1/1     Running   0          8d
kube-system        kube-proxy-z9z99                               1/1     Running   0          8d
```

You can see that all these pods are running in the namespace `kube-system`. In the next section you will see how you can use namespaces to segment your cluster for management purposes.

To see pods running on a specifc node:
```
kubectl get pods -A --field-selector spec.nodeName=<node>
```

### Starting a new pod

Run a NGINX server using:
```
kubectl run nginx --image=nginx
```

You should see:
```
pod/nginx created
```

Get a summary of your pod status
```
kubectl get pod nginx
```

Add `-o wide` to see which node it has been scheduled on:
```
kubectl get pod nginx -o wide
```

Verify that the NGINX server is running:

Run
```
kubectl port-forward nginx 8080:80
```
This will forward all traffic to port 8080 on the local host to port 80 in the nginx pod. Note this command blocks until you want to stop forwarding.

Open another terminal window and use
```
curl 127.0.0.1:8080/
```
to connect to the NGINX server running in the pod.

When done, return to the terminal where `port-forward` is running and hit *Ctrl-C* to stop forwarding.

To delete the pod:
```
kubectl delete pod nginx
```

### Run pod using Amazon ECR public registry

By default, images are pulled from the public [Docker Hub](https://hub.docker.com/) registry. Note that there are limits in place on the number
of image pulls:
- For anonymous users, the rate limit is set to 100 pulls per 6 hours per IP address.
- For authenticated users, it is 200 pulls per 6 hour period.

You can use alternate public registries, or your own private registry, by specifying the full path name to the image. For example:

```
kubectl run nginx --image=public.ecr.aws/docker/library/nginx:stable-perl
```

Check out the ECR gallery at https://gallery.ecr.aws/.

### Namespaces

You can use namespaces to segment your cluster. In the earlier example, you created a pod in the `default` namespace.

To list namespaces use:
```
kubectl get namespaces
```

To list pods in the `kube-system` namespace, use:
```
kubectl get pods -n kube-system
```

To create your own namespace, use:
```
kubectl create namespace my-namespace
```

Run an NGINX server pod in your namespace using:
```
kubectl run nginx --image=public.ecr.aws/docker/library/nginx:stable-perl -n my-namespace
```

Try:
```
kubectl get pods -A
```
to see all pods across all namespaces, and you should see your pod in your namespace.

To delete the pod, use:
```
kubectl delete pod nginx -n my-namespace
```

To delete the namespace, use:
```
kubectl delete namespace my-namespace
```

### Using manifest files

While it is possible to create and manage resources directly from the command line, this starts to become impractical for more complex configurations at scale.

Kubernetes object configurations are typically captured using a manifest file containing one or more YAML documents.
These files can be stored in a git repo and under version control, just like application code.

Let's reproduce the previous exmample using a manifest file. Create a file `nginx.yaml` with the following contents:
```
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: my-namespace
spec:
  containers:
  - image: public.ecr.aws/docker/library/nginx:stable-perl
    name: nginx
```

Apply the manifest using:
```
kubectl apply -f nginx.yaml 
```

This will create a namespace and pod. Try applying the manifest again, and see what happens.

To delete the pod and namespace, use:
```
kubectl delete -f nginx.yaml
```

## ReplicaSets and Deployments

When running services at scale, it is often necessary to run multiple copies of a pod to enable horizontal scaling. Further, mechanisms are needed
to ensure that the number of pods can be adjusted as needed, and to support the deployment of new versions.

### Creating a ReplicaSet

Kubernetes uses ReplicaSets to maintain a specified number of replicas of a pod.

Create the following manifest in a file `frontend-rs.yaml`
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: public.ecr.aws/docker/library/nginx:stable-perl
```

Apply the file, using:
```
kubectl apply -f frontend-rs.yaml
```
and immediately follow with:
```
kubectl get pods
```

You should see three pods being created
```
NAME             READY   STATUS              RESTARTS   AGE
frontend-72hhc   0/1     ContainerCreating   0          6s
frontend-bzqwg   0/1     ContainerCreating   0          6s
frontend-zzjgd   0/1     ContainerCreating   0          6s
```
Note that pods have been named automatically to ensure they all receive a unique name.

Check the status again a few seconds later, and you should see all pods in the `Running` state. You can also use the `-o wide` option to
see which nodes the pods have been placed on.

You can also check the status of the ReplicaSet itself:
```
kubectl get replicaset frontend
```
and you should see:
```
NAME       DESIRED   CURRENT   READY   AGE
frontend   3         3         3       3m32s
```
Try adding `-o wide` for more info, and `-o yaml` for even more info.

### Scaling pods using a ReplicaSet

Suppose you now needed to scale out your service to 5 pods.

To do this, edit the manifest `frontend-rs.yaml` and change the `replicas` attribute from
3 to 5. Apply the manifest again and check the pod status. You should now see five pods!

```
NAME             READY   STATUS    RESTARTS   AGE
frontend-5pjw2   1/1     Running   0          5s
frontend-72hhc   1/1     Running   0          9m24s
frontend-9z75w   1/1     Running   0          5s
frontend-bzqwg   1/1     Running   0          9m24s
frontend-zzjgd   1/1     Running   0          9m24s
```
Notice how the age of the 2 newer pods is much lower than the pre-existing pods.

Try scaling in to 4 pods by editing the manifest and applying it again. You should see one of your pods disappear.

### Replacing failed pods using a ReplicaSet

A ReplicaSet will ensure that the desired number of pods is kept running. To see this, you can simulate a pod failure by deleting a pod.
Pick any of the pods created by the ReplicaSet and delete it. For example (please replace pod name as required):
```
kubectl delete pod frontend-bzqwg
```
Check your pods again and you will see that the ReplicaSet has created a new Pod to replace the one you just deleted (look for a pod with a low Age).




## Other things to try

Check out the [kubectl cheat sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-autocomplete)
