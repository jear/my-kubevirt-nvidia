# GPU operator
```
#  for Kubevirt with passthrough and sandboxing
helm upgrade --install gpu-operator nvidia/gpu-operator -n gpu-operator --create-namespace \
             --set operator.upgradeCRD=true --disable-openapi-validation \
             --set sandboxWorkloads.enabled=true

#  for Kubevirt with vGPU and sandboxing
helm upgrade --install gpu-operator nvidia/gpu-operator -n gpu-operator --create-namespace  \
             --set operator.upgradeCRD=true --disable-openapi-validation \
             --set sandboxWorkloads.enabled=true \
             --set vgpuManager.enabled=true 
```

```
# For Node selector
kubectl get node -o json | jq '.items[].metadata.labels' | grep -i product

  "nvidia.com/gpu.product": "Tesla-M6",

```


# Kubevirt + Rook-Ceph + nVidia GPU operator and passthrough

```
# Test Kubevirt Windows without attaching a nVidia GPU   => OK
https://medium.com/adessoturkey/create-a-windows-vm-in-kubernetes-using-kubevirt-b5f54fb10ffd

https://kubevirt.io/user-guide/virtual_machines/service_objects/
```


```
sudo vi /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="maybe-ubiquity"
GRUB_CMDLINE_LINUX="quiet splash pci=realloc=off intel_iommu=on nouveau.modeset=0"
#### nofb kvm.ignore_msrs=1 vfio-pci.ids=xxxx:yyyy

sudo update-grub



cat /etc/modprobe.d/blacklist-nvidia-nouveau.conf 
blacklist nouveau
options nouveau modeset=0

#cat /etc/modprobe.d/vfio.conf
#blacklist snd_hda_intel
#options vfio-pci ids=xxxx:yyyy


sudo update-initramfs -u


# reboot
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
lspci -nnv|grep -i nvidia
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

spec:
  configuration:
    developerConfiguration:
      featureGates:
      - GPU
      - DisableMDEVConfiguration
    permittedHostDevices:
      pciHostDevices:
      - pciVendorSelector: 10DE:13f3
        resourceName: nvidia.com/XXXXXXXXXXXXXXX


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


# Troubleshooting
```
# Host preparation for PCI Passthrough
https://kubevirt.io/user-guide/virtual_machines/host-devices/#host-preparation-for-pci-passthrough
https://askubuntu.com/questions/1406888/ubuntu-22-04-gpu-passthrough-qemu

lspci -nnk | grep -i nvidia
23:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM204GL [Tesla M6] [10de:13f3] (rev a1)
	Subsystem: NVIDIA Corporation GM204GL [Tesla M6] [10de:1143]
	Kernel modules: nvidiafb, nouveau
26:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM204GL [Tesla M6] [10de:13f3] (rev a1)
	Subsystem: NVIDIA Corporation GM204GL [Tesla M6] [10de:1143]
	Kernel modules: nvidiafb, nouveau

# Check for IOMMU Support on your CPU
# For AMD processor:
$ cat /proc/cpuinfo | grep --color svm
# For Intel processor:
$ cat /proc/cpuinfo | grep --color vmx

# Check that IOMMU is enabled
sudo dmesg | grep -i -e DMAR -e IOMMU
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
modinfo vfio
name:           vfio
filename:       (builtin)
softdep:        post: vfio_iommu_type1 vfio_iommu_spapr_tce
alias:          devname:vfio/vfio
alias:          char-major-10-196
description:    VFIO - User Level meta-driver
author:         Alex Williamson <alex.williamson@redhat.com>
license:        GPL v2
version:        0.3
parm:           enable_unsafe_noiommu_mode:Enable UNSAFE, no-IOMMU mode.  This mode provides no device isolation, no DMA translation, no host kernel protection, cannot be used for device assignment to virtual machines, requires RAWIO permissions, and will taint the kernel.  If you do not know what this is for, step away. (default: false) (bool)



lspci -DD|grep NVIDIA
0000:23:00.0 VGA compatible controller: NVIDIA Corporation GM204GL [Tesla M6] (rev a1)
0000:26:00.0 VGA compatible controller: NVIDIA Corporation GM204GL [Tesla M6] (rev a1)

# Bind that device to the vfio-pci driver: 
https://kubevirt.io/user-guide/virtual_machines/host-devices/#preparation-of-pci-devices-for-passthrough

echo 0000:23:00.0 > /sys/bus/pci/drivers/nvidia/unbind
echo "vfio-pci" > /sys/bus/pci/devices/0000\:23\:00.0/driver_override
echo 0000:23:00.0 > /sys/bus/pci/drivers/vfio-pci/bind

echo 0000:26:00.0 > /sys/bus/pci/drivers/nvidia/unbind
echo "vfio-pci" > /sys/bus/pci/devices/0000\:26\:00.0/driver_override
echo 0000:26:00.0 > /sys/bus/pci/drivers/vfio-pci/bind


```

```
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
