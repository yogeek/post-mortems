# Post Mortems
Quick notes about issues encountered in production

## draino and statefulset

configure to drain statefulset to avoid issues on nodes + PR to add option do documentation https://github.com/planetlabs/draino/pull/122

## cluster-autoscaler and daemonsets

threshold 50% node allocation => traefik daemonset request were too high to the CA never deleted node. When we updated DS request, we needed 1 less node per cluster + PR to add options to documentation https://github.com/kubernetes/autoscaler/pull/4846

## kube-proxy monitoring

`kube-proxy monitoring` components were added in latest kube-prometheus release but not backported to the previous ones (in particular the one that we must use because of K8S version compatibility matrix) => PR to backport this PodMonitor to fix the kube-proxy monitoring https://github.com/prometheus-operator/kube-prometheus/pull/1715

## containerd + kubeadm

kubeadm config image pull was used to pre-pull images in AMI, but when switching from docker to containerd, this command was not working anymore because it does not interact with containerd => we needed to install nerdctl to have a "docker-similar" UX (but without docker) and use `kubeadm config image list` to loop over k8s images and for each do a pull with nerdctl
```
sandbox_image=""
for img in $(kubeadm config images list --config ./kubeadm-config.yaml); do
  echo "Pulling $img"
  sudo nerdctl --namespace k8s.io image pull "$img"
  if [[ $img == "${ECR_HOSTNAME}/pause:"* ]]
  then
    sandbox_image="$img"
  fi
done
if [[ $sandbox_image == "" ]]
then
  echo "[ERROR] pause image not found in 'kubeadm config images list --config ./kubeadm-config.yaml'"
  exit 1
fi
echo "Force pause image name in containerd to ${sandbox_image}"
sudo sed -i "/sandbox_image /s%=.*$%= \"${sandbox_image}\"%" /etc/containerd/config.toml
```

## Istio sidecar caused workloads to scale up

  * microdemo pods scaled up to max replicas regularly and then scaled down
  * HPA triggered the scaling because of resource consumption, even if our app had no traffic
  * No default Istio Sidecar configured so istio sent the whole cluster config to all istio proxies, which made the resource consumption of the pod reach the HPA target
  * conclusion : Always add a default Sidecar in istio-system

## AggregatedAPIErrors for  (name="v1beta1.metrics.k8s.io", prometheus="monitoring/k8s")
  * `kubectl get apiservice v1beta1.metrics.k8s.io -oyaml` shows that the prometheus-adapter is meant to serve this metrics API but is not responding
  * on this specific cluster, prometheus-operator stack is installed from the helm chart instead of the kube-prometheus-stack repo (jsonnet) and the promtheus-adapter is not included in the chart, it has its own chart.
  * metrics-server should be deleted if prometheus-adapter is deployed with "resource metrics API service" (https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus-adapter/README.md#horizontal-pod-autoscaler-metrics) : it provides the same functionality as metrics-server so we cannot install both on the same cluster (https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus-adapter/templates/resource-metrics-apiservice.yaml and https://github.com/kubernetes-sigs/metrics-server/blob/master/manifests/base/apiservice.yaml)

## KubeAPIErrorBudgetBurn alerts mystery

  * https://github.com/prometheus-operator/kube-prometheus/wiki/KubeAPIErrorBudgetBurn
  * grafana-operator has [an issue](https://github.com/grafana-operator/grafana-operator/issues/808) that caused the controller to update frequently the grafana deployment (several times per seconds) which should not be an issue on its own, but with kyverno installed and many policies looking for deployments to check and produce policyreportes, that led to multiple API servers calls and caused API Server timeouts because of kyverno long answers (validatingwebhooks are blocking by default) 

## Calico and NodeLocalCacheDNS

https://github.com/projectcalico/calico/issues/6910

## ArgoCD + PrometheusOperator

- kube-prometheus v0.9.0 does not include a `status` in the Prometheus Custom Resource so ArgoCD recently added [custom healthcheck](https://github.com/argoproj/argo-cd/tree/master/resource_customizations/monitoring.coreos.com/Prometheus) does not work correctly and the resource is always in "Progressing"
- https://blog.ediri.io/kube-prometheus-stack-and-argocd-23-how-to-remove-a-workaround
- https://www.arthurkoziel.com/fixing-argocd-crd-too-long-error/

## ALPN http1 -> http2 connection with Istio AWS NLB

client ---> Istio ingress SVC ---> backend pod

- problem : connection between client and istio is HTTP1 whereas connection between istio and pod is HTTP2 (because project DestinationRule has  `h2UpgradePolicy: UPGRADE`) and for large payload (download GBs files), there is a contention in frames because HTTP2 part is much faster than HTTP1

- Status : client proposes HTTP1 and HTTP2 to the server, the server chooses HTTP1 (`curl -vvv`) 

client --- [http1] ---> Istio ingress --- [http2] ---> pod

- In curl logs : `ALPN, server did not agree to a protocol`
- In fact, it is not istio SVC which isin charge of the SSL negociation, but the AWS NLB created by the K8S istio ingress SVC

client ---> AWS NLB ---> Istio ingress ---> backend pod

- And by default, AWS NLB listener has ALPN policy to `None` so it does not do any ALPN negociation
- By configuring ALPN Policy to `HTTP2Preferred`, the ALPN negociation can lead to an HTTP2 connection between client and NLB !

client --- [http2] ---> Istio ingress --- [http2] ---> pod

- WARNING : updating the SVC annotation to change a LoadBalancer option is not currently propagated to AWS LoadBalancer resource (https://github.com/kubernetes/kubernetes/issues/114111) so the modification has to be done in AWS console in addition to the source code (or the LoadBalancer service has to be deleted to be recreated with the good options)
