1. A deployment named frontend in the apparmor-demo namespace requires additional security isolation. An AppArmor profile has been created at /etc/apparmor.d/containers/restricted-frontend with the following security restrictions:

Prevents the container from writing to /etc/, /bin/, /sbin/, /usr/bin/, and /usr/sbin/ directories
Allows network access only on TCP and UDP protocols
Blocks raw socket access
Allows write access only to /tmp/ directory
Prevents capability escalation
Your tasks:

Load the AppArmor profile using apparmor_parser
Configure the deployment to use the localhost/restricted-frontend AppArmor profile using the securityContext field (not annotations)
Use the modern securityContext approach instead of deprecated annotations


Solution
AppArmor Profile Solution
Step 1: Load the AppArmor Profile
apparmor_parser /etc/apparmor.d/containers/restricted-frontend

Step 2: Verify the Profile is Loaded
apparmor_status | grep restricted-frontend

Step 3: Configure Deployment to Use AppArmor Profile (Modern Approach)
kubectl patch deployment frontend -n apparmor-demo -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "web",
            "securityContext": {
              "appArmorProfile": {
                "type": "Localhost",
                "localhostProfile": "restricted-frontend"
              }
            }
          }
        ]
      }
    }
  }
}
'

# After patching, wait for new pod with AppArmor to be created
kubectl rollout status deployment/frontend -n apparmor-demo --timeout=30s

Step 4: Verify Configuration
kubectl get deployment frontend -n apparmor-demo -o yaml | grep -A5 appArmorProfile



2. Create a RuntimeClass and configure a deployment to use it for workload isolation.

Tasks:

Create a RuntimeClass named secured using the runc handler
Create a deployment named isolated-app in the runtime-demo namespace that uses this RuntimeClass
RuntimeClass Requirements:

Name: secured
Handler: runc
Deployment Requirements:

Deployment name: isolated-app
Namespace: runtime-demo
Replicas: 1
Container name: app
Image: nginx:alpine
Container port: 80
RuntimeClass: secured
Create both resources with the exact specifications above.


Solution
RuntimeClass Solution
Step 1: Create RuntimeClass
cat << EOF | kubectl apply -f -
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: secured
handler: runc
EOF

Step 2: Create Deployment with RuntimeClass
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: isolated-app
  namespace: runtime-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: isolated-app
  template:
    metadata:
      labels:
        app: isolated-app
    spec:
      runtimeClassName: secured
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF

Step 3: Verify RuntimeClass
kubectl get runtimeclass secured

Step 4: Verify Deployment
kubectl get deployment isolated-app -n runtime-demo

Step 5: Check Pod Status
kubectl get pods -n runtime-demo -l app=isolated-app

Step 6: Test Application
kubectl exec -n runtime-demo deployment/isolated-app -- nginx -t


3. OPA Gatekeeper is installed to enforce that all pods in the gatekeeper-demo namespace have resource limits defined. A ConstraintTemplate named k8srequiredresources is already created that validates CPU and memory limits. Create a Constraint that enforces this policy.

Tasks:

Ensure Gatekeeper pods are ready in the gatekeeper-system namespace
Verify the existing ConstraintTemplate k8srequiredresources is ready
Create a Constraint named must-have-resources that targets the gatekeeper-demo namespace
Test the policy by attempting to create a pod without resource limits
Constraint Requirements:

Name: must-have-resources
Kind: K8sRequiredResources
Target namespace: gatekeeper-demo
Parameters: limits: true


Solution
OPA Gatekeeper Solution
Step 1: Wait for Gatekeeper to be Ready
kubectl get pods -n gatekeeper-system 

Step 2: Verify ConstraintTemplate is Ready
# Check that the ConstraintTemplate exists and is ready
kubectl get constrainttemplate k8srequiredresources

Step 3: Create Constraint
cat << EOF | kubectl apply -f -
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: must-have-resources
spec:
  match:
    namespaces: ["gatekeeper-demo"]
  parameters:
    limits: true
EOF

Step 4: Test the Policy
# This should fail - no resource limits
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: gatekeeper-demo
spec:
  containers:
  - name: nginx
    image: nginx:alpine
EOF

