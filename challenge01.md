# Kodekloud CKS --> Lab- Challenge 1
### [Visit KodeKloud](https://learn.kodekloud.com/user/courses/cks-challenges)

![image](https://github.com/parichay/KodeKloud-CKS-Solutions/assets/5514596/9aa21438-d446-4ebb-95d4-9c14ff1d992f)

## Bring it on! ✊


_Create a deployment called 'alpha-xyz' that uses the image with the least 'CRITICAL' vulnerabilities? (Use the sample YAML file located at '/root/alpha-xyz.yaml' to create the deployment. Please make sure to use the same names and labels specified in this sample YAML file!)
Deployment has exactly '1' ready replica
data-volume is mounted at '/usr/share/nginx/html' on the pod_


_Lets identify the image with lease CRITICAL vulnerabilities:_

```console
image=$(docker images --format '{{.Repository}}:{{.Tag}}')

echo $image
```
_This has all images available on host. Let's filter them further_

```console
images=$(echo $image | tr ' ' '\n' | grep -v k8 | grep -v weave)

for i in $images; do echo $i; trivy image -s CRITICAL $i | grep -i Total ;done
```

| Image                             | Critical Vulnerabilities |
|-----------------------------------|--------------------------|
| `bitnami/nginx:latest`            | 2                        |
| `nginx:alpine`                    | 0                        |
| `busybox:latest`                  | -                        |
| `nginx:<none>`                    | Error                    |
| `nginx:latest`                    | 3                        |
| `busybox:<none>`                  | Error                    |
| `nginx:1.17`                      | 43                       |
| `nginx:1.16`                      | 43                       |
| `nginx:1.14`                      | 64                       |
| `kodekloud/fluent-ui-running:latest` | 0                    |
| Other entries not fully identified| Varies                   |
| `nginx:1.13`                      | 85                       |



Based on the output it is observed that the image with least **CRITICAL** ⚠️ issue is `nginx:alpine`

Lets update the deployment alpha-xyz

```console
vim alpha-xyz.yaml 

#update the image
- image: nginx:alpine

#update the volume

spec:
      containers:
      - image: nginx:alpine
        name: nginx
        volumeMounts:
                - name: data-volume
                  mountPath: /usr/share/nginx/html
      volumes:
              - name: data-volume
                persistentVolumeClaim:
                        claimName: alpha-pvc

```

_Lets check, if this helps us solve the challenge_ :crossed_fingers:

```shell
kubectl replace -f alpha-xyz.yaml --force
```


![image](https://github.com/parichay/KodeKloud-CKS-Solutions/assets/5514596/e523c6c0-7e6d-45a3-a1e3-e846ef4399e5)

_Looks like something is worng, lets check the logs_

![image](https://github.com/parichay/KodeKloud-CKS-Solutions/assets/5514596/0921ff87-f40f-4a4f-9b8a-79baca62aa54)

_Issue seems to be with the pvc which in our case is alpha-pvc_


```console
kubectl describe --namespace alpha pv alpha-pv 
Name:              alpha-pv
Labels:            <none>
Annotations:       <none>
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      local-storage
Status:            Available
Claim:             
Reclaim Policy:    Delete
Access Modes:      RWX
VolumeMode:        Filesystem
Capacity:          1Gi
Node Affinity:     
  Required Terms:  
    Term 0:        kubernetes.io/hostname in [controlplane]
Message:           
Source:
    Type:  LocalVolume (a persistent volume backed by local storage on a node)
    Path:  /data/pages
Events:    <none>

```
```console
kubectl describe --namespace alpha pvc alpha-pvc 
Name:          alpha-pvc
Namespace:     alpha
StorageClass:  local-storage
Status:        Pending
Volume:        
Labels:        <none>
Annotations:   <none>
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      
Access Modes:  
VolumeMode:    Filesystem
Used By:       alpha-xyz-75ddb5d4b6-ktj5k
Events:
  Type     Reason              Age                 From                         Message
  ----     ------              ----                ----                         -------
  Warning  ProvisioningFailed  2m (x162 over 42m)  persistentvolume-controller  storageclass.storage.k8s.io "local-storage" not found
```

_Seems an issue with the AccessMode of PVC_

```console
kubectl edit --namespace alpha pvc alpha-pvc
```
_change it to as below:_

``` console
#Before
accessModes:
  - ReadWriteOnce
#After
accessModes:
  - ReadWriteMany
```

_Recreate PVC_

```console
root@controlplane ~ ✖ kubectl delete --namespace alpha pvc alpha-pvc 
persistentvolumeclaim "alpha-pvc" deleted

kubectl create -f /tmp/kubectl-edit-562716351.yaml
persistentvolumeclaim/alpha-pvc created
```

_Lets test this again and try our luck one more time:_


![image](https://github.com/parichay/KodeKloud-CKS-Solutions/assets/5514596/8c41b97e-8543-40a3-a937-ccb8bdde7908)


_**Lets now tackle the custom-nginx**_

_Move the AppArmor profile '/root/usr.sbin.nginx' to '/etc/apparmor.d/usr.sbin.nginx' on the controlplane node_
_Load the 'AppArmor` profile called 'custom-nginx' and ensure it is enforced._

```console
ls
cp usr.sbin.nginx /etc/apparmor.d/
cat usr.sbin.nginx | grep profile
```

_We have identified the apparmor profile name as **custom-nginx**._ _It's time to check the apparmor status and load the profile_

```console
aa-status
apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
aa-status | grep custom-nginx
```

_We have oconfirmed that our policy is loaded, hence lets start to use it via editing the alpha-xyz deployment!_

```console
kubectl edit deployments.apps --namespace alpha alpha-xyz

#Edit As Below:

template:
    metadata:
            annotations:
                    container.apparmor.security.beta.kubernetes.io/nginx: localhost/custom-nginx

```

![image](https://github.com/parichay/KodeKloud-CKS-Solutions/assets/5514596/90a20805-421f-4f40-be16-238cd706439b)


_**Moving Ahead**_ 🚀 

_Create a NetworkPolicy called 'restrict-inbound' in the 'alpha' namespace_

_Policy Type = 'Ingress'_

_Inbound access only allowed from the pod called 'middleware' with label 'app=middleware'_

_Inbound access only allowed to TCP port 80 on pods matching the policy_

```console
vim nwpol.yaml

#Add the below
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-inbound
  namespace: alpha
spec:
  podSelector:
    matchLabels:
      app: alpha-xyz
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: middleware
    ports:
    - protocol: TCP
      port: 80
```

```shell
kubectl create -f nwpol.yaml
```

_Lets check this out and test if this what is expected !_

![image](https://github.com/parichay/KodeKloud-CKS-Solutions/assets/5514596/72ac078f-51d9-4262-af3e-5932895b8a4e)



_**One step away from coffee!**_

_Expose the 'alpha-xyz' as a 'ClusterIP' type service called 'alpha-svc'_

'alpha-svc' should be exposed on 'port: 80' and 'targetPort: 80'




```console
kubectl expose --namespace alpha deployment alpha-xyz --type ClusterIP --port 80 --target-port 80 --name alpha-svc
```


![image](https://github.com/parichay/KodeKloud-CKS-Solutions/assets/5514596/4a85c23a-f6bb-425d-93d4-1e53e64ad499)






_It's party time!_ :tada:

_Let's grab a coffee!_ ☕
