# my-kubevirt-nvidia
```
# Windows
https://medium.com/adessoturkey/create-a-windows-vm-in-kubernetes-using-kubevirt-b5f54fb10ffd


k apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: winhd
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 35Gi
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: iso-win10
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/domain: iso-win10
    spec:
      domain:
        cpu:
          cores: 6
        devices:
          disks:
          - bootOrder: 1
            cdrom:
              bus: sata
            name: cdromiso
          - bootOrder: 2
            disk:
              bus: virtio
            name: harddrive
          - bootOrder: 3
            cdrom:
              bus: sata
            name: virtiocontainerdisk
        machine:
          type: q35
        resources:
          requests:
            memory: 12G
      volumes:
      - name: cdromiso
        persistentVolumeClaim:
          claimName: iso-win10
      - name: harddrive
        persistentVolumeClaim:
          claimName: winhd
      - containerDisk:
          image: quay.io/kubevirt/virtio-container-disk
        name: virtiocontainerdisk
EOF
```
