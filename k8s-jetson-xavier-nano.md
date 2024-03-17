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

> [!NOTE]
> For Jetson Nano during installation it can request override old nvidia-runtime configuration file /etc/nvidia-container-runtime/config.toml. Choose "Y" option and override this file with new one

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

You can update .yaml file to choose proper selectors for you cluster configuration:
```
.....
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-role.kubernetes.io/worker
            operator: In
            values:
            - worker
.....
```

For Jetson Nano node:
```
kubectl apply -f cuda-sample.yaml
``` 
or Xavier NX Node node:
```
kubectl apply -f cuda-sample.yaml
``` 
Check container logs when completed
```
kubectl logs nvidia-l4t-jetpack
```
Possible Output
```
mnt/data/deviceQuery Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

Detected 1 CUDA Capable device(s)

Device 0: "Xavier"
  CUDA Driver Version / Runtime Version          11.4 / 11.4
  CUDA Capability Major/Minor version number:    7.2
  Total amount of global memory:                 6854 MBytes (7186464768 bytes)
  (006) Multiprocessors, (064) CUDA Cores/MP:    384 CUDA Cores
  GPU Max Clock rate:                            1109 MHz (1.11 GHz)
  Memory Clock rate:                             510 Mhz
  Memory Bus Width:                              256-bit
  L2 Cache Size:                                 524288 bytes
  Maximum Texture Dimension Size (x,y,z)         1D=(131072), 2D=(131072, 65536), 3D=(16384, 16384, 16384)
  Maximum Layered 1D Texture Size, (num) layers  1D=(32768), 2048 layers
  Maximum Layered 2D Texture Size, (num) layers  2D=(32768, 32768), 2048 layers
  Total amount of constant memory:               65536 bytes
  Total amount of shared memory per block:       49152 bytes
  Total shared memory per multiprocessor:        98304 bytes
  Total number of registers available per block: 65536
  Warp size:                                     32
  Maximum number of threads per multiprocessor:  2048
  Maximum number of threads per block:           1024
  Max dimension size of a thread block (x,y,z): (1024, 1024, 64)
  Max dimension size of a grid size    (x,y,z): (2147483647, 65535, 65535)
  Maximum memory pitch:                          2147483647 bytes
  Texture alignment:                             512 bytes
  Concurrent copy and kernel execution:          Yes with 1 copy engine(s)
  Run time limit on kernels:                     No
  Integrated GPU sharing Host Memory:            Yes
  Support host page-locked memory mapping:       Yes
  Alignment requirement for Surfaces:            Yes
  Device has ECC support:                        Disabled
  Device supports Unified Addressing (UVA):      Yes
  Device supports Managed Memory:                Yes
  Device supports Compute Preemption:            Yes
  Supports Cooperative Kernel Launch:            Yes
  Supports MultiDevice Co-op Kernel Launch:      Yes
  Device PCI Domain ID / Bus ID / location ID:   0 / 0 / 0
  Compute Mode:
     < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >

deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 11.4, CUDA Runtime Version = 11.4, NumDevs = 1
Result = PASS
```


