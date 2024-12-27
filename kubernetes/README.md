# PersistentVolume to allow ReadWriteMany

## **Why a PersistentVolume is Necessary**
In Kubernetes, a **PersistentVolume (PV)** provides a way to manage and persist storage independent of Pods and their lifecycle. It ensures that data remains available even if Pods are deleted or rescheduled.

For deployments like **Nextcloud**, which require persistent storage for configuration files, user data, and applications, a properly configured PV is essential. The PV works alongside a **PersistentVolumeClaim (PVC)**, which allows Pods to request specific storage resources.

When using **Topolvm**, you might encounter challenges if the `storageClassName` is mismatched or the `topolvm-provisioner` is not properly installed. In such cases, a manually defined NFS-backed PV provides a reliable alternative.

---

## **Handling the Topolvm Issue**
### **The Problem**
When using `storageClassName: topolvm-provisioner`, the following issues may arise:
- The `topolvm-provisioner` is not installed or running.
- The `storageClassName` in the PVC and PV do not match, preventing the PVC from binding to the PV.
- Topolvm may not support certain access modes like `ReadWriteMany` (RWX), which are often required for shared storage setups.

---

## **Setting Up NFS for Kubernetes**
Using NFS as a backend for PVs allows multiple Pods to share the same storage.

### **Steps to Set Up NFS**
1. **Install NFS Server on the Host Machine:**
   - Install the NFS utilities:
     ```bash
     sudo yum install -y nfs-utils
     ```

2. **Create and Export a Shared Directory:**
   - Create a directory for the shared storage:
     ```bash
     sudo mkdir -p /srv/nfs/kubedata
     ```
   - Set appropriate permissions:
     ```bash
     sudo chown -R nobody:nobody /srv/nfs/kubedata
     sudo chmod -R 755 /srv/nfs/kubedata
     ```
   - Add an export entry in `/etc/exports`:
     ```plaintext
     /srv/nfs/kubedata *(rw,sync,no_subtree_check,no_root_squash)
     ```
   - Restart the NFS server:
     ```bash
     sudo systemctl enable nfs-server
     sudo systemctl start nfs-server
     ```

3. **Verify the Export:**
   - Check that the NFS export is visible:
     ```bash
     showmount -e
     ```

4. **Test Mounting the NFS Share:**
   - Mount the NFS share on the same or another machine to confirm:
     ```bash
     sudo mount -t nfs 127.0.0.1:/srv/nfs/kubedata /mnt
     ```

---

## **Creating the PersistentVolume (PV)**
Once the NFS server is ready, you can create a Kubernetes PV.

### **Example PV Manifest**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany
  storageClassName: topolvm-provisioner
  nfs:
    path: /srv/nfs/kubedata
    server: 127.0.0.1
```

### **Create the PV**
Apply the PV manifest:
```bash
kubectl apply -f nfs-pv.yaml
```

### **Verify the PV**
Ensure the PV is available:
```bash
kubectl get pv
```