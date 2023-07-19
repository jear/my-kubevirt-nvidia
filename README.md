# my-kubevirt-nvidia
```


# Windows
https://medium.com/adessoturkey/create-a-windows-vm-in-kubernetes-using-kubevirt-b5f54fb10ffd

k virt image-upload pvc iso-win10 --size=10G --image-path=/home/ubuntu/workspace/github/win.iso --uploadproxy-url=https://10.69.41.74 --insecure

```

# Kubevirt
```
# if you can access to nvid.nvidia.com..... Build nVidia vGPU (for kubevirt use case )
Download the vGPU Software from the NVIDIA Licensing Portal. https://nvid.nvidia.com/dashboard/#/dashboard
https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-kubevirt.html#build-vgpu-manager-image
# Install the GPU Operator (without NVIDIA vGPU)
helm .........  --set sandboxWorkloads.enabled=true
```

```
# Host preparation for PCI Passthrough
https://kubevirt.io/user-guide/virtual_machines/host-devices/#host-preparation-for-pci-passthrough
```

```
# kubevirt install https://kubevirt.io/quickstart_cloud/
echo $VERSION
v1.0.0
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-operator.yaml
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-cr.yaml


# uninstall
kubectl delete -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-cr.yaml
kubectl delete -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-operator.yaml
```


```
# Kubevirt and nVidia pGPU ( passthrough GPU )

https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-kubevirt.html


https://kubevirt.io/user-guide/virtual_machines/host-devices/#listing-permitted-devices

# Get device ID
####
ssh worker-gpu-1
lspci -nnv|grep -i nvidia
21:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM204GL [Tesla M6] [10de:13f3] (rev a1) (prog-if 00 [VGA controller])
	Subsystem: NVIDIA Corporation GM204GL [Tesla M6] [10de:1143]
	Kernel driver in use: nvidia
	Kernel modules: nvidiafb, nouveau

lspci -DD|grep NVIDIA
0000:21:00.0 VGA compatible controller: NVIDIA Corporation GM204GL [Tesla M6] (rev a1)

####
ssh worker-gpu-2
23:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM204GL [Tesla M6] [10de:13f3] (rev a1) (prog-if 00 [VGA controller])
	Subsystem: NVIDIA Corporation GM204GL [Tesla M6] [10de:1143]
	Kernel driver in use: nvidia
	Kernel modules: nvidiafb, nouveau
26:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM204GL [Tesla M6] [10de:13f3] (rev a1) (prog-if 00 [VGA controller])
	Subsystem: NVIDIA Corporation GM204GL [Tesla M6] [10de:1143]
	Kernel driver in use: nvidia
	Kernel modules: nvidiafb, nouveau

lspci -DD|grep NVIDIA
0000:23:00.0 VGA compatible controller: NVIDIA Corporation GM204GL [Tesla M6] (rev a1)
0000:26:00.0 VGA compatible controller: NVIDIA Corporation GM204GL [Tesla M6] (rev a1)

####
# FeatureGate
( https://kubevirt.io/user-guide/operations/activating_feature_gates/ )

# permittedHostDevices
( https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-kubevirt.html#add-gpu-resources-to-kubevirt-cr )

k get kubevirts.kubevirt.io -n kubevirt kubevirt -o yaml

kubectl edit kubevirts.kubevirt.io -n kubevirt
...
spec:
  certificateRotateStrategy: {}
  configuration:
    developerConfiguration:
      featureGates:
      - GPU
      - DisableMDEVConfiguration
#      - HostDevices
#      - LiveMigration
    permittedHostDevices:
      gpus:
      - externalResourceProvider: true
        pciVendorSelector: "10DE:13f3"
        resourceName: "nvidia.com/GM204GL"
#      mediatedDevices:
#      - externalResourceProvider: true
#        mdevNameSelector: NVIDIA A10-24Q
#        resourceName: nvidia.com/NVIDIA_A10-24Q
#      mediatedDevices:
#      - mdevNameSelector: "GRID T4-1Q"
#        resourceName: "nvidia.com/GRID_T4-1Q"
#      hostDevices:
#      - deviceName: intel.com/qat
#        name: quickaccess1


...

```

```
# patching kubevirt kind
kubectl patch kubevirt  -n kubevirt kubevirt --type='json' \
        -p='[{"op": "add", "path": "/spec/configuration/developerConfiguration/featureGates/-", "value": "DisableMDEVConfiguration" }]'
The request is invalid: the server rejected our request due to an error in our request

kubectl patch kubevirt  -n kubevirt kubevirt --type=merge \
        -p='[{"op": "add", "path": "/spec/configuration/developerConfiguration/featureGates/-", "value": "DisableMDEVConfiguration" }]'
The request is invalid: patch: Invalid value: "[{\"op\":\"add\",\"path\":\"/spec/configuration/developerConfiguration/featureGates/-\",\"value\":\"DisableMDEVConfiguration\"}]": couldn't get version/kind; json parse error: json: cannot unmarshal array into Go value of type struct { APIVersion string "json:\"apiVersion,omitempty\""; Kind string "json:\"kind,omitempty\"" }

# Nested virtualization
kubectl patch kubevirt -n kubevirt  kubevirt --type=merge \
        --patch '{"spec":{"configuration":{"developerConfiguration":{"useEmulation":true}}}}'
```

```
# label worker GPU nodes ( here to allo vm-passthrough )
kubectl label node worker-gpu-1 --overwrite nvidia.com/gpu.workload.config=vm-passthrough
kubectl label node worker-gpu-2 --overwrite nvidia.com/gpu.workload.config=vm-passthrough

k get pods -n gpu-operator 
NAME                                                          READY   STATUS      RESTARTS   AGE
gpu-feature-discovery-wgg2t                                   1/1     Running     0          5d1h
gpu-operator-85949d795c-v4nxg                                 1/1     Running     6          5d1h
gpu-operator-node-feature-discovery-master-7b7d875f6b-qqhjg   1/1     Running     3          5d1h
gpu-operator-node-feature-discovery-worker-b24zx              1/1     Running     9          5d1h
gpu-operator-node-feature-discovery-worker-h9kcj              1/1     Running     3          5d1h
nvidia-container-toolkit-daemonset-p4m75                      1/1     Running     0          5d1h
nvidia-cuda-validator-jbdrp                                   0/1     Completed   0          31m
nvidia-cuda-validator-z6tlf                                   0/1     Completed   0          5d1h
nvidia-dcgm-exporter-zczh7                                    1/1     Running     0          5d1h
nvidia-device-plugin-daemonset-99zm6                          1/1     Running     0          5d1h
nvidia-device-plugin-validator-r4hdj                          0/1     Completed   0          5d1h
nvidia-device-plugin-validator-znr2z                          0/1     Completed   0          31m
nvidia-driver-daemonset-svj6d                                 1/1     Running     0          5d1h
nvidia-operator-validator-s52g7                               1/1     Running     0          5d1h

nvidia-sandbox-device-plugin-daemonset-xdjqg                  1/1     Running     0          66s
nvidia-sandbox-validator-p5dvq                                1/1     Running     0          66s
nvidia-vfio-manager-sswb8                                     1/1     Running     0          112s

```
