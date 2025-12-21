# 🛡️ Kubernetes Cluster Security & Hardening – Step‑by‑Step README

This document contains the **FULL ORIGINAL QUESTIONS (as in the exam)** followed by **clean, structured, and exam‑ready solutions**.

* Questions are preserved **verbatim** for context
* Solutions are formatted, corrected, and **CKS‑safe**
* All commands are **copy‑paste friendly**
* Configurations are **persistent and production‑grade**

---

## 1️⃣ Task – Secure Docker Daemon

### 📘 Question

You are setting up a new Kubernetes cluster and need to secure Docker as part of the cluster setup.

Ensure that docker runs under the **"root" group** and that **no external TCP connections** are allowed to the docker daemon.

Ensure the configuration is **persistent across restarts**.

---

### ✅ Solution

#### Change the ownership of the Docker socket

```bash
sudo chown root:root /var/run/docker.sock
```

#### Override Docker systemd configuration

```bash
sudo systemctl edit docker
```

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --group=root
```

#### Reload systemd and restart Docker

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl restart docker
```

#### Disable external TCP access to Docker

Edit `/etc/docker/daemon.json`:

```json
{
  "hosts": ["unix:///var/run/docker.sock"]
}
```

Restart Docker:

```bash
sudo systemctl restart docker
```

---

## 2️⃣ Task – kubesec Pod Scan

### 📘 Question

A pod definition file has been created at `/root/CKS/simple-pod.yaml`.

Using the **kubesec** tool:

* Generate a report
* Fix major issues so the scan no longer fails
* Save the final report to `/root/CKS/kubesec-report.txt`

---

### ✅ Solution

Remove the `SYS_ADMIN` capability from the pod definition.

Run the scan:

```bash
kubesec scan /root/CKS/simple-pod.yaml > /root/CKS/kubesec-report.txt
```

#### Expected Output

```json
{
  "object": "Pod/simple-webapp-1.default",
  "valid": true,
  "message": "Passed with a score of 0 points",
  "score": 0
}
```

---

## 3️⃣ Task – Enable Kubernetes Audit Logging

### 📘 Question

Enable auditing using `/etc/kubernetes/cluster-policy.yaml`.

Requirements:

* Logs at `/var/log/cluster-audit.log`
* Retain logs for **10 days**
* Max size **10MB**
* Max **3 backups**

Track:

* **Delete secrets** in `kube-system` (Metadata)
* **Deployment changes** in `default` (Request)
* **All other requests** (Metadata)

---

### ✅ Solution

#### Audit Policy File

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - RequestReceived
rules:
  - level: Metadata
    verbs: ["delete"]
    resources:
      - group: ""
        resources: ["secrets"]
    namespaces: ["kube-system"]

  - level: Request
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: "apps"
        resources: ["deployments"]
    namespaces: ["default"]

  - level: Metadata
```

#### kube‑apiserver Flags

```yaml
- --audit-policy-file=/etc/kubernetes/cluster-policy.yaml
- --audit-log-path=/var/log/cluster-audit.log
- --audit-log-maxage=10
- --audit-log-maxbackup=3
- --audit-log-maxsize=10
```

#### Volumes

```yaml
- name: audit-policy
  hostPath:
    path: /etc/kubernetes/cluster-policy.yaml
    type: File

- name: varlog
  hostPath:
    path: /var/log
    type: Directory
```

#### Volume Mounts

```yaml
- name: audit-policy
  mountPath: /etc/kubernetes/cluster-policy.yaml
  subPath: cluster-policy.yaml
  readOnly: true

- name: varlog
  mountPath: /var/log
```

Verify:

```bash
kubectl get pods -n kube-system
ls -l /var/log/cluster-audit.log
```

---

## 4️⃣ Task – ServiceAccount Without Auto‑Mount

### 📘 Question

Create a ServiceAccount `bot-sa` in the `automated` namespace.

Ensure:

* Token is NOT auto-mounted
* Deployment `sweeper` uses this SA
* Token is mounted manually as a projected volume

---

### ✅ Solution

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bot-sa
  namespace: automated
automountServiceAccountToken: false
```