# This should succeed - with resource limits
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-with-limits
  namespace: gatekeeper-demo
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    resources:
      limits:
        cpu: "100m"
        memory: "128Mi"
EOF

Step 5: Verify Constraint Status
kubectl get k8srequiredresources must-have-resources -o yaml


4. A security audit revealed potential privilege escalation attempts in the privilege-monitor namespace.
Create a Falco rule that detects when a process attempts to gain higher privileges through setuid/setgid binaries.

The rule should be added to /etc/falco/falco_rules.local.yaml with below specs:

Trigger when any process executes setuid/setgid binaries from non-standard directories
Set the priority to CRITICAL
Output the format: PRIV_ESC_ALERT: %evt.time,%user.name,%proc.name,%proc.cmdline
Tag the events with [container, privilege_escalation, mitre_privilege_escalation]
Exclude common system directories like /bin, /usr/bin, /sbin, /usr/sbin
Make sure the rule persists across Falco updates by adding it to the local rules file.


Solution
Privilege Escalation Detection Solution
To set up a custom Falco rule for detecting privilege escalation attempts, please follow the instructions below:

Save the following rule to /etc/falco/falco_rules.local.yaml to ensure it persists across updates:
   - rule: Detect Unexpected Privilege Escalation
     desc: Detection of setuid/setgid execution from non-standard directories
     condition: >
       spawned_process and container and
       proc.aname in ("setuid", "setgid") and
       not proc.exepath startswith "/bin" and
       not proc.exepath startswith "/usr/bin" and
       not proc.exepath startswith "/sbin" and
       not proc.exepath startswith "/usr/sbin"
     output: >
       PRIV_ESC_ALERT: %evt.time,%user.name,%proc.name,%proc.cmdline
     priority: CRITICAL
     tags: [container, privilege_escalation, mitre_privilege_escalation]

Restart Falco to load the new rule using the following command:
   systemctl restart falco

To test the rule, simulate a suspicious setuid execution:
kubectl exec -n privilege-monitor deploy/suspicious-app -- sh -c "cp /usr/bin/whoami /tmp/suspicious-setuid && chmod 4755 /tmp/suspicious-setuid && /tmp/suspicious-setuid"

Check the Falco logs for alerts by running:
   journalctl -u falco -f

You should see alerts formatted as follows: PRIV_ESC_ALERT: timestamp,username,process_name,command_line.


5. Create pod named secure-pod in the security-context-demo namespace. This pod requires enhanced security configurations. Please perform the following tasks to configure it with the specified security context requirements:

Set the pod to run as a non-root user with user ID 1001.
Configure the pod to run with group ID 1001.
Prevent privilege escalation for the pod.
Drop all Linux capabilities to enhance security.
Utilize the nginxinc/nginx-unprivileged image to serve content on port 8080.
Ensure that these security settings are applied effectively to enhance the pod's security while maintaining its intended functionality.



Solution
Pod Security Context Solution
Step 1: Create the secure pod
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: security-context-demo
  labels:
    app: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    runAsGroup: 1001
  containers:
  - name: nginx
    image: nginxinc/nginx-unprivileged:alpine
    ports:
    - containerPort: 8080
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
EOF

Step 2: Verify the pod is running
kubectl get pod secure-pod -n security-context-demo

Step 3: Check security context details
kubectl describe pod secure-pod -n security-context-demo

Step 4: Test the pod functionality
kubectl exec -n security-context-demo secure-pod -- nginx -t



6. Create Security Context Constraints using Pod Security Standards for the scc-demo namespace. Implement the following restrictions:

Prevent privileged containers
Require runAsNonRoot
Block host namespaces
Require seccomp profiles
Restrict capabilities
Apply these restrictions using Pod Security Standards labels. The restricted profile will enforce most security requirements automatically.




Solution
Security Context Constraints with PSS Solution
Step 1: Apply Pod Security Standards
kubectl label namespace scc-demo \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted \
  --overwrite

Step 2: Test the Restrictions
# This should fail - violates multiple PSS rules
kubectl run test-violation -n scc-demo --image=busybox --restart=Never --command -- sleep 3600

