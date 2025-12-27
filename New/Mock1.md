1. A pod in the crypto-monitor namespace is suspected of running crypto-mining software.
Create a Falco rule that detects the execution of known mining processes like xmrig, minerd, or cpuminer.

The rule should be added to /etc/falco/falco_rules.local.yaml with below specs:

Trigger when any of these processes are executed: xmrig, minerd, cpuminer
Set the priority to CRITICAL
Output the format: MINING_ALERT: %evt.time,%container.name,%proc.name
Tag the events with [container, crypto_mining, mitre_execution]
Make sure the rule persists across Falco updates by adding it to the local rules file.


Solution
Crypto Mining Detection Solution
To set up a custom Falco rule for detecting crypto-mining processes, please follow the instructions below:

Save the following rule to /etc/falco/falco_rules.local.yaml to ensure it persists across updates:
   - rule: Detect Crypto Mining Processes
     desc: Detection of known crypto mining software execution
     condition: >
       spawned_process and container and
       (proc.name in ("xmrig", "minerd", "cpuminer"))
     output: >
       MINING_ALERT: %evt.time,%container.name,%proc.name
     priority: CRITICAL
     tags: [container, crypto_mining, mitre_execution]

Restart Falco to load the new rule using the following command:
   systemctl restart falco

To test the rule, simulate a mining process execution with this command:
   kubectl exec -n crypto-monitor deploy/suspicious-app -- sh -c "cp /bin/busybox /tmp/xmrig && /tmp/xmrig"

Check the Falco logs for alerts by running:
   journalctl -u falco -f

You should see alerts formatted as follows: MINING_ALERT: timestamp,container_name,process_name.


2. A security team has identified that pods in the threat-prevention namespace are attempting to connect to known malicious IP ranges used for command and control servers.

Create a NetworkPolicy that:

Blocks ALL egress traffic to the following malicious CIDR ranges:
192.168.100.0/24
10.0.99.0/24
Allows DNS traffic (UDP and TCP port 53) to ensure basic network functionality
Allows all other egress traffic except the blocked malicious ranges
Applies to all pods in the threat-prevention namespace
The policy should be named block-malicious-egress and should not affect ingress traffic.


Solution
Malicious Egress Blocking Solution
Create a NetworkPolicy that effectively blocks egress to known malicious IP ranges while allowing all other traffic:

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-malicious-egress
  namespace: threat-prevention
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 192.168.100.0/24
        - 10.0.99.0/24

To apply the policy, use the following command:

kubectl apply -f - <<EOF
[above yaml content]
EOF

This policy operates based on the following principles:

podSelector: {} - This selector applies to all pods within the namespace.
policyTypes: [Egress] - This configuration only impacts outgoing traffic.
The ipBlock with the except clause permits all traffic except for the specified malicious ranges.

Testing the Policy:
To test allowed traffic, use the command:

kubectl exec -n threat-prevention test-pod -- curl --connect-timeout 3 www.google.com

To test blocked traffic (this should timeout), execute:

kubectl exec -n threat-prevention test-pod -- curl --connect-timeout 3 http://192.168.100.10


3. A pod named secure-app in the security-context namespace is currently running with excessive privileges. The container is capable of escalating privileges and executing as the root user, which presents a significant security risk.

Please modify the pod configuration to achieve the following objectives:

Prevent privilege escalation
Ensure the container operates as a non-root user
Set the user ID to 101 and the group ID to 101
Utilize the nginxinc/nginx-unprivileged:alpine image
Configure the container to listen on port 8080
Ensure that the pod remains capable of serving web content on port 8080. You may need to delete and recreate the pod with the appropriate security context.


Privilege Escalation Prevention Solution
To secure the pod and prevent privilege escalation, follow these steps:

Delete the existing insecure pod:
   kubectl delete pod secure-app -n security-context --force --grace-period=0

Create a new pod with the appropriate security context:
   apiVersion: v1
   kind: Pod
   metadata:
     name: secure-app
     namespace: security-context
     labels:
       app: secure-app
   spec:
     securityContext:
       runAsNonRoot: true
       runAsUser: 101    # nginx user UID in unprivileged image
       runAsGroup: 101   # nginx group GID in unprivileged image
     containers:
     - name: app-container
       image: nginxinc/nginx-unprivileged:alpine
       securityContext:
         allowPrivilegeEscalation: false
         runAsUser: 101
         runAsGroup: 101
       ports:
       - containerPort: 8080  # Non-privileged port

