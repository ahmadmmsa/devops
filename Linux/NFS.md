


only on control plane
```bash
sudo apt install -y nfs-kernel-server nfs-common
```

install on all kubernetes nodes

```bash
sudo apt install -y nfs-common
```

<br>

-----

```bash
sudo mkdir -p /srv/odoo/filestore
sudo chown 1001:1001 /srv/odoo/filestore
sudo chmod 777 /srv/odoo/filestore
```

```bash
nano /etc/exports
```

```bash
/srv/odoo/filestore 192.168.8.0/24(rw,sync,no_subtree_check,no_root_squash)
```

```bash
sudo exportfs -rav
sudo systemctl restart nfs-kernel-server
sudo systemctl enable nfs-kernel-server
```

```bash
showmount -e 192.168.8.100
```
> /srv/odoo/filestore 192.168.8.0/24


<br>

-----


Install NFS CSI Driver
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/v4.11.0/deploy/install-driver.sh
```
Verify installation

```bash
kubectl get pods -n kube-system | grep nfs
```

<br>

-----

Create StorageClass

```bash
nano nfs-storageclass.yaml
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.8.100
  share: /srv/odoo/filestore
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - nfsvers=4.1
```

apply
```bash
kubectl apply -f nfs-storageclass.yaml
```

Verify
```bash
kubectl get storageclass
```


(Optional) Set as Default StorageClass

```bash
kubectl patch storageclass nfs-csi -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

<br>

-----

Create a PVC

```bash
nano odoo-pvc.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: odoo-filestore-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 50Gi
```

```bash
kubectl apply -f odoo-pvc.yaml
```

docker-compose.yaml

```yaml
services:
  odoo:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: odoo_app
    restart: unless-stopped
    ports:
      - "8069:8069"   # main web UI
      - "8071:8071"   # xmlrpc
      - "8072:8072"   # longpolling (live chat / bus)

    volumeMounts:
     - name: filestore
       mountPath: /var/lib/odoo

    volumes:
      - name: filestore
        persistentVolumeClaim:
         claimName: odoo-filestore-pvc  
```