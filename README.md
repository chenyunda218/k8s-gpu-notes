# k8s-gpu-notes


## Nvidia driver installation
參考官網指引
https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```
```bash
sed -i -e '/experimental/ s/^#//g' /etc/apt/sources.list.d/nvidia-container-toolkit.list
```
```bash
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```
安裝nvidia容器運行時和驅動，這裏的驅動使用535版本，根據需要自行更改。
```bash
apt install -y nvidia-container-runtime cuda-drivers-fabricmanager-535 nvidia-headless-535-server
```
# 安裝kubernets
## k3s
k3s是一個開箱即用的k8s發行版，為降低部署的複雜度，我們選擇使用k3s。
## 安裝k3s
如果已經安裝請重啓k3s(`systemctl restart k3s`)。k3s會自動根據環境配置。
```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=<密碼> sh -s - server \
    --cluster-init
```
驗證k3s是否有自動配置nvidia的運行時。
```bash
grep nvidia /var/lib/rancher/k3s/agent/etc/containerd/config.toml
```
輸出如下信息
```bash
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes."nvidia"]
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes."nvidia".options]
  BinaryName = "/usr/bin/nvidia-container-runtime"
```
## Rancher
Rancher是一套k8s的GUI操作介面，為方便後續使用，建議安裝rancher。

## k8s GPU相關組件
參照https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html，使用helm安裝gpu-operator。
由於k3s的containerd是內置的，與本機的位置不同，因此需要在helm values中作出更改
```yaml
# ...
toolkit:
  enabled: true
  env:
    # 指定containerd socket的位置。
    - name: CONTAINERD_SOCKET
      value: /run/k3s/containerd/containerd.sock
# ...
```
```bash
kubectl create ns gpu-operator
```

```bash
helm upgrade -i nvdp nvdp/nvidia-device-plugin \
  --namespace nvidia-device-plugin \
  --set runtimeClassName=nvidia \
  --create-namespace 
```