# This should work - compliant pod
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-compliant
  namespace: scc-demo
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: test
    image: nginxinc/nginx-unprivileged:alpine
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
    ports:
    - containerPort: 8080
EOF

Step 3: Verify PSS Enforcement
# Check namespace labels
kubectl get namespace scc-demo --show-labels

# Test that privileged pods are blocked
kubectl run test-privileged -n scc-demo --image=busybox --restart=Never --privileged --command -- sleep 3600



7. Migrate pods from the deprecated PodSecurityPolicy to Pod Security Standards in the pss-migration namespace. Update the existing deployment legacy-app to comply with the restricted policy level.

The deployment currently uses the nginx-unprivileged image; however, it has a privileged security context enabled. Please update the deployment to comply with the restricted policy requirements while ensuring that it maintains functionality on port 8080.


Solution
PSS Migration Solution
Step 1: Enable Pod Security Standards
kubectl label namespace pss-migration \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  --overwrite

Step 2: Update Deployment to be PSS Compliant
kubectl patch deployment legacy-app -n pss-migration -p '
{
  "spec": {
    "template": {
      "spec": {
        "securityContext": {
          "runAsNonRoot": true,
          "seccompProfile": {
            "type": "RuntimeDefault"
          }
        },
        "containers": [
          {
            "name": "app",
            "image": "nginxinc/nginx-unprivileged:alpine",
            "securityContext": {
              "privileged": false,
              "runAsNonRoot": true,
              "runAsUser": 101,
              "allowPrivilegeEscalation": false,
              "capabilities": {
                "drop": ["ALL"]
              }
            },
            "ports": [
              {
                "containerPort": 8080
              }
            ]
          }
        ]
      }
    }
  }
}
'

Step 3: Verify Migration
kubectl get deployment legacy-app -n pss-migration
kubectl get pods -n pss-migration -l app=legacy-app
kubectl exec -n pss-migration deployment/legacy-app -- nginx -t



8. Create a NetworkPolicy in the egress-control namespace that restricts egress traffic to only allow:

DNS queries (UDP/TCP port 53) to any destination
HTTPS traffic (TCP port 443) to public IP ranges (excluding private IPs)
Block all other egress traffic
Use CIDR blocks to allow public IP ranges while blocking private IP ranges.

The policy should apply to all pods in the namespace and demonstrate egress control using IP-based rules.


Solution
Egress Network Policy Solution
Step 1: Create Egress NetworkPolicy
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: egress-control
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP

  # Allow HTTPS to public IP ranges (excluding private IPs)
  - ports:
    - port: 443
      protocol: TCP
    to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
EOF

Step 2: Test the Policy
# Create a test pod
kubectl run test-pod -n egress-control --image=appropriate/curl --command -- sleep 3600

# Wait for pod to be ready
kubectl wait --for=condition=ready pod/test-pod -n egress-control --timeout=30s

echo "=== Testing DNS (should work) ==="
kubectl exec -n egress-control test-pod -- nslookup google.com

echo "=== Testing HTTPS to allowed domain (should work) ==="
kubectl exec -n egress-control test-pod -- curl -I --connect-timeout 3 https://google.com

echo "=== Testing HTTP (should be blocked) ==="
kubectl exec -n egress-control test-pod -- curl -I --connect-timeout 3 http://example.com

Step 3: Verify Policy Configuration
kubectl get networkpolicy restrict-egress -n egress-control -o yaml



9. Enable audit logging for the Kubernetes API server to monitor security-relevant events. Please adhere to the following steps:

Create an audit policy that logs metadata for all requests.
Configure audit log rotation with the following specifications: a maximum size of 100MB, retain 10 backups, and maintain logs for a maximum of 30 days.
Set the audit log path to /var/log/kubernetes/audit.log.
Mount the necessary directories to facilitate the API server's access to policy files and enable log writing.
Please ensure that the API server continues to function normally after implementing these changes to be able to resume the exam.

In the event that the API server does not recover, a backup is stored at /root/kube-apiserver-backup.yaml. To restore the API server, execute the following commands:

cp /root/kube-apiserver-backup.yaml /etc/kubernetes/manifests/kube-apiserver.yaml
sleep 45
kubectl get nodes


