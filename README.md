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
kubectl run nginx --image=public.ecr.aws/nginx/nginx:1-alpine-perl
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
kubectl run nginx --image=nginx -n my-namespace
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

## Using manifest files



### Other things to try

Check out the [kubectl cheat sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-autocomplete)