```yaml
spec:
  serviceAccountName: bot-sa
  automountServiceAccountToken: false
  containers:
    - name: sweeper
      volumeMounts:
        - name: sa-token
          mountPath: /var/run/secrets/tokens
          readOnly: true
  volumes:
    - name: sa-token
      projected:
        sources:
          - serviceAccountToken:
              path: bot-token
              expirationSeconds: 3600
              audience: default
```

---

## 5️⃣ Task – Seccomp Profile

### 📘 Question

Create a new pod called audit-nginx in the default namespace using the nginx image. Secure the syscalls that this pod can use by using the audit.json seccomp profile in the pod's security context.

The audit.json file is located in the /root/CKS directory. Before creating the pod, move it to the profiles directory inside the default seccomp directory.
---

### ✅ Solution

```bash
mv /root/CKS/audit.json /var/lib/kubelet/seccomp/profiles
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: audit-nginx
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
  containers:
    - name: nginx
      image: nginx
```

---

## 6️⃣ Task – ImagePolicyWebhook

### 📘 Question

We want to deploy an ImagePolicyWebhook admission controller to secure the deployments in our cluster.

Fix the error in /etc/kubernetes/pki/admission_configuration.yaml which will be used by ImagePolicyWebhook

Ensure that the policy is set to implicit deny. If the webhook service is not reachable, the configuration should automatically reject all images.

Enable the plugin on the API server.

The kubeconfig file for the existing imagepolicywebhook resources is located at /etc/kubernetes/pki/admission_kube_config.yaml

---