Apply the secure configuration:
   kubectl apply -f - <<EOF
   [above yaml content]
   EOF

Check the security context:
   kubectl get pod secure-app -n security-context -o jsonpath='{.spec.containers[0].securityContext}' | jq

Verify the non-root user:
   kubectl exec -n security-context secure-app -- whoami

Test pod functionality:
   kubectl exec -n security-context secure-app -- wget -q -O - http://127.0.0.1:8080



4. The web-apps namespace contains a frontend application that should only be accessible from specific sources.

Create a NetworkPolicy that:

Allows ingress traffic to pods with label app: frontend ONLY from:
Pods in the same namespace with label app: backend
Any pod in the monitoring namespace
Blocks all other ingress traffic to the frontend pods
Does not affect egress traffic
The policy should be named frontend-access and should apply to the web-apps namespace.

Verify that the policy correctly allows and blocks traffic as specified.


Solution
Network Policy for Ingress Control Solution
Create a NetworkPolicy to manage ingress traffic directed to the frontend pods.

NetworkPolicy Definition:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-access
  namespace: web-apps
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
    ports:
    - protocol: TCP
      port: 80

To apply the NetworkPolicy, execute the following command:

kubectl apply -f - <<EOF
[insert the above YAML content here]
EOF

Verification Commands:
To check the NetworkPolicy, run the following commands:

kubectl get networkpolicy -n web-apps
kubectl describe networkpolicy frontend-access -n web-apps

To test connectivity, execute the following script:
cd /root
./q4-check.sh

Expected output :

Detected Pods:
  Frontend: frontend-7996796dbf-vgr49
  Backend: backend-77f8b6986-l5hpd
  External: external-client-7c487fdcf-9t2lh
  Frontend Service IP: 10.106.25.81

1. Checking NetworkPolicy:
NAME              POD-SELECTOR   AGE
frontend-access   app=frontend   31s

2. Testing ALLOWED connections:
   Backend pod -> Frontend service:
   ✓ SUCCESS - Backend can access frontend
   Monitoring namespace -> Frontend service:
   ✓ SUCCESS - Monitoring namespace can access frontend

3. Testing BLOCKED connections:
   External pod -> Frontend service:
command terminated with exit code 6
   ✓ CORRECT - External cannot access frontend
   Default namespace -> Frontend service:
   ✓ CORRECT - Default namespace cannot access frontend

4. Testing self-access:
   ✓ SUCCESS - Frontend can access itself

5. Testing egress (should not be affected):
   ✓ SUCCESS - Egress traffic works normally

=== FINAL TEST RESULTS ===
Backend -> Frontend: ✓ ALLOWED (correct)
Monitoring -> Frontend: ✓ ALLOWED (correct)
External -> Frontend: ✓ BLOCKED (correct)
Default namespace -> Frontend: ✓ BLOCKED (correct)
Self-access: ✓ ALLOWED (correct)
Egress: ✓ ALLOWED (correct)

🎉 NETWORK POLICY IS WORKING CORRECTLY! 🎉
All traffic is being allowed/blocked as specified in the policy.


5. A service account named ci-bot in the ci-cd namespace has been granted excessive permissions that could potentially enable it to create cluster-admin bindings.

Your task is to modify the RBAC configuration as follows:

Prevent the ci-bot service account from binding to any role that has "admin" or "cluster-admin" in its name.
Ensure that the service account retains its current permissions within the ci-cd namespace.
Apply this restriction broadly across the entire cluster.
To achieve this, it may be necessary to delete and recreate the existing role binding with the appropriate restrictions.


Solution
Service Account Restriction Solution
The current ClusterRole ci-bot-role permits the service account to create and modify role bindings, which poses a risk of privilege escalation.

Solution:

Remove the overly permissive ClusterRole:
kubectl delete clusterrole ci-bot-role

Create a restricted ClusterRole that omits binding creation permissions:
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ci-bot-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterrolebindings", "rolebindings"]
  verbs: ["get", "list", "watch"]
  # Note: This role does not include create, update, or patch permissions for bindings

Apply the restricted role:

kubectl apply -f - <<EOF
[above yaml content]
EOF

Verify standard operations (expected result: "yes"):

kubectl auth can-i get pods --as=system:serviceaccount:ci-cd:ci-bot
kubectl auth can-i create deployments --as=system:serviceaccount:ci-cd:ci-bot

