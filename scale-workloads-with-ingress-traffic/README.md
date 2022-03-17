# Scale workloads with ingress traffic

Main article here: https://blog.sighup.io/scale-workloads-with-ingress-traffic/

## Prerequisites

- `kustomize` 3.5.3
- `kubectl` 1.18
- `hey` https://github.com/rakyll/hey
- `jq` https://github.com/stedolan/jq
- `furyctl` https://github.com/sighupio/furyctl

## How to

First of all, you need a cluster up & running, a simple minikube node will suffice.

Bring up your node with:

```bash
minikube start --vm-driver=virtualbox --kubernetes-version v1.18.10 --memory=6000m --cpus=3
```

We will then use furyctl to download all the component we will install on the cluster (all the manifests are already downloaded in this directory, you can skip this step):

```bash
# if using furyctl version <1.0.0
furyctl vendor -H
# if using furyctl version >=1.0.0 
furyctl distribution download -H
```

Install all the Fury components in your minikube node with:

```bash
kustomize build . | kubectl apply -f -
```

Run the command above two times, because we are creating some CRDs and APIserver is not able to serve them immediately.

After this part, we have a minikube node up&running with Nginx Ingress Controller, Prometheus and Prometheus Adapter.

Check with:

```bash
kubectl get pods -n monitoring
kubectl get pods -n ingress-nginx
```

Let's go ahead deploying our app, all the code we need is in the subfolder `demo-prometheus-adapter-app`

```bash
kustomize build demo-prometheus-adapter-app | kubectl apply -f -
```

Let's wait that everything is up&running in the new namespace `autoscale-all-the-things` :

```bash
kubectl get pods -n autoscale-all-the-things
```

Now we are ready for some testing!

Get your minikube ip with: `minikube ip`

We need this ip to put on out `/etc/hosts` file the dummy domain we are using, pointing to the minikube node, for example:

```txt
192.168.99.166 apache2.example.com
```

Test the connection to the ingress controller via https on port 31443: https://apache2.example.com:31443

Run some load testing:

```bash
hey -n 20000 -q 120 -c 1 https://apache2.example.com:31443/
```

Check if metrics are populated:

```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/autoscale-all-the-things/ingress/apache2/nginx_ingress_controller_requests_rate" | jq .
```

You should see something like this:

```json
{
  "kind": "MetricValueList",
  "apiVersion": "custom.metrics.k8s.io/v1beta1",
  "metadata": {
    "selfLink": "/apis/custom.metrics.k8s.io/v1beta1/namespaces/autoscale-all-the-things/ingress/apache2/nginx_ingress_controller_requests_rate"
  },
  "items": [
    {
      "describedObject": {
        "kind": "Ingress",
        "namespace": "autoscale-all-the-things",
        "name": "apache2",
        "apiVersion": "extensions/v1beta1"
      },
      "metricName": "nginx_ingress_controller_requests_rate",
      "timestamp": "2021-05-26T08:32:42Z",
      "value": "19090m",
      "selector": null
    }
  ]
}
```

You should see, describing HPA object, that the deployment has scaled up to 2 replicas!

```bash
-> kubectl describe hpa apache2 -n autoscale-all-the-things                                                                                                 
Name:                                                                                  apache2
Namespace:                                                                             autoscale-all-the-things
Labels:                                                                                <none>
Annotations:                                                                           <none>
CreationTimestamp:                                                                     Wed, 26 May 2021 10:25:56 +0200
Reference:                                                                             Deployment/apache2
Metrics:                                                                               ( current / target )
  "nginx_ingress_controller_requests_rate" on Ingress/apache2 (target average value):  119880m / 100
Min replicas:                                                                          1
Max replicas:                                                                          100
Deployment pods:                                                                       1 current / 2 desired
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    SucceededRescale    the HPA controller was able to update the target scale to 2
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from external metric nginx_ingress_controller_requests_rate(nil)
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
```
