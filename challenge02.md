# Kodekloud CKS --> Challenge 2  
### [Visit KodeKloud](https://learn.kodekloud.com/user/courses/cks-challenges)

## Avoid Unnecessary Files in Docker Image - DOCKERFILE Refinement

_To keep our Docker image clean and efficient, we need to ensure that only the necessary files are copied into the image. We'll move the required files into a subdirectory and update the Dockerfile accordingly._

```console
cd webapp/
mkdir app && mv app.py requirements.txt templates/ app
vim Dockerfile
```
_Make the below changes:_

```console
#Before 
COPY . /opt/
# After
COPY ./app/ /opt/

#Remove the following 2 lines

Expose port 22
EXPOSE 22

#Change user from root to worker

#Before
USER root
#After  
USER worker
```
_Save and quit (:wq) & Build the Docker image:_
```console
docker build -t kodekloud/webapp-color:stable .
```

![image](https://github.com/parichay/KK-CKS-Solutions/assets/5514596/97df06ab-425c-45bf-a07a-92d1c5ad0e0f)



## Security Scanning with Kubesec [Kubesec Requirements] 
_We'll scan the Kubernetes YAML files using kubesec and address any security issues found._

```console
ls  # Ensure you are in the root's home directory

kubesec scan dev-webapp.yaml

kubesec scan staging-webapp.yaml
```

### Observations:

_"reason": "***CAP_SYS_ADMIN*** is the most privileged capability and should always be avoided"
"selector": "containers[] .securityContext .allowPrivilegeEscalation == ***true***"_


_Edit the YAML files to address the issues:_

```console

vim dev-webapp.yaml
```

_Modify the security settings:_

```console
#Before
allowPrivilegeEscalation: true
image: kodekloud/webapp-color
capabilities:
  add: ["SYS_ADMIN"]
```

```console
#After
allowPrivilegeEscalation: false
image: kodekloud/webapp-color:stable
capabilities:
  add: []  # Remove SYS_ADMIN
```

_Repeat the same for staging-webapp.yaml and scan the YAML files again to ensure they are clean:_


```console
kubesec scan dev-webapp.yaml
kubesec scan staging-webapp.yaml
```

_This should solve the task and you should see the kubesec square glow green_ 😄


![image](https://github.com/parichay/KK-CKS-Solutions/assets/5514596/dc155eb2-bddd-44f6-a9f8-b7adb3a0b15f)


## Pods Configuration: Startup Probe for Pods


_Edit the YAML files for the dev-webapp and staging-webapp pods to add a startup probe as follows:_

```console
vim dev-webapp.yaml
````
_Add the following lines after imagePullPolicy: Never_

```console
imagePullPolicy: Never
startupProbe:
  exec:
    command:
      - sh
      - -c
      - "rm /bin/sh && rm /bin/ash"
  initialDelaySeconds: 5
  periodSeconds: 5
```

_Repeat for staging-webapp.yaml._

```console
vim staging-webapp.yaml
```
```console
imagePullPolicy: Never
startupProbe:
  exec:
    command:
      - sh
      - -c
      - "rm /bin/sh && rm /bin/ash"
  initialDelaySeconds: 5
  periodSeconds: 5
```

_Apply the changes and replace the pods:_

```console
kubectl replace -f dev-webapp.yaml --force
kubectl replace -f staging-webapp.yaml --force
```

_This should solve the task and you should see the Pods configurations square glows green_ :heart:


![image](https://github.com/parichay/KK-CKS-Solutions/assets/5514596/3762c9a6-8c77-4eeb-80f6-5501395ba86f)


## Deployment Configuration:  Managing Secrets in Deployment

_We'll create a Kubernetes secret to manage sensitive information and update the deployment configuration._
_Create the secret:_

```console
kubectl create secret generic prod-db --namespace prod --from-literal=DB_Host=prod-db --from-literal=DB_User=root --from-literal=DB_Password=paswrd
```

_Confirm the secret creation:_
```console
kubectl get secrets --namespace prod
```

_Edit the deployment to use the secret:_
```console
kubectl edit --namespace prod deployments.apps prod-web and replace the environment variables with references to the secret:

env:
  - name: DB_Host
    valueFrom:
      secretKeyRef:
        name: prod-db
        key: DB_Host
  - name: DB_User
    valueFrom:
      secretKeyRef:
        name: prod-db
        key: DB_User
  - name: DB_Password
    valueFrom:
      secretKeyRef:
        name: prod-db
        key: DB_Password

```
_This should solve the task and you should see the deployment configurations as requested and hence square glows green, Good job!_ &#128077;

![image](https://github.com/parichay/KK-CKS-Solutions/assets/5514596/08a27ce9-bb1c-46a0-84d3-b76c5bc8c87b)


## Network Policy: Implementing Network Policy to restrict ingress traffic to the prod namespace.

```console
vim nwpol.yaml
```

_Add the following configuration:_

```console
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prod-netpol
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: prod
```

_Save and quit, then apply the network policy:_

```console
kubectl apply -f nwpol.yaml
```
![image](https://github.com/parichay/KK-CKS-Solutions/assets/5514596/ce5aaca8-1bc0-4889-8a82-75ede817517a)


_By following these steps, you should have a secure, efficient, and properly configured setup as requested in the challenege. Enjoy solving the challenge and happy deploying!_


_It's party time!_ :tada:

_Let's grab a coffee!_ ☕