Verify binding restrictions (expected result: "no"):

kubectl auth can-i create clusterrolebindings --as=system:serviceaccount:ci-cd:ci-bot
kubectl auth can-i create rolebindings --as=system:serviceaccount:ci-cd:ci-bot


6. A deployment named data-processor in the immutable-apps namespace is currently at risk of file system attacks due to the use of a writable root filesystem.

The deployment has the following volumes mounted for writable storage:

/tmp
/var/log
/var/cache/nginx
/var/run
Your task is to:

Configure the containers to utilize a read-only root filesystem
Ensure that the application retains the capability to write to the mounted volumes
Confirm that the nginx web server remains operational
Please modify the deployment as necessary and test the application's functionality.


Solution
Read-Only Root Filesystem Solution
Objective: Enable a read-only root filesystem while ensuring full functionality of the application.

Step 1: Patch the deployment to enable readOnlyRootFilesystem:

kubectl patch deployment data-processor -n immutable-apps -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "processor-container",
            "securityContext": {
              "readOnlyRootFilesystem": true
            }
          }
        ]
      }
    }
  }
}'

Step 2: Wait for the rollout to complete:

kubectl rollout status deployment/data-processor -n immutable-apps

Step 3: Test writable directories:

# Expected to succeed - writing to mounted volumes
kubectl exec -n immutable-apps <pod-name> -- sh -c "echo 'test' > /tmp/testfile"
kubectl exec -n immutable-apps <pod-name> -- sh -c "echo 'log' > /var/log/nginx/test.log"

# Expected to fail - writing to root filesystem  
kubectl exec -n immutable-apps <pod-name> -- sh -c "echo 'fail' > /etc/testfile"

Step 4: Test application functionality:

kubectl exec -n immutable-apps <pod-name> -- wget -q -O - http://127.0.0.1:80



7. A Dockerfile at /opt/course/image/api-server.Dockerfile is currently using a full Ubuntu base image with unnecessary packages that increase the attack surface.

Convert the Dockerfile to use a minimal distroless base image by:

Changing the base image from ubuntu:20.04 to gcr.io/distroless/base
Removing package managers (apt-get, dpkg) and shell (bash)
Copying only the necessary application binary
Setting the non-root user nonroot (UID: 65532)
Using the exec form for ENTRYPOINT
Do not add any new lines - only modify existing ones. The application binary is a Go binary that listens on port 8080.



Solution
Minimal Base Images Solution
Convert the Dockerfile from an Ubuntu base image to a distroless base image.

Original Dockerfile:

FROM ubuntu:20.04
RUN apt-get update && apt-get install -y curl wget python3 python3-pip
RUN useradd -m appuser
COPY ./app-server /app/server
RUN chmod +x /app/server
USER root
ENTRYPOINT /app/server

Secure Dockerfile:

FROM gcr.io/distroless/base
COPY ./app-server /app/server
USER nonroot:nonroot
ENTRYPOINT ["/app/server"]

Apply the changes:

cat > /opt/course/image/api-server.Dockerfile <<'EOF'
FROM gcr.io/distroless/base
COPY ./app-server /app/server
USER nonroot:nonroot
ENTRYPOINT ["/app/server"]
EOF



8. A pod definition file at /root/CKS/data-processor.yaml has excessive Linux capabilities that pose a security risk.

Secure the pod by:

Dropping ALL Linux capabilities by default
Adding back only the NET_BIND_SERVICE capability
Ensuring the pod can still bind to port 8080
Generating a kubesec scan report after fixing the pod
Once done, generate the report again and save it to /root/CKS/capabilities-report.txt


Solution
Capabilities Reduction Solution
The current pod possesses excessive Linux capabilities that must be restricted in accordance with the principle of least privilege.

In the file /root/CKS/data-processor.yaml, you will observe the following:

Original Pod Capabilities:

NET_ADMIN (network administration)
SYS_ADMIN (system administration)
NET_RAW (raw network access)
SYS_PTRACE (process debugging)
Please replace the original capabilities with the following secure configuration:

Secure Configuration:

securityContext:
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE

To apply the changes, delete and recreate the pod using the following commands:

kubectl delete pod data-processor-1 --force --grace-period=0
kubectl create -f /root/CKS/data-processor.yaml

Next, verify that the pod is running with the command:

kubectl get pod data-processor-1

Check the applied capabilities with the following command:

kubectl get pod data-processor-1 -o jsonpath='{.spec.containers[0].securityContext.capabilities}' | jq

Test the pod's functionality by executing:

kubectl exec data-processor-1 -- wget -q -O - http://127.0.0.1:8080

Finally, generate a kubesec report using:

kubesec scan /root/CKS/data-processor.yaml > /root/CKS/capabilities-report.txt



9. The financial-apps namespace contains sensitive financial applications that require strict security controls.

Configure Pod Security Admission to:

Enforce the restricted policy level on the financial-apps namespace
Use the latest version of the Pod Security Standards
Add a warning level for the baseline policy to alert on less strict pods
Label the namespace appropriately for the PSA controller
Verify that the configuration is working by attempting to create a privileged pod and observing the rejection.


Solution
Pod Security Admission Solution
Configure Pod Security Admission to enforce the restricted policy on the financial-apps namespace:

Apply PSA Labels to the Namespace:

kubectl label namespace financial-apps \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/warn-version=latest \
  --overwrite

Verify Configurations:

kubectl get namespace financial-apps --show-labels

Test PSA Enforcement:

# vi privileged-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-privileged
  namespace: financial-apps
spec:
  containers:
  - name: test
    image: busybox
    command: ["sleep", "3600"]
    securityContext:
      privileged: true
      runAsUser: 0

Apply and Observe Rejection:

kubectl apply -f privileged-pod.yaml
# Expected output: "forbidden: violates Pod Security Standard"



10. A security audit identified that containers in the secure-runtime namespace could be vulnerable to process debugging attacks. There is already a pod named secure-app running in this namespace.

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



11. A deployment named fruits in the namespace salad has three containers:

apple
banana
kiwi
One of these containers has the package curl installed. Identify which container has that package from the running containers, and create an SBOM SPDX for the container's image.

Use the tarball archive of that particular image stored under /root/ImageTarballs directory for generating the SPDX JSON. The archives have names matching their images.

Save the output in ~/bugged-fruit.spdx. Save the container name in ~/bugged-container.txt.

Note: bom and all its required dependencies are already installed.


Solution
First, check the deployments running in the salad namespace by executing the following command:

kubectl get deployments -n salad

Next, run the following command in succession to fetch the container name and save it to a text file:

kubectl exec -n salad fruits-<string> -c apple -- apk info | grep curl && echo apple > ~/bugged-container.txt

Please repeat this process for both the kiwi and banana containers.

To determine the image name used in the container, describe the pod and check the Containers section in the output using the command:

kubectl describe pod fruits-54665c68db-mcwg9 -n salad

Note that your pod name may vary.

After identifying the image name, navigate to the /root/ImageTarballs/ directory to locate the tarball associated with that image.

Finally, execute the following command to generate the SPDX JSON:

bom generate --image-archive /root/ImageTarballs/<image_name>.tar --format json --output ~/bugged-fruit.spdx

Ensure to replace <image_name> with the actual name of the tar file.


12. The secure-web namespace contains a web application deployment secure-app exposed by a service of the same name.

Create an ingress resource named secure-ingress with the following security requirements:

Route traffic for host secure-app.company.com to the backend service on path /
Enable TLS using the existing secret web-tls in the secure-web namespace
Configure the ingress to:
Force SSL redirect (HTTP to HTTPS)
Use the nginx ingress class
Add the annotation nginx.ingress.kubernetes.io/ssl-passthrough: "false"
Ensure the ingress only accepts HTTPS traffic
Verify the ingress is working with TLS termination by testing through the ingress-nginx controller's NodePort.

Note: The ingress-nginx controller uses NodePorts for external access. Use the correct HTTPS NodePort assigned to the ingress-nginx service.



Solution
Use kubectl get svc ingress-nginx-controller -n ingress-nginx to find the correct NodePort for HTTPS.

Ingress with TLS Solution
Create the ingress resource with TLS configuration and security annotations:

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  namespace: secure-web
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "false"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure-app.company.com
    secretName: web-tls
  rules:
  - host: secure-app.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-app
            port:
              number: 80

Verification
Ensure that ingress pods are healthy. Check the ingress status and TLS configuration using the following commands:

kubectl get ingress secure-ingress -n secure-web
kubectl describe ingress secure-ingress -n secure-web

Verify Annotations
To confirm that the annotations are set correctly, execute the following command:

kubectl get ingress secure-ingress -n secure-web -o jsonpath='{.metadata.annotations}'

