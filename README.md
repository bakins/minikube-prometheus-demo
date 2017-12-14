# Minikube and Prometheus Demo #

This is a quick demo of using
[minikube](https://github.com/kubernetes/minikube) to test
[Prometheus](https://prometheus.io/).  This is meant to familiarize
people with working with minikube, kubectl, prometheus, and grafana.

## Prerequisites ##

Note: this has been developed and tested on OS X. Others should be
similar.

* install [minikube](https://github.com/kubernetes/minikube) - on OS X
  it is highly recommended to use the [hyperkit driver](https://github.com/kubernetes/minikube/blob/master/docs/drivers.md#hyperkit-driver).
* install [kubectl]() - I install using [homebrew](http://brew.sh/index.html), but you can just
  grab the
  [binary](https://coreos.com/kubernetes/docs/latest/configure-kubectl.html)
  as well.
* Clone this repository to you machine.

## Bootstrap ##

Install the prerequisites.

If using the hyperkit driver for minikube, you should be able to create
and start a single node, local Kubernetes cluster by running:
`minikube start --vm-driver=hyperkit`.  If not using the hyperkit driver,
just run `minikube start`. See the [minikube docs](https://github.com/kubernetes/minikube/blob/master/docs/drivers.md) for more information.

You can check that the node is up and running by running: `minikube
status`. You should see something like:
```
minikube: Running
cluster: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.64.48
```

If you want to stop the cluster, run `minikube stop`. To start it, run
`minikube start`. Note: you only need to pass the driver argument the
first time you create the cluster (or if you destroy and recreate it).

You can run `minikube dashboard` and a browser window should open with
the [Kubernetes Dashboard](https://github.com/kubernetes/dashboard)
running. You may to click around to familiarize yourself with it.


Run `kubectl cluster-info` and you should see something like:
```
kubernetes master is running at https://192.168.64.2:8443
kubernetes-dashboard is running at https://192.168.64.2:8443/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard
```

To select the minikube local cluster for kubectl - useful if you are
using multiple clusters - then run `kubectl config use-context
minikube`

### Monitoring Namespace ###
We are going to install the monitoring components into a "monitoring"
namespace.  While this is not necessary, it does show "best practices"
in organizing applications by namespace rather than deploying
everything into the default namespace.


First, create the monitoring namespace: `kubectl apply -f monitoring-namespace.yaml`.

You can now list the namespaces by running `kubectl get namespaces`
and you should see something similar to:

```
NAME          STATUS    AGE
default       Active    6d
kube-system   Active    6d
monitoring    Active    3d
```

## Deploying Prometheus and Grafana ##

Let's step through deploying Prometheus.  The configuration we will
use is based on a
[blog post by CoreOS](https://coreos.com/blog/monitoring-kubernetes-with-prometheus.html)
and the
[example configuration](https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus-kubernetes.yml)
included with Prometheus.

### Prometheus Configuration ###
Prometheus will get its configuration from a
[Kubernetes ConfigMap](http://kubernetes.io/docs/user-guide/configmap/).
This allows us to update the configuration separate from the image.
Note: there is a large debate about whether this is a "good" approach
or not, but for demo purposes this is fine.

Look at [prometheus-config.yaml](./prometheus-config.yaml). The
relevant part is in `data/prometheus.yml`.  This is just a [Prometheus
configuration](https://prometheus.io/docs/operating/configuration/)
inlined into the Kubernetes manifest. Note that we are using the
in-cluster
[Kubernetes service account](http://kubernetes.io/docs/user-guide/service-accounts/)
to access the Kubernetes API.

To deploy this to the cluster run `kubectl apply -f
prometheus-config.yaml`.  You can view this by running `kubectl get
configmap --namespace=monitoring prometheus-config -o yaml`. You can
also see this in the Kubernetes Dashboard.


### Prometheus Pod ###
We will use a single Prometheus
[pod](http://kubernetes.io/docs/user-guide/pods/) for this demo.  Take
a look at [prometheus-deployment.yaml](./prometheus-deployment.yaml).
This is a [Kubernetes Deployment](http://kubernetes.io/docs/user-guide/deployments/) that describes the image to use for
the pod, resources, etc.  Note:

* In the metadata section, we give the pod a label with a key of
`name` and a value of `prometheus`. This will come in handy later.
* In annotations, we set a couple of key/value pairs that will
actually allow Prometheus to autodiscover and scrape itself.
* We are using an
  [emptyDir volume](http://kubernetes.io/docs/user-guide/volumes/#emptydir)
  for the Prometheus data.  This is basically a temporary directory
  that will get erased on every restart of the container.  For a demo
  this is fine, but we'd do something more persistent for other use
  cases.

Deploy the deployment by running `kubectl apply -f
prometheus-deployment.yaml`.  You can see this by running `kubectl get
deployments --namespace=monitoring`.

### Prometheus Service ###

Now that we have Prometheus deployed, we actually want to get to the
UI.  To do this, we will expose it using a
[Kubernetes Service](http://kubernetes.io/docs/user-guide/services/).

In [prometheus-service.yaml](./prometheus-service.yaml), there are a
few things to note:

* The label selector searches for pods that have been labeled with
`name: prometheus` as we labeled our pod in the deployment.
* We are exposing port 9090 of the running pods.
* We are using a "NodePort."  This means that Kubernetes will open a
port on each node in our cluster. You can query the API to get this
port.

Create the service by running `kubectl apply -f
prometheus-service.yaml`.  You can then view it by running `kubectl
get services --namespace=monitoring prometheus -o yaml`.

One thing to note is that you will see something like `nodePort:
30827` in the output.  We could access the service on that port on any
node in the cluster.  Minikube comes with a helper to do just that,
just run `minikube service --namespace=monitoring prometheus` and it
will open a browser window accessing the service.

From the Prometheus console, you can explore the metrics is it
collecting and do some basic graphing.  You can also view the
configuration and the targets. Click Status->Targets and you should
see the Kubernetes cluster and nodes.  You should also see that
Prometheus discovered itself under `kubernetes-pods`

### Deploying Grafana ###

You can deploy [grafana](http://grafana.org/) by creating its deployment and service by
running `kubectl apply -f grafana-deployment.yaml` and `kubectl
apply -f grafana-service.yaml`. Feel free to explore via the kubectl
command line and/or the Dashboard.

Go to  grafana by running `minikube service --namespace=monitoring
grafana`.  Username is `admin` and password is also `admin`.

Let's add Prometheus as a datasource.
* Click on the icon in the upper
left of grafana and go to "Data Sources".
* Click "Add data
source".
* For name, just use "prometheus"
* Select "Prometheus" as the type
* For the URL, we will actual use [Kubernetes DNS service
  discovery](http://kubernetes.io/docs/user-guide/services/#dns). So,
  just enter `http://prometheus:9090`. This means that grafana will
  lookup the `prometheus` service running in the same namespace as it
  on port 9090.

Create a New dashboard by clicking on the upper-left icon and
selecting Dashboard->New.  Click the green control and add a graph
panel.  Under metrics, select "prometheus" as the datasource. For the
query, use `sum(container_memory_usage_bytes) by (pod_name)`.  Click
save. This graphs the memory used per pod.

### Prometheus Node Explorer ###

We can also use Prometheus to collect metrics of the nodes
themselves.  We use the
[node exporter](https://github.com/prometheus/node_exporter) for
this.  We can also use Kubernetes to deploy this to every node.  We
will use a
[Kubernetes DaemonSet](http://kubernetes.io/docs/admin/daemons/) to do
this.

In [node-exporter-daemonset.yml](./node-exporter-daemonset.yml) you
will see that it looks similar to the deployment we did earlier.
Notice that we run this in privileged mode (`privileged: true`) as it
needs access to various information about the node to perform
monitoring.  Also notice that we are mounting in a few node directories
to monitor various things.

Run `kubectl apply -f node-exporter-daemonset.yml` to create the
daemon set.  This will run an instance of this on every node. In
minikube, there is only one node, but this concept scales to thousands
of nodes.

You can verify that it is running by using the command line or the
dash board.

After a minute or so, Prometheus will discover the node itself and
begin collecting metrics from it.  To create a dashboard in grafana
using node metrics, follow the same procedure as before but use
`node_load1` as the metric query.  This will be the one minute load
average of the nodes.

Note: in a "real" implementation, we would label the pods in an easily
queryable pattern.

To cleanup, you can delete the entire monitoring namespace `kubectl delete namespace monitoring`