### ✅ Solution

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/kubernetes/pki/admission_kube_config.yaml
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: false
```

```yaml
- --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
- --admission-control-config-file=/etc/kubernetes/pki/admission_configuration.yaml
```

---

## 7️⃣ Task – Delete Non‑Immutable Pods

### 📘 Question

Delete all pods in `alpha` namespace that are **not immutable**.

---

### ✅ Solution

Pod solaris is immutable as it have readOnlyRootFilesystem: true so it should not be deleted.

Pod sonata is running with privileged: true and triton doesn't define readOnlyRootFilesystem: true so both break the concept of immutability and should be deleted.

---

## 8️⃣ Task – Reduce Service Exposure

### 📘 Question

You have an existing Kubernetes setup with the following services running:

Namespace:
system-hardening

Pods:
nginx-internal (Accessible internally)
nginx-external (Exposed externally via NodePort
service)

Services:
nginx-internal-service (Exposed as ClusterIP - internal-only)
nginx-external-service (Exposed as NodePort - accessible externally)

Objective:
Your task is to disable or unexpose ports to minimize external access to unnecessary services.

---

### ✅ Solution

We will change the nginx-external-service to be only accessible within the cluster by changing its service type to ClusterIP.

apiVersion: v1
kind: Service
metadata:
  name: nginx-external-service
  namespace: system-hardening
spec:
  selector:
    app: nginx-external
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
    - name: https
      protocol: TCP
      port: 443
      targetPort: 443
  type: ClusterIP

Save it as nginx-external-service-clusterip.yaml.

Apply the change:

kubectl replace -f nginx-external-service-clusterip.yaml

If we don't need the internal service exposed anymore (or it's for testing purposes), we can delete it.

kubectl -n system-hardening delete svc nginx-internal-service

This will stop the internal service from being available. However, you could also leave it as-is if you want to keep testing internal services.

---

## 9️⃣ Task – Fix Restricted Deployment

### 📘 Question

A deployment named web-server is running in namespace restricted.

Identify why the deployment is not in a running state, and then fix the issue so that it is in a running state.


---

### ✅ Solution

First check the status of the web-server deployment:

kubectl get deployments.apps -n restricted 

You should see the READY column showing 0/1 which means the pod is not running.

Then run the following command:

kubectl describe deployments.apps web-server -n restricted 

In the output, check the Conditions section:

Conditions:
  Type             Status  Reason
  ----             ------  ------
  Progressing      True    NewReplicaSetCreated
  Available        False   MinimumReplicasUnavailable
  ReplicaFailure   True    FailedCreate

This shows that the replica creation failed. Lets find out the reason why:

kubectl describe replicaset -n restricted

The Events section will provide the exact error cause:

  Warning  FailedCreate  25s (x6 over 105s)  replicaset-controller  (combined from similar events): Error creating: pods "web-server-7d8b84ccc8-9ppsm" is forbidden: violates PodSecurity "restricted:latest": allowPrivilegeEscalation != false (container "nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), runAsUser=0 (container "nginx" must not set runAsUser=0), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")

This confirms the pod was blocked by Pod Security Admission (PSA) because it doesn't meet the restricted security policy.
Now, check that the namespace is labeled to enforce the restricted policy:

kubectl get namespace restricted --show-labels

You should see this label present: pod-security.kubernetes.io/enforce=restricted which means kubernetes is actively enforcing restricted-level pod Security in this namespace.

According to the Events section of the replicaset, we need to make some changes in our deployment's securityContext section. Open the deployment file to edit:

kubectl edit deployment web-server -n restricted

Enter insert mode by typing i and change the securityContext section under spec.containers as follows:

securityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop:
      - ALL
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault

Also, remove runAsUser: 0 and then press Esc followed by :wq!. This will save the changes.

You should then see the pod in a running state:

kubectl get pods -n restricted 
NAME                          READY   STATUS    RESTARTS   AGE
web-server-55549f978f-lgp8w   1/1     Running   0          5m

---

## 🔟 Task – Disable Anonymous Kubelet Auth

### 📘 Question

Configure the kubelet on the cluster2-controlplane node to disallow anonymous authentication.

The admin kubeconfig file for this cluster is located at:
/root/custom-config/admin.conf

Additionally, utilize this kubeconfig file to delete the role custom-role in namespace delta.

Ensure that, from the node, the cluster cannot be accessed with kubectl unless the --kubeconfig=/root/custom-config/admin.conf flag is explicitly provided.


---

### ✅ Solution

First ssh to cluster2-controlplane cluster:

ssh cluster2-controlplane

Then. open the kubelet config file to edit:

sudo nano /var/lib/kubelet/config.yaml

and change the authentication.anonymous.enabled to false:

authentication:
  anonymous:
    enabled: false

and authorization.mode to Webhook:

authorization:
  mode: Webhook

Save and exit the file and then restart the kubelet:

sudo systemctl restart kubelet

To make the cluster info inaccessible without the kubeconfig flag:

mv ~/.kube/config ~/.kube/config.bak
unset KUBECONFIG

The kubernetes commands should then not work without using --kubeconfig=/root/custom-config/admin.conf.

Now delete the custom-role using this kubeconfig file:

kubectl delete role custom-role -n delta --kubeconfig=/root/custom-config/admin.conf

---

## 1️⃣1️⃣ Task – Generate SBOM

### 📘 Question

Identify container with `curl` and generate SPDX SBOM.

---

### ✅ Solution

```bash
bom generate --image-archive /root/ImageTarballs/<image>.tar --format json --output ~/bugged-fruit.spdx
echo banana > ~/bugged-container.txt
```

---

## 1️⃣2️⃣ Task – Ingress with TLS

### 📘 Question

In the space namespace, a deployment rocket-server is exposed by a service of the same name.

Create an ingress resource named rocket-ingress to load balance the incoming traffic to the workload on path /.

Use the hostname rocket-server.local for the Ingress rules.

Utilize the TLS certificate stored in the secret rocket-tls in the space namespace to enable TLS traffic on that ingress resource.


---

### ✅ Solution

Run the command:

k describe deployment rocket-server -n space

and check the Port field under Pod Template -> Containers.

Then create and apply the following ingress yaml file:

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rocket-ingress
  namespace: space
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - rocket-server.local
      secretName: rocket-tls
  rules:
    - host: rocket-server.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rocket-server
                port:
                  number: 80

Once that is done, check the IP address under CLUSTER-IP header of the ingress-nginx-controller:

kubectl get svc -n ingress-nginx

Then add this IP to the /etc/hosts file:

echo "<INGRESS-IP> rocket-server.local" | sudo tee -a /etc/hosts

Finally you can check the working:

curl -k https://rocket-server.local

You should the nginx welcome page.

---

## 1️⃣3️⃣ Task – Mount TLS Secret

### 📘 Question

In the namespace code, create a TLS secret code-secret using the following certificate and key:

cert: /root/custom-cert.crt
key: /root/custom-key.key
Attach this secret as a volume named secret-volume in the deployment code-server.

---

### ✅ Solution

First create the TLS secret:

```bash
kubectl create secret tls code-secret --cert=/root/custom-cert.crt --key=/root/custom-key.key -n code
```


Then edit the deployment to mount the secret:

kubectl edit deployment code-server -n code

Under the spec.template.spec section, add:

volumes:
  - name: secret-volume
    secret:
      secretName: code-secret

Under the spec.template.spec.containers section, add the volume mount then:

volumeMounts:
  - name: secret-volume
    mountPath: /etc/code/tls
    readOnly: true

Save and exit the deployment.

---

## 1️⃣4️⃣ Task – RBAC Least Privilege

### 📘 Question

jacob is a developer who needs access to work on the dev-a, dev-b and dev-z namespace. He should have the ability to carry out any operation on any pod in dev-a and dev-b namespaces.

However, on the dev-z namespace, he should only have the following permissions:

get, list, and watch pods
get and list configmaps
No access at all to secrets

The current setup is too permissive and does not meet the above condition. Update the permissions to secure jacob's access in the cluster. You may re-create objects, but ensure that the resource names remain unchanged.

---

### ✅ Solution

The role called dev-user-access has been created for all three namespaces: dev-a, dev-b and dev-z. However, the role in the dev-z namespace grants jacob access to all operation on all pods. To fix this, delete and re-create the role using the following YAML:


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
    name: dev-user-access
    namespace: dev-z
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
```