Solution
API Server Audit Logging Solution
Step 1: Prepare
mkdir -p /etc/kubernetes
mkdir -p /var/log/kubernetes

cat > /etc/kubernetes/audit-policy.yaml << 'EOF'
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  omitStages:
  - RequestReceived
EOF

Step 2: Update API Server Manifest
Edit /etc/kubernetes/manifests/kube-apiserver.yaml:

Add to command array (insert among other flags):
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
- --audit-log-path=/var/log/kubernetes/audit.log
- --audit-log-maxage=30
- --audit-log-maxbackup=10
- --audit-log-maxsize=100

Add to volumeMounts section:
- mountPath: /etc/kubernetes
  name: k8s-config
  readOnly: true
- mountPath: /var/log/kubernetes
  name: audit-logs

Add to volumes section:
- hostPath:
    path: /etc/kubernetes
    type: DirectoryOrCreate
  name: k8s-config
- hostPath:
    path: /var/log/kubernetes
    type: DirectoryOrCreate
  name: audit-logs

Step 3: Verify Configuration
# Wait for API server to restart (may take 30-60 seconds)
sleep 45

# Verify API server is running
kubectl get nodes

# Check audit log is being created
ls -la /var/log/kubernetes/audit.log

# Generate some audit events
kubectl get pods
kubectl get nodes

Step 4: Troubleshooting
If API server doesn't recover:

cp /root/kube-apiserver-backup.yaml /etc/kubernetes/manifests/kube-apiserver.yaml
sleep 45
kubectl get nodes


10. A security team needs cross-namespace monitoring capabilities with restricted permissions. Create RBAC resources that allow a ServiceAccount to read pods across multiple namespaces while following the principle of least privilege.

Requirements:

Create a ServiceAccount named cluster-monitor in the monitoring-ns namespace
Create a ClusterRole named pod-reader-clusterrole that allows get, list, and watch operations on pods
Grant the ServiceAccount access only in the monitoring-ns and apparmor-demo namespaces
Label the monitoring-ns and apparmor-demo namespaces with monitoring: enabled
Use a ClusterRole with namespaced RoleBindings named cross-namespace-monitor to grant access only within the specified namespaces
Ensure the ServiceAccount has the minimum required permissions following the principle of least privilege.


Solution
Advanced RBAC with Restricted Namespace Access Solution
Step 1: Create the ServiceAccount
kubectl create serviceaccount cluster-monitor -n monitoring-ns

Step 2: Label the namespaces
kubectl label namespace monitoring-ns monitoring=enabled --overwrite
kubectl label namespace apparmor-demo monitoring=enabled --overwrite

Step 3: Create the ClusterRole with pod read permissions
cat << EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
EOF

Step 4: Create RoleBindings in allowed namespaces only
for ns in monitoring-ns apparmor-demo; do
cat << EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cross-namespace-monitor
  namespace: ${ns}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pod-reader-clusterrole
subjects:
- kind: ServiceAccount
  name: cluster-monitor
  namespace: monitoring-ns
EOF
done

Step 5: Verify permissions
# Allowed namespaces
kubectl auth can-i get pods --as=system:serviceaccount:monitoring-ns:cluster-monitor -n monitoring-ns
kubectl auth can-i get pods --as=system:serviceaccount:monitoring-ns:cluster-monitor -n apparmor-demo

# Disallowed namespace (should return no)
kubectl auth can-i get pods --as=system:serviceaccount:monitoring-ns:cluster-monitor -n default



11. Secure the Kubernetes Dashboard deployment by implementing network segmentation:

Create a NetworkPolicy named restrict-dashboard-access in the kubernetes-dashboard namespace with the following specifications:

Apply to all pods with label k8s-app: kubernetes-dashboard
Only allow ingress traffic from within the same namespace (kubernetes-dashboard)
Only allow TCP traffic on port 8443
Block all other ingress traffic
This will restrict dashboard access to only pods within the dashboard namespace.

Focus on network-level security. The dashboard is already deployed and running.


Solution
Kubernetes Dashboard Network Security Solution
Step 1: Create NetworkPolicy
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-dashboard-access
  namespace: kubernetes-dashboard
spec:
  podSelector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: kubernetes-dashboard
    ports:
    - protocol: TCP
      port: 8443
