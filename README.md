
# GPU Slicing for Cost Efficiency in Kubernetes

GPU slicing/sharing is a powerful technique that allows you to maximize the utilization of GPU resources in your Kubernetes clusters. By enabling multiple workloads to share a single GPU, you can significantly reduce costs while ensuring that your applications have the necessary compute power to perform efficiently. This document focuses on normal GPU slicing, without the complexities of Multi-Instance GPU (MIG) or time slicing.

We’ll also cover how to set up GPU slicing using Karpenter


## Installing the NVIDIA Device Plugin

To install the NVIDIA device plugin, run the following commands:

1. **Add the NVIDIA Helm repository**:
   
```shell
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin helm repo update
```
2. **Install the NVIDIA device plugin**:

```shell
helm upgrade -i nvdp nvdp/nvidia-device-plugin \
  --namespace nvidia-device-plugin \
  --create-namespace \
  --version 0.16.1
```
### What It Does

- Enables Kubernetes to manage GPU resources.
- Allows scheduling of containers that require GPUs.

## Deploying a Pod that Requires a GPU

Now you can deploy a Pod that requires a GPU. Here’s an example:
```shell
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-example-deployment
  labels:
    app: gpu-example-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gpu-example-app
  template:
    metadata:
      labels:
        app: gpu-example-app
    spec:
      containers:
      - name: gpu-example-app
        image: gpu-example-image
        resources:
          limits:
            nvidia.com/gpu: 1  # This specifies that the pod requires 1 GPU
```



## Using Karpenter for GPU Node Pools

For clusters that have Karpenter configured, managing GPU nodes becomes more dynamic. Unlike manual node provisioning, Karpenter automatically adjusts the node pool size based on the needs of the workloads. This means when a Pod requests a GPU, Karpenter will automatically provision a node with a GPU from the specified instance family.

You can define a NodePool for GPU instances as follows:

```shell
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: gpu
spec:
  template:
    spec:
      requirements:
      - key: karpenter.k8s.aws/instance-family
        operator: In
        values:
          - p3
      taints:
      - key: nvidia.com/gpu
        value: "true"
        effect: "NoSchedule"
```


### Pod Configuration to Use the NodePool

To run a Pod on a node that has this NodePool, set a toleration in the Pod specification:

```shell
apiVersion: v1
kind: Pod
metadata:
  name: gpu-example-pod
spec:
  containers:
  - name: gpu-example-app
    resources:
      requests:
        nvidia.com/gpu: 1
      limits:
        nvidia.com/gpu: 1
    image: gpu-example-image
  tolerations:
  - key: "nvidia.com/gpu"
    operator: "Exists"
    effect: "NoSchedule"
```
