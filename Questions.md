<details>
<summary>1. There are 3 deployments (nvdia , cpu, gpu) that uses the same image. And was asking to identify the pod that access the memory location /dev/mem. And scale down the deployment</summary>

Explanation
Commands / Steps

Create a custom Falco rule file rule.yaml:

```
- rule: read write below /dev/mem
  desc: An attempt to read or write to /dev/mem directory
  condition: >
    ((evt.is_open_read=true or evt.is_open_write=true) and fd.name contains /dev/mem)
  output: "Process %proc.name accessed /dev/mem (command=%proc.cmdline user=%user.name container=%container.id image=%container.image.repository pod_name=%k8s.pod.name namespace=%k8s.ns.name)"
  priority: WARNING
  tags: [security]
```

Run Falco manually with the custom rule file:

```
falco -r rule.yaml  | grep -i 'dev/mem'
```

Check Falco output/logs for alerts:

# Example alert:

23:15:42.567890: Warning Process evil-binary accessed /dev/mem (command=evil-binary user=root container=abc123 image=malicious/image pod_name=mem-hacker-7d89d9c7f8-xyz namespace=default)

→ Pod: mem-hacker-7d89d9c7f8-xyz
→ Namespace: default
→ Deployment: mem-hacker

Identify the Deployment that owns the container if only container ID is available, map it back to Pod/Deployment:

```
# Using crictl to find the container
crictl ps -id abc123
crictl pods -id <pod_id>
kubectl get pod -A | grep mem-hacker
```

Scale the Deployment replicas to 0:

```

kubectl scale deployment mem-hacker --replicas=0 -n default
```

Verify scaling:

```
kubectl get deploy mem-hacker -n default
```

Expected output:

NAME READY UP-TO-DATE AVAILABLE AGE
mem-hacker 0/0 0 0 1m

Explanation of Falco Rule:

Falco rules use system call fields to filter events. For /dev/mem monitoring:

evt.is_open_read=true → matches syscalls like open, openat, openat2, open_by_handle_at with read flag.

evt.is_open_write=true → same syscalls but with write flag.

Combined condition detects attempts to open /dev/mem with read or write permissions.

👉 If you also want to detect actual read/write operations:

((evt.is_open_read=true or evt.is_open_write=true) or evt.type in (read, write)) and fd.name contains /dev/mem
👉 If you want to monitor only open attempts:

(evt.is_open_read=true or evt.is_open_write=true) and fd.name contains /dev/mem

⚠️ Common mistake:

Writing something like:

evt.is_open_read=true and evt.type=read
will never match, because evt.is_open_read exists only for open-related syscalls, not for read/write syscalls.

⚠️ Note:

Created a Falco rule to detect /dev/mem access.

Ran Falco and caught the malicious Pod.

Identified the Deployment (mem-hacker).

Scaled replicas to 0 to stop the attack.

open_by_handle_at: this is a rare syscall used in advanced file APIs. Falco includes it so that rules reliably cover all ways a process might open a file.

</details>

<details>
<summary>2. A pod with 3 containers. 3 containers are using the same Image but different tags. So I need to get the image that has a specific version of libcrypto version. And create a spdx sbom with a tool called “bom”.</summary>

Examine each container's image to check the version of libcrypto. You can do this by pulling the image locally and inspecting it or by using the docker run command to inspect the installed packages

```
k get pods -n <namespace> -o yaml | grep image
for i in <first_image> <second_image> <third_image>; do bom generate --image $i | grep libcrypto; done
```

</details>

<details>
<summary>3. Remove a Linux user called “developer” from the "docker" group. And also deny tcp traffic from docker daemon. How to do this?</summary>
```
sudo gpasswd -d developer docker 
```
 To deny tcp traffic from docker daemon you would edit the /etc/docker/daemon.json file and remove tcp host entries. Leave unix:///var/run/docker.sock in there as that allows socket communication. Then restart docker daemon and check status. 
 ```
 sudo systemctl restart docker
```
</details>

<details>
<summary>4. There is an incomplete configuration in '/etc/kubernetes/image-config' and a external image scanner server in this address: 'https://image-cks-webhook:1234'First, reconfigure the API server to enable related plugins to support the provided AdmissionConfiguration. Second, reconfigure ImagePolicyWebhook to reject images if the backend is unavailable.</summary>

