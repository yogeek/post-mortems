# post-mortems
Quick notes about issues encountered in production

- `draino`: configure to drain statefulset to avoid issues on nodes + PR to add option do documentation https://github.com/planetlabs/draino/pull/122

- `cluster-autoscaler` : threshold 50% node allocation => traefik daemonset request were too high to the CA never deleted node. When we updated DS request, we needed 1 less node per cluster + PR to add options to documentation https://github.com/kubernetes/autoscaler/pull/4846

- `kube-proxy monitoring` components were added in latest kube-prometheus release but not backported to the previous ones (in particular the one that we must use because of K8S version compatibility matrix) => PR to backport this PodMonitor to fix the kube-proxy monitoring https://github.com/prometheus-operator/kube-prometheus/pull/1715

- `containerd + kubeadm` : kubeadm config image pull was used to pre-pull images in AMI, but when switching from docker to containerd, this command was not working anymore because it does not interact with containerd => we needed to install nerdctl to have a "docker-similar" UX (but without docker) and use `kubeadm config image list` to loop over k8s images and for each do a pull with nerdctl
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

- Istio sidecar
 * microdemo pods scaled up to max replicas regularly and then scaled down
 * HPA triggered the scaling because of resource consumption, even if our app had no traffic
 * No default Istio Sidecar configured => istio sent the whole cluster config to all istio proxies, which made the resource consumption of the pod reach the HPA target
=> Always add a default Sidecar in istio-system
