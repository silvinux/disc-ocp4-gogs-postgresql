## Deploy gogs with postgresql in a disconnected OCP 4 with persistent storage.

### Storage.

⋅⋅* As this guide pretend to be used as a learning resource, so the exercise it will be provisioned with persistent storage using NFS. The NFS shares will be provided by a RHEL 8 server.

##### Install Packages needed

```
$ yum install firewalld nfs-utils -y
```

##### Active services.

```
$ systemctl enable --now nfs-server rpcbind firewalld;systemctl start firewalld nfs-server rpcbind
```

##### Add firewall rules. 

```
$ firewall-cmd --add-service={nfs,nfs3,mountd,rpc-bind} --permanent;firewall-cmd --reload

```

##### Create the directories and apply the corresponding permissions.  

Take into account that [nobody user replaces nfsnobody on RHEL8](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html-single/considerations_in_adopting_rhel_8/index#the-nobody-user-replaces-nfsnobody_shells-and-command-line-tools)


```
$ sudo mkdir /var/nfsshare/{gogs,postgresql}
$ sudo chmod 777 /var/nfsshare/{gogs,postgresql}
$ sudo chown nobody:nobody /var/nfsshare/{gogs,postgresql}
```

##### Create the shares configuration and export the shares.

$ cat << EOF > /etc/exports.d/cidc.exports
"/var/nfsshare/gogs" *(rw,no_root_squash)
"/var/nfsshare/postgresql" *(rw,no_root_squash)
EOF
$ sudo exportfs -rva
```

### Images

The image [docker.io/wkulhanek/gogs](https://github.com/wkulhanek/docker-openshift-gogs) will be used. It must be downloaded first, and upload it to a registry. Podman has been used to manage the images.

##### Search and download the image.

```
$ podman search gogs
$ podman pull docker.io/wkulhanek/gogs
```

##### Save the image. You must have internet access.

```
$ podman save -o local_image_wkulhanek_gogs.tar docker.io/wkulhanek/gogs
```

##### Copy the **local_image_wkulhanek_gogs.tar** to the OCP disconnected infrastructure.

```
$ scp -i local_image_wkulhanek_gogs.tar bastion:/home/ocpuser/.
```

##### Login to the external registry.

```
$ podman login --tls-verify=false registry.docp4.lab.bcnconsulting.com:5000 --log-level debug
```

..* Load image into the external registry.

```
$ podman load -i gogs-official.tar
```

##### Tag and push the image to the external registry.

```
$ podman tag docker.io/wkulhanek/gogs registry.docp4.lab.bcnconsulting.com:5000/wkulhanek/gogs
$ podman push registry.docp4.lab.bcnconsulting.com:5000/wkulhanek/gogs --tls-verify=false
```

##### Check image 

```
$ podman search registry.docp4.lab.bcnconsulting.com:5000/wkulhanek/gogs
```

### Task to be performed by a Cluster Administrator.

#####  Create Project

```
$ oc new-project gogs
```

##### Create Quotas and Limits


```
$ cat  << EOF > compute-resources.yml 
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    pods: "5" 
    requests.cpu: "1" 
    requests.memory: "1Gi" 
    limits.cpu: "1" 
    limits.memory: 1Gi
EOF
```

```
$ cat << EOF > core-resource-limits.yml 
apiVersion: "v1"
kind: "LimitRange"
metadata:
  name: "core-resource-limits" 
spec:
  limits:
    - type: "Pod"
      max:
        cpu: "500m" 
        memory: "512Mi" 
      min:
        cpu: "100m" 
        memory: "3Mi" 
    - type: "Container"
      max:
        cpu: "250m" 
        memory: "512Mi" 
      min:
        cpu: "50m" 
        memory: "2Mi" 
      default:
        cpu: "125m" 
        memory: "100Mi" 
      defaultRequest:
        cpu: "100m" 
        memory: "50Mi" 
      maxLimitRequestRatio:
        cpu: "2"

EOF
```

```
$ cat << EOF > storage-consumption.yml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-consumption
spec:
  hard:
    persistentvolumeclaims: "6" 
    requests.storage: "1Gi
EOF
```

```
$ oc apply -f storage-consumption.yml 
$ oc apply -f core-resource-limits 
$ oc apply -f compute-resources.yml
```

##### Create Physical Volumes.

```
$ cat << EOF > pv_postgresql.yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-postgresql
spec:
  capacity:
    storage: 250Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: nfs-lb.docp4.lab.bcnconsulting.com
    path: /var/nfsshare/postgresql 
EOF

```

```
$ cat pv_gogs.yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-gogs
spec:
  capacity:
    storage: 250Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: nfs-lb.docp4.lab.bcnconsulting.com
    path: /var/nfsshare/gogs

```

```
$ oc create -f pv_gogs.yml
$ oc create -f pv_postgresql.yml
```

### Deploy postgresql server. 

**It should be done first, because gogs apps will connect to it.**

```
$ oc new-app postgresql-persistent --param POSTGRESQL_DATABASE=gogs --param POSTGRESQL_USER=gogs\
--param POSTGRESQL_PASSWORD=gogs --param VOLUME_CAPACITY=250Mi \
-lapp=postgresql --insecure-registry

```
..* Once the pods are deployed, database access should be checked.


```
POD_DB=$(oc get pods -lname=postgresql -o jsonpath='{range .items[*].metadata}{.name}{"\n"}{end}')
echo $POD_DB
oc port-forward $POD_DB 5432:5432
```

..* Open other terminal, connect with psql client.

```
        $ psql -h127.0.0.1 -Ugogs -W
        $ \l
```

### Deploy gogs. 

```
$ oc new-app registry.docp4.lab.bcnconsulting.com:5000/wkulhanek/gogs -lapp=gogs_postgresql --insecure-registry --as-deployment-config
$ oc patch dc gogs --patch='{ "spec": { "strategy": { "type": "Recreate" }}}'
$ oc set volume dc/gogs --add --overwrite --name=gogs-volume-1 --mount-path=/data/ --type persistentVolumeClaim --claim-name=gogs-data --claim-size=250Mi
$ oc expose svc gogs --hostname=gogs.apps.docp4.lab.bcnconsulting.com
---Config Via Web---
    - Database Type: PostgreSQL
    - Host: postgresql:5432
    - User: gogs
    - Password: gogs
    - Database Name: gogs
    - Run User: gogs
    - Application URL: http://gogs.docp4.lab.bcnconsulting.com
---Config Via Web---
```

##### Create a configMap to for gogs configuration.

```
$ POD_GOGS=$(oc get pods -lapp=gogs_postgresql -o jsonpath='{range .items[*].metadata}{.name}{"\n"}{end}' --field-selector=status.phase==Running)
$ oc cp $POD_GOGS:/opt/gogs/custom/conf/app.ini $PWD/app.ini
$ sed -i 's/localhost/docp4.lab.bcnconsulting.com/g' app.ini
$ oc create configmap gogs --from-file=$PWD/app.ini
$ oc set volume dc/gogs --add --name=config-volume -m /opt/gogs/custom/conf/ -t configmap --configmap-name=gogs
```

##### Sources
- https://github.com/silvinux/ocp-gogs-postgresql
