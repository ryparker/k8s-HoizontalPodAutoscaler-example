# Experimenting with K8s HorizontalPodAutoscaler

Following the Kubernetes.io's [HorizontalPodAutoscaler Walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)

[HPA docs](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

To resolve the `metrics-server error because it doesn’t contain any IP SANs` error in Minikube I followed this [blog post](https://thospfuller.com/2020/11/29/easy-kubernetes-metrics-server-install-in-minikube-in-five-steps/) (related to this [GitHub issue](https://github.com/kubernetes-sigs/metrics-server/issues/196)).

## :rocket: Quick start

1. Start Minikube with 2 nodes

```sh
minikube start --nodes 2
```

2. Apply the [metrics server](https://github.com/kubernetes-sigs/metrics-server#deployment)

```sh
kubectl apply -f src/metrics-server.yaml
```

3. Apply the PHP apache application

```sh
kubectl apply -f src/php-apache.yaml
```

4. Apply the HorizontalPodAutoscaler

```sh
kubectl apply -f src/hpa.yaml
```

5. [Open new terminal] Increase the load on the PHP apache application

```sh
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

6. [Open new terminal] Watch the HorizontalPodAutoscaler scale up

```sh
kubectl get hpa -w
```

7. Stop the load generator (terminal used in step 5)

```sh
<Ctrl> + C
```

8. Watch the HorizontalPodAutoscaler scale down (terminal used in step 6)

## Useful commands

View the HPA status

```sh
kubectl describe hpa php-apache
```

## Deploy Kubernetes Dashboard

1. Apply the service

```sh
kubectl apply -f src/dashboard/service.yaml
```

2. Apply the admin-user

```sh
kubectl apply -f src/dashboard/admin-user.yaml
```

3. Get the token

```sh
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

4. Start the proxy

```sh
kubectl proxy
```

5. Open the dashboard

http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/


## How HPA works

- The HPA controller periodically queries the metrics API for the current CPU utilization of the pods in the deployment. - Default 15 seconds

- The algorithm for scaling is:

  desiredReplicas = ceil[currentReplicas * ( currentMetricValue / desiredMetricValue )]

  _The control plane skips any scaling action if the ratio is sufficiently close to 1.0 (within a globally-configurable tolerance, 0.1 by default)._

  _All Pods with a deletion timestamp set (objects with a deletion timestamp are in the process of being shut down / removed) are ignored, and all failed Pods are discarded._

## Defining metrics on resources

`targetAverageValue` · `targetAverageUtilization`

`averageUtilization` - Utilization is the ratio between the current usage of resource to the requested resources of the pod.

## HPA flags

- `--horizontal-pod-autoscaler-initial-readiness-delay` - default is 30 seconds - determining whether to set aside certain CPU metrics for the first 30 seconds of the pod's life.

- `--horizontal-pod-autoscaler-initial-readiness-delay` - default is 5 minutes - Once a pod has become ready, it considers any transition to ready to be the first if it occurred within a this configurable time since it started.

- `--horizontal-pod-autoscaler-downscale-stabilization` - default is 5 minutes - The period since the last downscale, before another downscale can be performed in response to a new scale event.

## Requirements

- API objects should follow the same constraints as subdomain names.
  - contain no more than 253 characters
  - contain only lowercase alphanumeric characters, '-' or '.'
  - start with an alphanumeric character
  - end with an alphanumeric character

## Changing HPA's target resource names

This can be done in the following way:
1. Add new name to the HPA target config.
2. Change the resource name
3. Remove the old name from the HPA target config.


## Autoscaling v2

- Supports custom metrics
- Specify multiple metrics to scale on.
- Allows setting a `behavior` for scaling up and down.

## Support for metrics APIs

By default, the HorizontalPodAutoscaler controller retrieves metrics from a series of APIs. In order for it to access these APIs, cluster administrators must ensure that:

- The API aggregation layer is enabled.

- The corresponding APIs are registered:

    - For resource metrics, this is the metrics.k8s.io API, generally provided by metrics-server. It can be launched as a cluster add-on.

    - For custom metrics, this is the custom.metrics.k8s.io API. It's provided by "adapter" API servers provided by metrics solution vendors. Check with your metrics pipeline to see if there is a Kubernetes metrics adapter available.

    - For external metrics, this is the external.metrics.k8s.io API. It may be provided by the custom metrics adapters provided above.


## Possible APIs

We will need an API to create the following:

- `HorizontalPodAutoscaler` resource - The HPA object
- `Metric` Enum - The metric to scale on
- `Scaling Policy` construct - The scaling policy object (used in autoscale/v2's `behavior` field)
- Possibly add a `maintenanceMode` option to `Pod`/`Container` resources (to prevent scaling on them). This would be useful for pods that are used for maintenance tasks (e.g. database migrations). See [Implicit maintenance-mode deactivation docs](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#implicit-maintenance-mode-deactivation)
-