Test the Ingress
First, find the NodePort assigned to the ingress controller for HTTPS:

NODE_PORT=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
echo "HTTPS NodePort: $NODE_PORT"

HTTPS Test: This should succeed.

curl -k -H "Host: secure-app.company.com" https://localhost:$NODE_PORT/

Expected output:

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

HTTP Test: This should redirect to HTTPS (308 redirect).

HTTP_PORT=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
curl -v -H "Host: secure-app.company.com" http://localhost:$HTTP_PORT/

Expected: Should show a 308 redirect to HTTPS.

Alternative test using the actual service:

# Get the ingress controller service details
kubectl get svc ingress-nginx-controller -n ingress-nginx

# The output will show the actual NodePorts assigned, for example:
# NAME                       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
# ingress-nginx-controller   NodePort   10.96.123.456   <none>        80:30080/TCP,443:30443/TCP   5m

# In this example, use:
# curl -k -H "Host: secure-app.company.com" https://localhost:30443/


13. The external-services namespace contains applications that need controlled access to external APIs and services.

Create a NetworkPolicy that:

Allows egress traffic ONLY to specific external services:
DNS servers (UDP port 53)
HTTPS services (TCP port 443)
A specific API endpoint at api.company.com (TCP port 8443) - assume this resolves to IP range 192.168.100.0/24
Blocks all other egress traffic from the namespace
Applies to all pods in the external-services namespace
The policy should be named restrict-egress and should use CIDR blocks for the allowed destinations.


Solution
Egress Network Policy Solution
Create a NetworkPolicy to restrict egress traffic from the external-services namespace.

NetworkPolicy Definition:

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: external-services
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # Allow DNS to any destination
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
  # Allow HTTPS to any destination
  - ports:
    - port: 443
      protocol: TCP
  # Allow specific API endpoint (api.company.com -> 192.168.100.0/24)
  - ports:
    - port: 8443
      protocol: TCP
    to:
    - ipBlock:
        cidr: 192.168.100.0/24

Apply the policy with the following command:

kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: external-services
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
  - ports:
    - port: 443
      protocol: TCP
  - ports:
    - port: 8443
      protocol: TCP
    to:
    - ipBlock:
        cidr: 192.168.100.0/24
EOF

Policy Explanation:

podSelector: {} applies to all pods in the namespace.
policyTypes: [Egress] restricts control to outgoing traffic only.
DNS Access: Allows UDP and TCP traffic on port 53 to any destination.
HTTPS Access: Allows TCP traffic on port 443 to any destination for secure web traffic.
Custom API: Allows TCP traffic on port 8443 to the specific CIDR block 192.168.100.0/24.
Implicit Deny: All other egress traffic is blocked by default.
Testing the Policy:
You can execute the following script to test the policy functionality:

cd /root
./q13_check.sh

Expected output :

=== Comprehensive NetworkPolicy Test ===
1. ✅ DNS Resolution - SUCCESS
   DNS is working
2. ✅ HTTPS (port 443) - SUCCESS
   HTTPS is working
3. ✅ HTTP (port 80) - CORRECTLY BLOCKED
command terminated with exit code 28
   HTTP correctly blocked
4. Testing FTP (port 21) - should be blocked...
   ✅ FTP correctly blocked
5. Testing SSH (port 22) - should be blocked...
   ✅ SSH correctly blocked
=== NetworkPolicy Test Complete ===
Summary: DNS and HTTPS allowed, other ports blocked - POLICY WORKING CORRECTLY!



14. The namespace encrypted has two applications, alpha and beta.

Since these applications handle critical communications, enforce strict mTLS using Istio in the encrypted namespace.

Make sure that the workloads have the istio sidecar injected.

Note: istio and istioctl have already been installed for you.


Solution
To begin, enable Istio sidecar injection in the designated namespace by executing the following command:

kubectl label namespace encrypted istio-injection=enabled --overwrite

Next, proceed to restart the deployments with the following commands:

kubectl rollout restart deployment alpha -n encrypted
kubectl rollout restart deployment beta -n encrypted

Now, apply the PeerAuthentication policy to enforce STRICT mTLS by running the command below:

cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: encrypted
spec:
  mtls:
    mode: STRICT
EOF

After that, check for sidecar injection by using this command:

kubectl describe pod -n encrypted alpha-6dc74c94df-8qwfj 

