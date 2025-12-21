# 🛡️ Kubernetes Cluster Security & Hardening – Step‑by‑Step README

This document consolidates **all tasks and solutions** into a clean, structured, and exam‑ready **Markdown README**.
All commands are **copy‑paste friendly**, configurations are **persistent**, and steps follow **CKS best practices**.

---

## 1️⃣ Secure Docker Daemon

### 🎯 Objective

* Run Docker under the `root` group
* Disable all external TCP access to Docker
* Ensure persistence across restarts

### ✅ Solution

#### 1. Change Docker socket ownership

```bash
sudo chown root:root /var/run/docker.sock
```

#### 2. Override Docker systemd configuration

```bash
sudo systemctl edit docker
```

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --group=root
```

#### 3. Reload and restart Docker

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl restart docker
```

#### 4. Disable TCP access

Edit `/etc/docker/daemon.json`:

```json
{
  "hosts": ["unix:///var/run/docker.sock"]
}
```

Restart Docker again:

```bash
sudo systemctl restart docker
```

---

## 2️⃣ Fix Pod Security Issues Using kubesec

### 🎯 Objective

* Scan pod definition using `kubesec`
* Fix critical failures
* Save final passing report

### ✅ Solution

#### 1. Fix pod definition

Remove `SYS_ADMIN` capability from `/root/CKS/simple-pod.yaml`

#### 2. Generate report

```bash
kubesec scan /root/CKS/simple-pod.yaml > /root/CKS/kubesec-report.txt
```

#### ✅ Expected Result

```json
{
  "valid": true,
  "message": "Passed with a score of 0 points"
}
```

---

## 3️⃣ Enable Kubernetes Audit Logging

### 🎯 Objective

* Enable auditing
* Retain logs for **10 days**
* Max **10MB** per file
* Keep **3 backups**

### ✅ Audit Policy (`/etc/kubernetes/cluster-policy.yaml`)

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

### ✅ kube‑apiserver Configuration

Edit `/etc/kubernetes/manifests/kube-apiserver.yaml`

#### Flags

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

## 4️⃣ ServiceAccount Without Auto‑Mount

### 🎯 Objective

* Create `bot-sa`
* Disable auto token mount
* Manually mount projected token

### ✅ ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bot-sa
  namespace: automated
automountServiceAccountToken: false
```

### ✅ Deployment Update

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

## 5️⃣ Apply Seccomp Profile to Pod

### 🎯 Objective

* Secure syscalls using custom seccomp profile

### ✅ Steps

```bash
mv /root/CKS/audit.json /var/lib/kubelet/seccomp/profiles
```

### Pod Definition

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

## 6️⃣ ImagePolicyWebhook Admission Controller

### 🎯 Objective

* Implicit deny on webhook failure

### ✅ Admission Configuration

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

### Enable Plugin

```yaml
- --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
- --admission-control-config-file=/etc/kubernetes/pki/admission_configuration.yaml
```

---

## 7️⃣ Delete Non‑Immutable Pods

### 🎯 Objective

* Remove pods with mutable state or elevated privileges

### ✅ Action

* ❌ Delete: `sonata`, `triton`
* ✅ Keep: `solaris` (readOnlyRootFilesystem enabled)

---

## 8️⃣ Minimize Service Exposure

### 🎯 Objective

* Remove unnecessary external access

### ✅ Convert NodePort → ClusterIP

```yaml
type: ClusterIP
```

```bash
kubectl replace -f nginx-external-service-clusterip.yaml
kubectl delete svc nginx-internal-service -n system-hardening
```

---

## 9️⃣ Fix Restricted Deployment Failure

### 🎯 Objective

* Fix PSA `restricted` violations

### ✅ Required Security Context

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
```

---

## 🔟 Disable Anonymous Kubelet Auth

### 🎯 Objective

* Secure kubelet
* Use admin kubeconfig

### ✅ kubelet config

```yaml
authentication:
  anonymous:
    enabled: false
authorization:
  mode: Webhook
```

```bash
sudo systemctl restart kubelet
```

Delete role:

```bash
kubectl delete role custom-role -n delta --kubeconfig=/root/custom-config/admin.conf
```

---

## 1️⃣1️⃣ Generate SBOM (SPDX)

```bash
bom generate \
  --image-archive /root/ImageTarballs/<image>.tar \
  --format json \
  --output ~/bugged-fruit.spdx
```

Save container name:

```bash
echo banana > ~/bugged-container.txt
```

---

## 1️⃣2️⃣ Ingress with TLS

```yaml
tls:
  - hosts:
      - rocket-server.local
    secretName: rocket-tls
```

```bash
curl -k https://rocket-server.local
```

---

## 1️⃣3️⃣ Mount TLS Secret

```bash
kubectl create secret tls code-secret \
  --cert=/root/custom-cert.crt \
  --key=/root/custom-key.key \
  -n code
```

---

## 1️⃣4️⃣ RBAC Least Privilege

```yaml
verbs: ["get", "list", "watch"]
```

No secret access granted ✅

---

## 1️⃣5️⃣ Worker Node Upgrade

```bash
kubectl drain node02 --ignore-daemonsets --delete-emptydir-data
sudo kubeadm upgrade node
kubectl uncordon node02
```

---

## ✅ Final Notes

* All tasks align with **CKS exam objectives**
* Uses **least privilege**, **defense in depth**, and **secure defaults**
* Ready for **real‑world production clusters**

---

🔥 **You now have a clean, exam‑ready Kubernetes Security README**
