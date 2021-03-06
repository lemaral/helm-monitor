Helm monitor example
====================

> In this example, we run the Prometheus operator and a GoLang application in
Minikube, upgrade then monitor for HTTP failure. If the amount of 5xx reach a
certain limit, then the application get automatically rolled back to its
previous state.


## Prepare

Build 2 versions and release the first application to Minikube:

```
$ eval $(minikube docker-env)
$ docker build --build-arg "VERSION=1.0.0" -t app:1.0.0 app
$ docker build --build-arg "VERSION=2.0.0" -t app:2.0.0 app
$ helm upgrade -i my-release ./app/charts --set image.tag=1.0.0
```

Access the application:

```
$ minikube service my-release-app
```

## Prometheus

### Setup

Install Prometheus using Prometheus operator:

```
$ helm install --name prometheus-operator coreos/prometheus-operator
$ kubectl apply -f ./prometheus.yaml
```

Access Prometheus:

```
$ minikube service prometheus
```

### Upgrade and monitor

```
$ kubectl port-forward prometheus-prometheus-0 9090
$ helm upgrade -i my-release ./app/charts --set image.tag=2.0.0
$ helm monitor my-release 'rate(http_requests_total{code=~"^5.*$"}[1m]) > 0'
```

Simulate internal server failure:

```
$ export APP=$(minikube service my-release-app --url)
$ while sleep 0.1; do curl $APP/internal-error; done
```


## ElasticSearch

### Setup

Minikube support the EFK stack via addons, to enable it:

```
$ minikube addons enable efk
```

Access Kibana (it can take a while before being accessible):

```
$ minikube service kibana-logging -n kube-system
```

### Upgrade and monitor

```
$ kubectl port-forward -n kube-system $(kubectl get po -n kube-system -l k8s-app=elasticsearch-logging -o jsonpath="{.items[0].metadata.name}") 9200
$ helm upgrade -i my-release ./app/charts --set image.tag=2.0.0
```

Monitor using via query DSL:
```
$ helm monitor elasticsearch my-release ./elasticsearch-query.json
```

Or via Lucene query
```
$ helm monitor elasticsearch my-release "status:500 AND kubernetes.labels.app:app AND version:2.0.0"
```

Simulate internal server failure:
```
$ export APP=$(minikube service my-release-app --url)
$ while sleep 0.1; do curl $APP/internal-error; done
```