At this point, you should observe two containers running within the alpha and beta pods.

Finally, verify the peer authentication by executing:

kubectl get peerauthentication -n encrypted

You should see a resource named default.


15. In the namespace code, create a TLS secret code-secret with the following certificate and key provided:

cert: /root/custom-cert.crt
key: /root/custom-key.key
Attach that secret as a volume named secret-volume in the deployment code-server.


Solution
First, create the TLS secret by executing the following command:

kubectl create secret tls code-secret \
  --cert=/root/custom-cert.crt \
  --key=/root/custom-key.key \
  -n code

Next, edit the deployment to mount the secret:

kubectl edit deployment code-server -n code

In the spec.template.spec section, add the following configuration:

volumes:
- name: secret-volume
  secret:
    secretName: code-secret

Within the spec.template.spec.containers section, incorporate the volume mount as follows:

volumeMounts:
- name: secret-volume
  mountPath: /etc/code/tls
  readOnly: true

After making these changes, save and exit the deployment.


16. Run a CIS Benchmark scan using kube-bench and fix the etcd data directory permission issue.

Tasks:

Run kube-bench to scan the master components
Identify the etcd data directory permission violations
Requirements:

Use kube-bench with appropriate targets to find the issue
Restrict etcd directory permissions to the CIS recommended level
Apply the fixes recursively to all files and subdirectories
Verify that the fix resolves the violation
kube-bench is pre-installed. Focus on finding and fixing the etcd data directory permission issue specifically. Note: You may need to apply permissions recursively to ALL possible etcd directories and their contents.


Solution
CIS Benchmark etcd Permission Fix
Step 1: Run CIS Benchmark Scan
kube-bench run --targets master

Step 2: Identify the etcd Permission Issue
Look for output like this in the scan results:

[FAIL] 1.1.11 Ensure that the etcd data directory permissions are set to 700 or more restrictive (Automated)
[FAIL] 1.1.12 Ensure that the etcd data directory ownership is set to etcd:etcd (Automated)

Step 3: Comprehensive etcd Directory Permission Fix
# Create etcd user/group if they don't exist
groupadd etcd 2>/dev/null || true
useradd -r -g etcd -s /bin/false etcd 2>/dev/null || true

# Apply comprehensive fixes to ALL possible etcd directories
for dir in /var/lib/etcd /var/lib/etcd-data /var/lib/kubernetes/etcd /etc/kubernetes/etcd; do
    if [ -d "$dir" ]; then
        echo "Securing etcd directory: $dir"
        # Set directory permissions to 700 recursively
        find "$dir" -type d -exec chmod 700 {} \;
        # Set file permissions to 600 recursively
        find "$dir" -type f -exec chmod 600 {} \;
        # Set ownership to etcd:etcd recursively
        chown -R etcd:etcd "$dir"
    fi
done

# Specifically fix the member subdirectory which contains actual etcd data
if [ -d "/var/lib/etcd/member" ]; then
    echo "Securing etcd member directory"
    find /var/lib/etcd/member -type d -exec chmod 700 {} \;
    find /var/lib/etcd/member -type f -exec chmod 600 {} \;
    chown -R etcd:etcd /var/lib/etcd/member
fi

Step 4: Verify the Fix
# Check current permissions (should show drwx------ for all directories)
echo "=== Current etcd permissions ==="
ls -ld /var/lib/etcd
find /var/lib/etcd -type d -exec ls -ld {} \; 2>/dev/null | head -10

# Run kube-bench again to verify the fix
kube-bench run --targets master --check 1.1.11,1.1.12

# Check if both tests pass
kube-bench run --targets master 2>/dev/null | grep -E "1.1.1[12]" | grep PASS

Alternative Comprehensive Fix (if above doesn't work)
# Stop kubelet temporarily to ensure no file locks
systemctl stop kubelet 2>/dev/null || true

# Apply permissions to ALL discovered etcd directories
find / -name "*etcd*" -type d 2>/dev/null | grep -E "(var/lib|etc/kubernetes)" | while read dir; do
    if [ -d "$dir" ]; then
        echo "Securing: $dir"
        chmod -R 700 "$dir"
        chown -R etcd:etcd "$dir" 2>/dev/null || true
    fi
done

# Restart kubelet
systemctl start kubelet 2>/dev/null || true

# Wait for services to stabilize
sleep 10

# Final verification
kube-bench run --targets master --check 1.1.11,1.1.12