Answer:

```
1-going to configuration folder quickly: cd /etc/kubernetes/image-config
2- find image-config-policy file with yaml extension and quickly change this line to false:defaultAllow: true ==> False
3- find kubeconfig in the same folder and add the server address on it
4- copy the kube-apiserver to prevent any issue then add the following:on '--enable-admission-plugins' line add: ImagePolicyWebhook and add this flag:- -- admission-control-config-file= <address of image policy file>the volume and mount address are already configures and no need to addsave and exit and wait 1 to 2 mins for kube-apiserver to back in serviceor also you can check bysudo watch crictl ps
```

</details>

<details>
<summary>
Identify a service running on port 389, list all its open files, and remove the binary:
Find the process ID (PID) of the service listening on port 389.
Store the list of all open files of the process in /candidate/13/files.txt.
Locate the executable binary of the process and delete it.</summary>

✅ Step 1: Identify the service running on port 389

Port 389 is commonly LDAP, but don’t assume — find the PID.

```
sudo ss -lntp | grep :389

OR

sudo lsof -i :389
```

Example output:

LISTEN 0 128 \*:389 users:(("slapd",pid=1234,fd=7))

📌 PID = 1234

✅ Step 2: List all open files of the process

Store open files:

```
sudo lsof -p 1234 > /candidate/13/files.txt
```

✔️ Requirement satisfied
✔️ Do NOT filter — store all open files

✅ Step 3: Locate the executable binary of the process

```
readlink -f /proc/1234/exe

```

Example output:

/usr/sbin/slapd

📌 This is the actual running binary

✅ Step 4: Remove (delete) the binary

```
sudo rm -f /usr/sbin/slapd
```

✔️ Binary removed
✔️ Process will terminate or break immediately
✔️ Matches “remove the binary” exactly

</details>

https://freedium-mirror.cfd/https://medium.com/@arunmrp90/mastering-the-2025-certified-kubernetes-security-specialist-cks-exam-16-realistic-scenarios-c69a5e951a6b

16 questions and I will share some

Falco: Given: 3 pods nvidia, cpu, ollama are accessing /dev/mem and we need to scale down replica to zero for those pod

Istio: apply mtls sidecar https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/#lock-down-to-mutual-tls-by-namespace https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/#deploying-an-app

Ingress with tls: Given a secret tls and create an Ingress tls. Also redirect http request to https (should use ingressClassName: nginx with the annotation ssl-redirect https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/

Docker daemon secure: Require 1: remove user “develop” from group docker Require 2: Then chown root:root of Docker sock /var/run/docker.sock Require 3: Docker daemon change to unix from tcp ( /lib/systemd/system/docker.service)

Bom: There’s a pod alpine with 3 containers using image alpine with different version 3.20.0, 3.19.6 and 3.16.1.
Require 1: Check with container has libcrypto3 version x.y.z and change the deployment yaml file remove that container, then redeploy
Require 2: Generate a SPDX report write to file.

Static file analysic: Given: A long Dockerfile and a deploy yaml file.
Require 1: change one line only and DO NOT add/remove any lines, dont build the image (it mentioned in the question) → Change USER root to USER couchdb.
Require 2: change one line only and DO NOT add/remove any lines → Change readOnlyRootFilesystem from false to true.

Secret TLS: Given: A deployment yaml file, a cert file and a key file Require: Create a tls secret in a namespace → apply it to the deployment yaml file and apply it.

Projected volume and SA: Given: an SA and a deployment yaml file.

Require 1: Change the SA automountServiceAccountToken to false

Require 2: Using projected volume for the deployment under /var/run/secrets/kubernestes.io/serviceaccount/token

- Kube-bench Fix 3 small issues
- Auditing
- ImagePolicyWebhook
- Network policies: create 2 policies (no CiliunmNetworkPolicies)
- PSS: Try to fix the given deployment yaml file to make the pod running. Check replicaset event.
- Kube-apiserver: change the anonymous-auth flag and delete a clusterrolebinding system:anonymous
- Seccomp profile apply
- Upgrade worker node from 1.33.0 to 1.33.1