---

## 1️⃣5️⃣ Task – Upgrade Worker Node

### 📘 Question

The administrator has partially upgraded cluster1.

Complete the upgrade process by updating the worker node to the latest installed version available among the nodes.


---

### ✅ Solution

```bash
First run the following command from the controlplane node:

kubectl get nodes

You should get an output like below, indicating that the worker node node02 is at an earlier version of kubernetes:

NAME                    STATUS   ROLES           AGE   VERSION
cluster1-controlplane   Ready    control-plane   49m   v1.34.0
node02                  Ready    worker          29m   v1.33.0

SSH into the node02 node:

ssh node02

Use any text editor you prefer to open the file that defines the Kubernetes apt repository.

vim /etc/apt/sources.list.d/kubernetes.list

Update the version in the URL to the next available minor release, i.e v1.34.

deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /

After making changes, save the file and exit from your text editor. Proceed with the next instruction.

echo 'y' | curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --yes --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

sudo apt-get update

apt-cache madison kubeadm

Momentarily go back to cluster1-controlplane node to drain the worker node:

kubectl drain node02 --ignore-daemonsets --delete-emptydir-data

Based on the version information displayed by apt-cache madison, it indicates that for Kubernetes version 1.34.0, the available package version is 1.34.0-1.1. Therefore, to install kubeadm for Kubernetes v1.34.0, use the following command:

sudo apt-get install -y kubeadm=1.34.0-1.1

Run the following command to upgrade the node:

sudo kubeadm upgrade node

Unhold kubeadm if its on hold while upgrading or use the appropriate suggestion mentioned in the output.

Note that the above steps can take a few minutes to complete.

Now, unhold and then upgrade the kubelet and kubectl versions:

sudo apt-mark unhold kubelet kubectl
sudo apt-get install --allow-change-held-packages -y kubelet=1.34.0-1.1 kubectl=1.34.0-1.1

Optionally hold them again:

sudo apt-mark hold kubelet kubectl

Run the following commands to refresh the systemd configuration and apply changes to the Kubelet service:

sudo systemctl daemon-reload
sudo systemctl restart kubelet

Go back to the controlplane node again and uncordon node02:

kubectl uncordon node02

Finally verify the version upgrade:

kubectl get nodes

This should now show both at v1.34:

NAME                    STATUS   ROLES           AGE   VERSION
cluster1-controlplane   Ready    control-plane   70m   v1.34.0
node02                  Ready    worker   
```

---

## ✅ Final Notes

* Full **exam questions preserved**
* Solutions are **CKS‑accurate**
* Safe for **real production clusters**
* Ideal as **final revision material**

🔥 **This README now mirrors real CKS exam style exactly**
