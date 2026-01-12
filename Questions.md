<details>
<summary>1. There are 3 deployments (nvdia , cpu, gpu) that uses the same image. And was asking to identify the pod that access the memory location /dev/mem. And scale down the deployment</summary>

Explanation
Commands / Steps



Create a custom Falco rule file rule.yaml:



- rule: read write below /dev/mem
  desc: An attempt to read or write to /dev/mem directory
  condition: >
    ((evt.is_open_read=true or evt.is_open_write=true) and fd.name contains /dev/mem)
  output: "Process %proc.name accessed /dev/mem (command=%proc.cmdline user=%user.name container=%container.id image=%container.image.repository pod_name=%k8s.pod.name namespace=%k8s.ns.name)"
  priority: WARNING
  tags: [security]


Run Falco manually with the custom rule file:



falco -r rule.yaml  | grep -i 'dev/mem'


Check Falco output/logs for alerts:



# Example alert:
23:15:42.567890: Warning Process evil-binary accessed /dev/mem (command=evil-binary user=root container=abc123 image=malicious/image pod_name=mem-hacker-7d89d9c7f8-xyz namespace=default)


→ Pod: mem-hacker-7d89d9c7f8-xyz
→ Namespace: default
→ Deployment: mem-hacker


Identify the Deployment that owns the container if only container ID is available, map it back to Pod/Deployment:



# Using crictl to find the container
crictl ps -id abc123
crictl pods -id <pod_id>
kubectl get pod -A | grep mem-hacker


Scale the Deployment replicas to 0:



kubectl scale deployment mem-hacker --replicas=0 -n default


Verify scaling:



kubectl get deploy mem-hacker -n default


Expected output:



NAME         READY   UP-TO-DATE   AVAILABLE   AGE
mem-hacker   0/0     0            0           1m


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

2. Apod with 3 containers. 3 containers are using the same Image but different tags. So I need to get the image that has a specific version of libcrypto version. And create a spdx sbom with a tool called “bom”.
3. Remove a Linux user called “developer” from the "docker" group. And also deny tcp traffic from docker daemon. How to do this?


https://freedium-mirror.cfd/https://medium.com/@arunmrp90/mastering-the-2025-certified-kubernetes-security-specialist-cks-exam-16-realistic-scenarios-c69a5e951a6b