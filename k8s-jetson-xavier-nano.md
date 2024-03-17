# Install Kubernatice on Xavier NX, Jetson Nano for Ubuntu 20.04

 - Jetson Mate carrier board is used for installation https://wiki.seeedstudio.com/Jetson-Mate/
 - Cluster contains one Jetson Xavier NX and Jetson Nano modules both using Ubuntu 20.04.

## Prepare Jetson Xavier NX
 - Use following guide https://wiki.seeedstudio.com/Jetson-Mate/ to install latest jetpack on Jetson Xavier NX module. 

> [!NOTE]
> If USB is used, by default nvidia SDK manager will use only 14GB of disk space.
> To use all space apply the following hack:
> https://forums.developer.nvidia.com/t/space-error-when-flashing-and-installing-sdk-on-nvme-ssd-in-nvidia-jetson-xavier-nx/275523/3

## Prepare Jetson Nano

### Basic installation

Install latest JetPack for Jetson Nano using standard guilde in case if you use Jetson Nano Developer Kit. https://developer.nvidia.com/embedded/learn/get-started-jetson-nano-devkit#intro

> [!NOTE]
> If USB is used, perform following steps to be able to boot from usb:
> https://jetsonhacks.com/2021/03/10/jetson-nano-boot-from-usb/

### Upgrade Jetson Nano to Ubuntu 20.04
Use following guide https://qengineering.eu/install-ubuntu-20.04-on-jetson-nano.html

## Install kubernatice cluster 

### Install on nodes Jetson Xavier NX / Jetson Nano

Use following guide for basic K8s setup https://wiki.seeedstudio.com/Jetson-Mate/#build-a-kubernetes-cluster-with-jetson-mate.

> [!NOTE]
> Execute command **sudo swapoff -a** each time when reboot

By default K8s uses containerd runtime for Xavier NX based node. In case if containerd is not installed uses following commands

```
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common

# Install containerd
sudo apt-get install -y containerd

# Configure containerd and start the service
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```
Check GPU availability for K8s nodes. At least 1 GPU showd be available.
```
kubectl get nodes -o=custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\\.com/gpu
``` 

Check that nvidia toolkit is installed
```
nvidia-ctk --version
```
Install toolkit if not istalled: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

Configure containerd to use nvidia runtime using following guide: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/22.9.2/installation.html#bare-metal-passthrough-with-pre-installed-drivers-and-nvidia-container-toolkit. Update containerd to use nvidia as the default runtime and add nvidia runtime configuration. This can be done by adding below config to /etc/containerd/config.toml and restarting containerd service.
```
version = 2
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "nvidia"

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
            BinaryName = "/usr/bin/nvidia-container-runtime"
```
Restart containerd service
```
sudo systemctl restart containerd
```

### Additional steps for Xavier NX
Fix root and cli paths in cat /etc/nvidia-container-runtime/config.toml file
```
sudo nano /etc/nvidia-container-runtime/config.toml
```
Make sure this lines are commented
```
....
#root = "/run/nvidia/driver"
#path = "/usr/bin/nvidia-container-cli"
....
```
Restart containerd service
```
sudo systemctl restart containerd
```

### Testing steps

The testing steps are common for Xavier NX and Jetson Nano but
- Jetson Nano image: "nvcr.io/nvidia/l4t-base:r32.4.2"
- Jetson Xavier image: "nvcr.io/nvidia/l4t-jetpack:r35.4.1"

Then you can validate installation with running test image. Make sure pod is scheduled on node with appropriate jetpack installed.
```
kubectl run -it --image <image-name> test

# Then inside container execute ./deviceQuery
apt-get update && apt-get install -y --no-install-recommends make g++
cp -r /usr/local/cuda/samples /tmp
cd /tmp/samples/1_Utilities/deviceQuery
make
./deviceQuery
```
Another approach is to use yaml files from the repo. Please make sure that node affinity in the files setup correctly. 

For Jetson Nano node:
```
kubectl apply -f cuda-sample.yaml
``` 


