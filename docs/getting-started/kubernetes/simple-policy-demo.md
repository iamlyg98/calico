---
Title: Simple Policy Demo
---
# Kubernetes NetworkPolicy: A simple example
This guide provides a simple way to try out Kubernetes NetworkPolicy with Calico.  It requires a Kubernetes cluster configured with Calico networking, and expects that you have `kubectl` configured to interact with the cluster.

You can quickly and easily deploy such a cluster by following one of the [getting started guides](../README.md#getting-started-guides)

### Configure Namespaces
This guide will deploy pods in a Kubernetes Namespaces.  Let's create the `Namespace` object for this guide.

```
kubectl create ns policy-demo
```

### Create demo Pods
We'll use Kubernetes `Deployment` objects to easily create pods in the `Namespace`.

1) Create some nginx pods in the `policy-demo` Namespace, and expose them through a Service.

```
# Run the Pods.
kubectl run --namespace=policy-demo nginx --replicas=2 --image=nginx

# Create the Service.
kubectl expose --namespace=policy-demo deployment nginx --port=80
```

2) Ensure the nginx service is accessible.

```
# Run a Pod and try to access the `nginx` Service.
$ kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh
Waiting for pod policy-demo/access-472357175-y0m47 to be running, status is Pending, pod ready: false

Hit enter for command prompt

/ # wget -q nginx -O -
```

You should see a response from `nginx`.  Great! Our Service is accessible.  You can exit the Pod now.

### Enable isolation

Let's turn on isolation in our policy-demo Namespace.  Calico will then prevent connections to pods in this Namespace.

```
kubectl annotate ns policy-demo "net.beta.kubernetes.io/network-policy={\"ingress\":{\"isolation\":\"DefaultDeny\"}}"
```

This will prevent all access to the nginx Service.  We can see the effect by trying to access the Service again.

```
# Run a Pod and try to access the `nginx` Service.
$ kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh
Waiting for pod policy-demo/access-472357175-y0m47 to be running, status is Pending, pod ready: false

Hit enter for command prompt

/ # wget -q --timeout=5 nginx -O -
wget: download timed out
/ #
```

The request should time out after 5 seconds.  Be enabling isolation on the Namespace, we've prevented access to the Service.

### Allow Access using a NetworkPolicy

Now, let's enable access to the nginx Service using a NetworkPolicy.  This will allow incoming connections from our `access` Pod, but not
from anywhere else.

Create a file called `nginx-policy.yaml` with the following contents:

```yaml
kind: NetworkPolicy
apiVersion: extensions/v1beta1
metadata:
  name: access-nginx
  namespace: policy-demo
spec:
  podSelector:
    matchLabels:
      run: nginx
  ingress:
    - from:
      - podSelector:
          matchLabels:
            run: access
```
> Notice the NetworkPolicy allows traffic from Pods with the label `run: access` to Pods with the label
`run: nginx`.  These are the labels automatically added to Pods started via `kubectl run` based on the name of the `Deployment`.

Then use kubectl to deploy it:
```
kubectl create -f nginx-policy.yaml
```

We should now be able to access the Service from the `access` Pod.
```
# Run a Pod and try to access the `nginx` Service.
$ kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh
Waiting for pod policy-demo/access-472357175-y0m47 to be running, status is Pending, pod ready: false

Hit enter for command prompt

/ # wget -q --timeout=5 nginx -O -
```

However, we still cannot access the Service from a Pod without the label `run: access`:
```
# Run a Pod and try to access the `nginx` Service.
$ kubectl run --namespace=policy-demo cant-access --rm -ti --image busybox /bin/sh
Waiting for pod policy-demo/cant-access-472357175-y0m47 to be running, status is Pending, pod ready: false

Hit enter for command prompt

/ # wget -q --timeout=5 nginx -O -
wget: download timed out
/ #
```

You can clean up the demo by deleting the demo Namespace:

```
kubectl delete ns policy-demo
```

This was just a simple example of the Kubernetes NetworkPolicy API and how Calico can secure your Kubernetes cluster.  For more
information on network policy in Kubernetes, see the [Kubernetes user-guide](http://kubernetes.io/docs/user-guide/networkpolicies/).

For a slightly more detailed demonstration of Policy, check out the [stars demo](../stars-demo/README.md).

[![Analytics](https://calico-ga-beacon.appspot.com/UA-52125893-3/calico-containers/docs/cni/kubernetes/simple-policy-demo/README.md?pixel)](https://github.com/igrigorik/ga-beacon)