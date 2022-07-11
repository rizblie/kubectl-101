# kubectl-101

## Basics

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
**Note**: If you are using Amazon EKS, you will only see details for your worker nodes. If you are running standard (self-managed) Kubernetes then you will see both worker nodes and control plane nodes.

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

You can see that all these pods are running in the namespace `kube-system`. In a later section, you will see how you can use namespaces to segment your cluster for management purposes.

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
to connect to the NGINX server running in the pod. Verfify that you can see the HTML output of the root page.

You can also view the container's log to verify that your HTTP request was received. To view the pod's container logs, use:
```
kubectl logs nginx
```
Run the `curl` command aboce a few more times to generate some more HTTP requests, and then check the logs to verify that you can see the requests.

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
Once you have tried this, you can delete this pod in the usual way with `kubectl delete pod nginx`.

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

Let's reproduce the previous example using a manifest file. Create a file `nginx.yaml` with the following contents:
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

This will create a namespace and pod. Verify this using `kubectl get pods -n my-namespace`.

Now try applying the manifest again, and see what happens.

To delete the pod and namespace, use:
```
kubectl delete -f nginx.yaml
```

## ReplicaSets

When running services at scale, it is often necessary to run multiple copies of a pod to enable horizontal scaling. Further, mechanisms are needed
to ensure that the number of pods can be adjusted as needed.

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

## Exposing pods as services

All pods get their own internal IP address, but if you want to use a group of pods (as might be created by a ReplicaSet) to work as one service then
you need a single service endpoint along with some mechanism for distributing load across the pods that make up the service.

Kubernetes offers a number of ways to expose a service:
- ClusterIP: Kubernetes assigns the service an innternal load-balanced IP address. Just like pods, the The IP address that is used for the ClusterIP is not routable outside of the cluster.
- NodePort: The service gets a ClusterIP address, and is additionally exposed on a port (in the range 30000-32767 by default) on every node in the cluster. Any node IP address can then be used to access the service.
- LoadBalancer: Creates a NodePort service and then exposes the service node ports using a load balancer, such as an AWS NLB.

### Creating a ClusterIP service

To expose our NGINX pods as a ClusterIP service, create the manifest file `frontend-svc.yaml`:
```
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: ClusterIP
  selector:
    tier: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
Note how the `selector` is used to include all pods that have the label `tier: frontend` as part of the service.

Apply the manifest in the usual way. Then, verify that the service has been created, using:
```
kubectl get service frontend-service
```
Note the assigned Cluster IP address.

To test the service, you can use port forwarding as done previously, but this time to a service instead of a pod. Run
```
kubectl port-forward svc/frontend-service 8080:80
```
and then use a different terminal to run `curl 127.0.0.1:8080`. Don't forget to Ctrl-C the forwarding process when done!

**Exercise:** Can you figure out which of the pods that sit behind the service actually handled your test request?

Alternatively you can test by running an interactive client pod inside the cluster.
You can use the `busybox` image to run an interactive pod for this purpose as follows:
```
kubectl run busybox -it --image=public.ecr.aws/docker/library/busybox --rm --restart=Never
```
`busybox` is a minimalist version of Linux that is convenient for running tests inside a cluster. From the busybox command prompt type:
```
wget -O - <cluster-ip>
```
to verify that you can retrieve a page from NGINX.
Note: Use the clusterIP address that you observed when you ran `kubectl get service frontend-service` earlier.

Once done type `exit` to leave the busybox pod. Use of the `--rm` flag in the run command automaticaly deletes the pod upon completion.

### Creating a NodePort service

Let's change the service type to NodePort. Edit `frontend-svc.yaml` and change the service `Type` to `NodePort` so that the file now looks like this:
```
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: NodePort
  selector:
    tier: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Use
```
kubectl apply -f frontend-svc.yaml
```
to apply the change.

Run
```
kubectl get service frontend-service
```
You shuold see:
```
NAME               TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
frontend-service   NodePort   10.100.92.62   <none>        80:30142/TCP   37m
```
You can see that the service still has a cluster IP, but has now also been assigned a node port in the range 30000-32767 (in this case 30142).

Again you can use busybox to test this. Let us use a slightly different approach. Run a busybox server using:
```
kubectl run busybox --image=public.ecr.aws/docker/library/busybox -- sleep 3600
```
You can now run commandson your busybox server using the `exec` command. For example:
```
kubectl exec busybox -- date
```
runs the `date` command on busybox and returns.

To test the NodePort service, use `kubectl get nodes` to identify a node IP address, then run
```
kubectl exec busybox -- wget -O - <node-ip>:<node-port>/
```
where `<node-ip>` is a node IP address, and `<node-port>` is the service node port identified above.

Once you have finished testing you can kill your busybox server pod in the usual way.

### Creating a LoadBalancer service
Let's change the service type to LoadBalancer. Edit `frontend-svc.yaml` and change the service `Type` to `LoadBalancer` so that the file now looks like this:
```
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: LoadBalancer
  selector:
    tier: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
Apply the manifest in the usual way, then run:
```
kubectl get service frontend-service
```
You should see the `EXTERNAL-IP` value populated with the URI for an AWS load balancer. Copy and paste this into a browser to test your service. Note that you will need to wait for a few minutes for DNS to propagate.
  
### Clean up resources
  
Once tested you can delete the service and replica set using:
```
kubectl delete -f frontend-svc.yaml
kubectl delete -f frontend-rs.yaml
```

## Further reading, and other things to try
  
Browse the kubernetes [documentation](https://kubernetes.io/docs/home/). Specifically, have a look at:
- [Pods lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [kubectl cheat sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-autocomplete)
