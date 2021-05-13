# RBD with PVC

## 在k8s中使用pvc来保证RBD 一写多读

PVC 

```yaml
spec:
    accessModes:
    - ReadOnlyMany
    resources:
      requests:
        storage: 2000Gi
    volumeMode: Filesystem
    volumeName: xxx
```

在多读时，设置accessModes为ReadOnlyMany

在单写时，设置accessModes为ReadWriteOnce 或者 ReadWriteMany

## Kubelet中RBD挂载部分的代码：

当 accessModes 为ReadOnlyMany 时去检查RBD当前是否已经有挂载

```
// kubernetes/pkg/volume/rbd/rbd_util.go
// AttachDisk attaches the disk on the node.
func (util *RBDUtil) AttachDisk(b rbdMounter) (string, error) {
...
needValidUsed := true
		if b.accessModes != nil {
			// If accessModes only contains ReadOnlyMany, we don't need check rbd status of being used.
			if len(b.accessModes) == 1 && b.accessModes[0] == v1.ReadOnlyMany {
				needValidUsed = false
			}
		}
		// If accessModes is nil, the volume is referenced by in-line volume.
		// We can assume the AccessModes to be {"RWO" and "ROX"}, which is what the volume plugin supports.
		// We do not need to consider ReadOnly here, because it is used for VolumeMounts.

		if needValidUsed {
			err := wait.ExponentialBackoff(backoff, func() (bool, error) {
				used, rbdOutput, err := util.rbdStatus(&b)
				if err != nil {
					return false, fmt.Errorf("fail to check rbd image status with: (%v), rbd output: (%s)", err, rbdOutput)
				}
				return !used, nil
			})
			// Return error if rbd image has not become available for the specified timeout.
			if err == wait.ErrWaitTimeout {
				return "", fmt.Errorf("rbd image %s/%s is still being used", b.Pool, b.Image)
			}
			// Return error if any other errors were encountered during waiting for the image to become available.
			if err != nil {
				return "", err
			}
		}
...
}
```
