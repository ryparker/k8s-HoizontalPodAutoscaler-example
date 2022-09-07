# Experimenting with K8s HorizontalPodAutoscaler

[HPA docs](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) · [HPA v1 API ref](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v1/) · [HPA v2 API ref](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v2/) · [kubectl autoscale commands](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#autoscale) · [Minikube docs](https://minikube.sigs.k8s.io/docs/handbook/config/)

Experimenting with K8s HorizontalPodAutoscaler by completing the recommended walkthroughs and logging notes in this README a long the way.

- [x] [HorizontalPodAutoscaler Walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/).
- [ ] Autoscale on multiple metrics and custom metrics [walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#autoscaling-on-multiple-metrics-and-custom-metrics)

## Table of contents

- [Experimenting with K8s HorizontalPodAutoscaler](#experimenting-with-k8s-horizontalpodautoscaler)
  - [Table of contents](#table-of-contents)
  - [:fuelpump: Minikube setup](#fuelpump-minikube-setup)
  - [:rocket: Quick start](#rocket-quick-start)
  - [Useful commands](#useful-commands)
  - [:newspaper: Deploy Kubernetes Dashboard](#newspaper-deploy-kubernetes-dashboard)
  - [:nut_and_bolt: How HPA works](#nut_and_bolt-how-hpa-works)
  - [:chart_with_upwards_trend: Defining metrics on resources](#chart_with_upwards_trend-defining-metrics-on-resources)
  - [:triangular_flag_on_post: HPA flags](#triangular_flag_on_post-hpa-flags)
  - [:white_check_mark: Requirements](#white_check_mark-requirements)
  - [:name_badge: Changing HPA's target resource names](#name_badge-changing-hpas-target-resource-names)
  - [:new: Autoscaling v2](#new-autoscaling-v2)
  - [:traffic_light: Pod conditions](#traffic_light-pod-conditions)
  - [:mag_right: Support for metrics APIs](#mag_right-support-for-metrics-apis)
  - [:key: Aggregation layer](#key-aggregation-layer)
  - [:balance_scale: Quantities](#balance_scale-quantities)
  - [:bulb: Possible APIs](#bulb-possible-apis)
  - [:arrow_up: Migrating to HPA](#arrow_up-migrating-to-hpa)

## :fuelpump: Minikube setup

To resolve the `metrics-server error because it doesn’t contain any IP SANs` error in Minikube (related to this [GitHub issue](https://github.com/kubernetes-sigs/metrics-server/issues/196)), I followed this [blog post](https://thospfuller.com/2020/11/29/easy-kubernetes-metrics-server-install-in-minikube-in-five-steps/).

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

## :newspaper: Deploy Kubernetes Dashboard

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

    [http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/]()


## :nut_and_bolt: How HPA works

- The HPA controller periodically queries the metrics API for the current CPU utilization of the pods in the deployment. - Default 15 seconds

- The algorithm for scaling is:

  desiredReplicas = ceil[currentReplicas * ( currentMetricValue / desiredMetricValue )]

  _The control plane skips any scaling action if the ratio is sufficiently close to 1.0 (within a globally-configurable tolerance, 0.1 by default)._

  _All Pods with a deletion timestamp set (objects with a deletion timestamp are in the process of being shut down / removed) are ignored, and all failed Pods are discarded._

## :chart_with_upwards_trend: Defining metrics on resources

`targetAverageValue` · `targetAverageUtilization`

`averageUtilization` - Utilization is the ratio between the current usage of resource to the requested resources of the pod.

## :triangular_flag_on_post: HPA flags

- `--horizontal-pod-autoscaler-initial-readiness-delay` - default is 30 seconds - determining whether to set aside certain CPU metrics for the first 30 seconds of the pod's life.

- `--horizontal-pod-autoscaler-initial-readiness-delay` - default is 5 minutes - Once a pod has become ready, it considers any transition to ready to be the first if it occurred within a this configurable time since it started.

- `--horizontal-pod-autoscaler-downscale-stabilization` - default is 5 minutes - The period since the last downscale, before another downscale can be performed in response to a new scale event.

## :white_check_mark: Requirements

- API objects should follow the same constraints as subdomain names.
  - contain no more than 253 characters
  - contain only lowercase alphanumeric characters, '-' or '.'
  - start with an alphanumeric character
  - end with an alphanumeric character

## :name_badge: Changing HPA's target resource names

This can be done in the following way:

1. Add new name to the HPA target config.
2. Change the resource name
3. Remove the old name from the HPA target config.

## :new: Autoscaling v2

- Supports custom metrics
- Specify multiple metrics to scale on.
- Allows setting a `behavior` for scaling up and down.
- See status conditions via `kubectl describe hpa <name>` [docs](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#autoscaling-on-metrics-not-related-to-kubernetes-objects)
  - `AbleToScale` - Indicates whether or not the HPA is able to fetch and update scales, as well as whether or not any backoff-related conditions would prevent scaling.
  - `ScalingActive` - Indicates whether or not the HPA is enabled (i.e. the replica count of the target is not zero) and is able to calculate desired scales.
  - `ScalingLimited` - Indicates that the desired scale was capped by the maximum or minimum of the HorizontalPodAutoscaler

## :traffic_light: Pod conditions

Useful to know since HPA scales depending on pod readiness. _[Docs](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-conditions)_

`PodScheduled` - the Pod has been scheduled to a node.
`PodHasNetwork` - (alpha feature; must be enabled explicitly) the Pod sandbox has been successfully created and networking configured.
`ContainersReady` - all containers in the Pod are ready.
`Initialized` - all init containers have completed successfully.
`Ready` - the Pod is able to serve requests and should be added to the load balancing pools of all matching Services.

## :mag_right: Support for metrics APIs

By default, the HorizontalPodAutoscaler controller retrieves metrics from a series of APIs. In order for it to access these APIs, cluster administrators must ensure that:

- The API aggregation layer is enabled.

- The corresponding APIs are registered:
  - For resource metrics, this is the metrics.k8s.io API, generally provided by [metrics-server](https://github.com/kubernetes-sigs/metrics-server#deployment). It can be launched as a cluster add-on.
  - For custom metrics, this is the custom.metrics.k8s.io API. It's provided by "adapter" API servers provided by metrics solution vendors. Check with your metrics pipeline to see if there is a Kubernetes metrics adapter available. [See boilerplate to get started](https://github.com/kubernetes-sigs/custom-metrics-apiserver)
  - For external metrics, this is the external.metrics.k8s.io API. It may be provided by the custom metrics adapters provided above.

## :key: Aggregation layer

[Docs](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/)

Configuring the aggregation layer allows the Kubernetes apiserver to be extended with additional APIs, which are not part of the core Kubernetes APIs.

## :balance_scale: Quantities

All metrics in the HorizontalPodAutoscaler and metrics APIs are specified using a special whole-number notation known in Kubernetes as a quantity. For example, the quantity 10500m would be written as 10.5 in decimal notation. The metrics APIs will return whole numbers without a suffix when possible, and will generally return quantities in milli-units otherwise. This means you might see your metric value fluctuate between 1 and 1500m, or 1 and 1.5 when written in decimal notation.

## :bulb: Possible APIs

We will need an API to create the following:

- `HorizontalPodAutoscaler` resource - The HPA object
- `Metric` Enum - The metric to scale on
- `Scaling Policy` construct - The scaling policy object (used in autoscale/v2's `behavior` field)
- Possibly add a `maintenanceMode` option to `Pod`/`Container` resources (to prevent scaling on them). This would be useful for pods that are used for maintenance tasks (e.g. database migrations). See [Implicit maintenance-mode deactivation docs](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#implicit-maintenance-mode-deactivation)
-

## :arrow_up: Migrating to HPA

[Migrating Deployments and StatefulSets to horizontal autoscaling docs](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#migrating-deployments-and-statefulsets-to-horizontal-autoscaling) - When an HPA is enabled, it is recommended that the value of spec.replicas of the Deployment and / or StatefulSet be removed from their manifest(s). If this isn't done, any time a change to that object is applied, for example via kubectl apply -f deployment.yaml, this will instruct Kubernetes to scale the current number of Pods to the value of the spec.replicas key. This may not be desired and could be troublesome when an HPA is active.