EOF

Step 2: Verify Configuration
# Check network policy
kubectl get networkpolicy restrict-dashboard-access -n kubernetes-dashboard

# Verify policy details
kubectl get networkpolicy restrict-dashboard-access -n kubernetes-dashboard -o yaml

# Ensure dashboard is still running
kubectl get pods -n kubernetes-dashboard -l k8s-app=kubernetes-dashboard


12. A private container registry requires authentication for pulling images. Configure the necessary resources to allow pods in the private-registry-ns namespace to pull images from a private registry.

Tasks:

Create a Secret of type docker-registry named my-registry-key in the private-registry-ns namespace
Use the following credentials:
Username: myuser
Password: mypassword
Email: myuser@example.com
Registry server: registry.example.com
Create a Pod named private-app that uses the registry.example.com/myapp:v1 image and references the image pull secret
This ensures secure access to private container images.

The pod should use the busybox image for testing since we don't have actual access to the private registry.



Solution
Image Pull Secrets Solution
Step 1: Create the docker-registry Secret
kubectl create secret docker-registry my-registry-key \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myuser@example.com \
  -n private-registry-ns

Step 2: Create a Pod that uses the image pull secret
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: private-app
  namespace: private-registry-ns
  labels:
    app: private-app
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
  imagePullSecrets:
  - name: my-registry-key
EOF

Step 3: Verify the configuration
# Check the secret was created
kubectl get secret my-registry-key -n private-registry-ns -o yaml

# Check the pod is using the image pull secret
kubectl get pod private-app -n private-registry-ns -o yaml | grep -A5 imagePullSecrets

# Verify the pod is running
kubectl get pod private-app -n private-registry-ns

# Check pod events for any pull issues
kubectl describe pod private-app -n private-registry-ns

Alternative: Using YAML manifest for the secret
# You can also create the secret using a YAML file:
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: my-registry-key
  namespace: private-registry-ns
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: $(echo '{"auths":{"registry.example.com":{"username":"myuser","password":"mypassword","email":"myuser@example.com","auth":"'$(echo -n 'myuser:mypassword' | base64)'"}}}' | base64 -w0)
EOF


13. A security audit identified that containers in the secure-runtime namespace could be vulnerable to process debugging attacks. There is already a pod named secure-app running in this namespace.

Your task is to:

Create a custom seccomp profile that:

Blocks the ptrace and process_vm_readv syscalls to prevent process debugging
Allows all other syscalls to maintain application functionality
Uses the SCMP_ACT_ERRNO default action for blocked syscalls
Save the profile as /var/lib/kubelet/seccomp/profiles/block-debug.json

Configure the existing secure-app pod to use this custom seccomp profile

Ensure the pod remains functional and can still serve nginx content after applying the seccomp profile

Note: You may need to restart or recreate the pod to apply the seccomp profile changes.



Solution
Syscall Filtering with Seccomp Solution
Create the custom seccomp profile:
cat > /var/lib/kubelet/seccomp/profiles/block-debug.json <<'EOF'
{
    "defaultAction": "SCMP_ACT_ALLOW",
    "syscalls": [
        {
            "names": [
                "ptrace",
                "process_vm_readv"
            ],
            "action": "SCMP_ACT_ERRNO"
        }
    ]
}
EOF

Apply the seccomp profile to the existing pod:
Since it is not possible to modify a running pod's security context directly, the pod must be recreated:

# Delete the existing pod
kubectl delete pod secure-app -n secure-runtime

# Create a new pod with the seccomp profile
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: secure-runtime
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/block-debug.json
  containers:
  - name: nginx-container
    image: nginx:alpine
    ports:
    - containerPort: 80
EOF



14. Configure resource limits for a pod to ensure predictable performance and prevent resource starvation. Create a pod named limited-pod in the resource-demo namespace with the following specifications:

Resource Requirements:

Container name: app
Image: nginx:alpine
Requests: 100m CPU and 64Mi memory
Limits: 100m CPU and 64Mi memory
Tasks:

Create the pod with both resource requests and limits defined
Ensure the pod achieves "Guaranteed" QoS class by setting equal requests and limits for both CPU and memory
Verify the pod is running successfully


Solution
Resource Limits and QoS Classes Solution
Step 1: Create the pod with equal requests and limits for Guaranteed QoS
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: limited-pod
  namespace: resource-demo
  labels:
    app: limited-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "64Mi"
EOF

Step 2: Verify the pod configuration
# Check the pod is running
kubectl get pod limited-pod -n resource-demo

# Check resource requests and limits
kubectl get pod limited-pod -n resource-demo -o jsonpath='{.spec.containers[0].resources}'

# Verify QoS class (should be Guaranteed)
kubectl get pod limited-pod -n resource-demo -o jsonpath='{.status.qosClass}'

Step 3: Detailed verification
# Get detailed pod information
kubectl describe pod limited-pod -n resource-demo

Understanding QoS Classes:
Guaranteed: All containers have memory/CPU limits EQUAL to requests
Burstable: At least one container has memory/CPU request < limit
BestEffort: No resource requests or limits specified
Key Points:
For Guaranteed QoS, BOTH CPU and memory must have equal requests and limits
Lower memory (64Mi) ensures the pod can schedule on resource-constrained nodes
Guaranteed pods are last to be killed during resource contention



15. Remove a user from the Docker group to enhance security. The user 'develop' should no longer have Docker privileges.

Use the gpasswd command to remove the user from the docker group


Solution
Remove User from Docker Group Solution
Step 1: Remove the user from docker group
sudo gpasswd -d develop docker

Step 2: Verify the user is no longer in docker group
groups develop

Step 3: Alternative verification methods
# Check /etc/group file
grep docker /etc/group

# Check user's groups
id develop

Security Benefits:
Prevents non-privileged users from running Docker commands
Reduces attack surface by limiting who can control containers
Follows principle of least privilege


16. A pod named volume-app in the volume-security namespace is configured to use a hostPath volume. Apply security measures to restrict the volume access and prevent potential host system compromise.

Security Requirements:

Mount the hostPath volume as read-only to prevent modifications to the host filesystem
Use a specific directory /var/log/app on the host (create it if needed)
Set the volume mount to read-only mode
Ensure the pod runs as non-root user (UID 1000)
Pod Specifications:

Container name: logger
Image: busybox
Command: ['sh', '-c', 'tail -f /dev/null']
Volume mount path: /app/logs
This configuration allows log reading without granting write access to the host system.

HostPath volumes can be security risks if not properly configured. Always use read-only mounts when possible.


Solution
HostPath Volume Security Solution
Step 1: Create the host directory (if not exists)
mkdir -p /var/log/app

Step 2: Create the pod with secured hostPath volume
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: volume-app
  namespace: volume-security
  labels:
    app: volume-app
spec:
  securityContext:
    runAsUser: 1000
    runAsNonRoot: true
  volumes:
  - name: log-volume
    hostPath:
      path: /var/log/app
      type: DirectoryOrCreate
  containers:
  - name: logger
    image: busybox
    command: ['sh', '-c', 'tail -f /dev/null']
    volumeMounts:
    - name: log-volume
      mountPath: /app/logs
      readOnly: true
EOF

Step 3: Verify the pod configuration
# Check the pod is running
kubectl get pod volume-app -n volume-security

# Check volume configuration
kubectl get pod volume-app -n volume-security -o jsonpath='{.spec.volumes[0]}'

# Check volume mount settings
kubectl get pod volume-app -n volume-security -o jsonpath='{.spec.containers[0].volumeMounts[0]}'

# Check security context
kubectl get pod volume-app -n volume-security -o jsonpath='{.spec.securityContext}'

Step 4: Test the volume access
# Test that we can read from the volume
kubectl exec -n volume-security volume-app -- ls -la /app/logs/

# Test that we cannot write to the volume (should fail)
kubectl exec -n volume-security volume-app -- touch /app/logs/test.txt

Step 5: Detailed verification
# Get full pod details
kubectl describe pod volume-app -n volume-security

Security Benefits:
Read-only mount: Prevents container from modifying host filesystem
Non-root user: Reduces impact if container is compromised
Specific directory: Limits access to only needed host path
DirectoryOrCreate: Ensures directory exists without excessive permissions
