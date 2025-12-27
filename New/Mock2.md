1. Harden the kubelet configuration on ssh cluster2-controlplane .

Tasks:

Modify the kubelet configuration to disable anonymous authentication.
Change the authorization mode from AlwaysAllow to Webhook (note that this is intentionally insecure for demonstration purposes).
Utilize the admin kubeconfig located at /root/custom-config/admin.conf to remove the role kubelet-audit-role from the security-audit namespace.
Ensure all security measures are properly implemented.

The kubelet configuration file is located at /var/lib/kubelet/config.yaml. Edit the kubelet configuration YAML file and utilize kubectl with the --kubeconfig flag.

Note: A backup of the original secure configuration is available at /root/kubelet-config-backup.yaml for reference. Ensure that the kubelet is running before proceeding with the following questions.


Solution
Kubelet Hardening and RBAC Cleanup
Step 1: Secure Kubelet Authentication
SSH into the control plane of the cluster:
   ssh cluster2-controlplane

Edit the kubelet configuration file:
   sudo vi /var/lib/kubelet/config.yaml

Locate the authentication and authorization sections, then modify them as follows:
   authentication:
     anonymous:
       enabled: false  # Change from true to false

   authorization:
     mode: Webhook     # Change from AlwaysAllow to Webhook

Restart the kubelet service to apply the changes:
   sudo systemctl daemon-reload
   sudo systemctl restart kubelet

Step 2: Remove Unnecessary Role
Delete the role using the specified kubeconfig:
   kubectl delete role kubelet-audit-role -n security-audit --kubeconfig=/root/custom-config/admin.conf

Step 3: Verification
Verify that anonymous access is blocked by executing the following command:
   curl -k https://localhost:10250/metrics

Confirm the deletion of the role:
   kubectl get role kubelet-audit-role -n security-audit --kubeconfig=/root/custom-config/admin.conf

Restoration from Backup
If you need to restore the configuration from a backup, run the following commands:

sudo cp /root/kubelet-config-backup.yaml /var/lib/kubelet/config.yaml
sudo systemctl daemon-reload
sudo systemctl restart kubelet



2. Please exit from cluster2-controlplane and ensure that you are in cluster1-controlplane for the subsequent question.

Three binary packages have been placed in /root/binary-verification/, with only one being authentic.

Tasks:

Review the official checksum in official-checksum.sha256.
Verify each binary package (v1.tar, v2.tar, v3.tar) against the official checksum.
Determine which binary package is authentic.
Complete the verification report at /root/binary-verification-report.txt with the following details:
The official checksum value
Verification results for each binary (AUTHENTIC or TAMPERED, along with the actual checksum)
Identification of the authentic binary
Binary packages to verify:

v1.tar
v2.tar
v3.tar


Solution
Binary Verification Solution
Step 0: Back to cluster1-controlplane
cluster2-controlplane ~ ➜  exit
logout
Connection to cluster2-controlplane closed.
cluster1-controlplane ~ ➜  

Step 1: Check the Official Checksum
cd /root/binary-verification
cat official-checksum.sha256

Step 2: Calculate Checksums for Each Binary
# Calculate checksum for v1.tar
sha256sum v1.tar

# Calculate checksum for v2.tar
sha256sum v2.tar

# Calculate checksum for v3.tar
sha256sum v3.tar

Step 3: Compare and Complete the Report
Edit the report template:

vi /root/binary-verification-report.txt

Fill in the following details:

OFFICIAL CHECKSUM: 8739dd0797f162c7d8b87c4d3213d074f91d9cbf0bdf4cba73afa0b5becb075c
v1.tar: 9a0b036a9b0885a7521bc63c65a7baf2ce63c52ca1c86c56ff101e07762be334
v2.tar: 8739dd0797f162c7d8b87c4d3213d074f91d9cbf0bdf4cba73afa0b5becb075c
v3.tar: 9431b841b7d5201ea6687ebbba02f78ed3854613bd307fca32d871c0004f7469
AUTHENTIC BINARY: v2.tar
Example Completed Report:
Kubernetes Binary Verification Report
====================================
Fri Oct  3 07:26:17 AM EDT 2025

OFFICIAL CHECKSUM: 8739dd0797f162c7d8b87c4d3213d074f91d9cbf0bdf4cba73afa0b5becb075c  correct-binary.tar

VERIFICATION RESULTS:
v1.tar: 9a0b036a9b0885a7521bc63c65a7baf2ce63c52ca1c86c56ff101e07762be334
v2.tar: 8739dd0797f162c7d8b87c4d3213d074f91d9cbf0bdf4cba73afa0b5becb075c
v3.tar: 9431b841b7d5201ea6687ebbba02f78ed3854613bd307fca32d871c0004f7469

AUTHENTIC BINARY: v2.tar


3. The service-account-audit namespace contains an overprivileged service account named overprivileged-sa and an insecure deployment called insecure-app.

Tasks:

Create a new secure service account named restricted-sa, ensuring that automatic token mounting is disabled.
Create a minimal RBAC role named restricted-role that permits only get and list operations on pods.
Bind the restricted-role to the restricted-sa service account.
Modify the existing deployment insecure-app to utilize the restricted-sa service account, incorporating a security context.
Requirements for the deployment modification:

Use the restricted-sa service account.

Disable automatic token mounting.

Configure the security context to run as a non-root user (UID 101).

Disable privilege escalation.

Drop all Linux capabilities.

Do not modify the existing overprivileged-sa.


Solution
Service Account Security Hardening Solution
Step 1: Create the Secure Service Account
Begin by creating and applying the restricted-sa.yaml configuration file:

apiVersion: v1
kind: ServiceAccount
metadata:
  name: restricted-sa
  namespace: service-account-audit
automountServiceAccountToken: false

Step 2: Create the Minimal Role
Next, create and apply the restricted-role.yaml configuration file:

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: restricted-role
  namespace: service-account-audit
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

Step 3: Bind the Role to the Service Account
Now, create and apply the restricted-binding.yaml configuration file:

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: restricted-binding
  namespace: service-account-audit
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: restricted-role
subjects:
- kind: ServiceAccount
  name: restricted-sa
  namespace: service-account-audit

Step 4: Modify the Existing Deployment
kubectl edit deployment insecure-app -n service-account-audit

Ensure to add the following sections to the deployment specification:

spec:
  template:
    spec:
      serviceAccountName: restricted-sa
      automountServiceAccountToken: false
      containers:
      - name: app
        securityContext:
          runAsNonRoot: true
          runAsUser: 101  # nginx user in unprivileged image
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL

Alternative one-command approach:

kubectl patch deployment insecure-app -n service-account-audit --type='strategic' --patch='
spec:
  template:
    spec:
      serviceAccountName: restricted-sa
      automountServiceAccountToken: false
      containers:
      - name: app
        image: nginxinc/nginx-unprivileged:1.25-alpine
        securityContext:
          runAsNonRoot: true
          runAsUser: 101
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
'

Verification Steps
To verify your changes, execute the following commands:

# Check the service account
kubectl get sa restricted-sa -n service-account-audit -o yaml

# Check the deployment is running
kubectl get deployment insecure-app -n service-account-audit

# Check pods are running (not crashlooping)
kubectl get pods -n service-account-audit -l app=insecure-app

# Check the deployment security settings
kubectl get deployment insecure-app -n service-account-audit -o yaml | grep -A10 "securityContext"

# Check permissions
kubectl auth can-i create pods --as=system:serviceaccount:service-account-audit:restricted-sa

# Verify non-root execution
kubectl get pod -n service-account-audit -l app=insecure-app -o jsonpath='{.items[0].spec.containers[0].securityContext.runAsUser}'


4. Secure pods in the node-security namespace by preventing access to the node metadata service (169.254.169.254).

Tasks:

Create a NetworkPolicy named block-metadata-access that blocks all egress traffic to the metadata service IP
Ensure the policy allows all other egress traffic
Test that metadata access is blocked while maintaining DNS functionality
Note: A test deployment metadata-test-pod is already running in the namespace for validation.


Solution
Node Metadata Protection Solution
Step 1: Create the NetworkPolicy
Create and apply a NetworkPolicy that blocks metadata access:

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-metadata-access
  namespace: node-security
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32

Apply it:

kubectl apply -f - <<EOF
[above yaml content]
EOF

Step 2: Verify the NetworkPolicy
# Check the policy was created
kubectl get networkpolicy -n node-security

# Verify it blocks the correct IP
kubectl get networkpolicy block-metadata-access -n node-security -o yaml

Step 3: Test the Protection
# Test metadata access is blocked
kubectl exec -n node-security deploy/metadata-test-pod -- sh -c \
  "curl -s --connect-timeout 2 http://169.254.169.254/latest/meta-data/ && echo 'ACCESSIBLE' || echo 'BLOCKED'"

# Test DNS still works
kubectl exec -n node-security deploy/metadata-test-pod -- sh -c \
  "nslookup kubernetes.default.svc.cluster.local && echo 'DNS WORKS' || echo 'DNS BROKEN'"



5. You are setting up a new Kubernetes cluster and need to secure Docker as part of the cluster setup.

Ensure that docker runs under the "root" group and that no external TCP connections are allowed to the docker daemon.

Ensure the configuration is persistent across restarts.

Backup of the original Docker configuration is available at /etc/docker/daemon.json.backup. Docker service must remain running for container operations in subsequent questions.


Solution
Change the ownership of the docker file:

sudo chown root:root /var/run/docker.sock

Then add --group=root to the ExecStart of docker systemd file:

sudo systemctl edit docker

[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --group=root

Then reload the docker daemon:

sudo systemctl daemon-reexec 
sudo systemctl daemon-reload 
sudo systemctl restart docker

To remove the TCP external connections, modify the /etc/docker/daemon.json to remove the tcp section so that the file looks like this:

{
  "hosts": ["unix:///var/run/docker.sock"]
}

Then restart docker again.


6. SSH into the cluster2-controlplane to address the following tasks:

Configure the kube-controller-manager to use the --use-service-account-credentials flag.
Set the --terminated-pod-gc-threshold to 50.
The admin kubeconfig file for this cluster is located at:

/root/controller-config/admin.conf

Additionally, please utilize this kubeconfig file to delete the cluster role named 'legacy-cluster-role'.

The backup of the original kube-controller-manager configuration is located at /tmp/kube-controller-manager-bak.yaml. It is essential to ensure that the Controller Manager remains running for the container operations in the upcoming questions.


Solution
Secure Controller Manager and ClusterRole Cleanup
Objective
Configure kube-controller-manager security and clean up potentially dangerous cluster roles.

Step-by-Step Instructions
1. SSH to the cluster2-controlplane
ssh cluster2-controlplane

2. Configure Controller Manager Security
Open the controller manager manifest by executing the following command:

sudo nano /etc/kubernetes/manifests/kube-controller-manager.yaml

Locate the command section and add or verify the following flags:

spec:
  containers:
  - command:
    - kube-controller-manager
    - --use-service-account-credentials=true
    - --terminated-pod-gc-threshold=50
    # ... other existing flags

Save the changes and exit the file. The kubelet will automatically restart the controller manager.

3. Verify Controller Manager Restart
Ensure that the controller manager pod is currently running by using the command:

kubectl get pods -n kube-system | grep controller-manager

4. Delete the Dangerous ClusterRole
Utilize the admin kubeconfig to remove the overly permissive cluster role by executing:

kubectl delete clusterrole legacy-cluster-role --kubeconfig=/root/controller-config/admin.conf

Verify that the cluster role has been successfully deleted:

kubectl get clusterrole legacy-cluster-role --kubeconfig=/root/controller-config/admin.conf

This command should return "Error from server (NotFound)".

5. Security Verification
Check that the controller manager is utilizing the new security settings by running:

kubectl get pods -n kube-system -l component=kube-controller-manager -o yaml --kubeconfig=/root/controller-config/admin.conf | grep -A20 "command"


7. Please exit from cluster2-controlplane and ensure that you are in cluster1-controlplane for the subsequent question.

Maria is a database administrator who requires varying levels of access across multiple database namespaces.

In the production-db namespace, she must have:

Full access to all resources (get, list, create, update, delete, patch)
In the staging-db namespace, her access should be:

Read-only for pods and services
No access to secrets or configmaps
In the backup-db' namespace, she should be granted:

Get and list permissions for persistentvolumeclaims only
No access to any other resources
The current RBAC configuration is overly permissive. Please update the permissions to align with the principle of least privilege while retaining the same resource names.


Solution
Database Administrator RBAC Restriction
Objective
Implement a least privilege Role-Based Access Control (RBAC) strategy for Database Administrator Maria across multiple namespaces, ensuring varying access levels.

Step-by-Step Solution
1. Review the Current RBAC Configuration
Begin by examining the existing roles and role bindings:

# Retrieve roles in all database namespaces
kubectl get role -n production-db
kubectl get role -n staging-db  
kubectl get role -n backup-db

# Retrieve role bindings
kubectl get rolebinding -n production-db
kubectl get rolebinding -n staging-db
kubectl get rolebinding -n backup-db

# View current role permissions
kubectl get role db-admin-role -n production-db -o yaml
kubectl get role db-admin-role -n staging-db -o yaml
kubectl get role db-admin-role -n backup-db -o yaml

2. Update the Production Database Role
The production role must permit full access. Maintain its current state or ensure it is configured as follows:

kubectl apply -n production-db -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: db-admin-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
EOF

3. Update the Staging Database Role
The staging environment should provide read-only access to pods and services, excluding secrets and config maps:

kubectl apply -n staging-db -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: db-admin-role
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
EOF

4. Update the Backup Database Role
The backup environment should only allow read access to persistent volume claims:

kubectl apply -n backup-db -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: db-admin-role
rules:
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list"]
EOF

5. Verify the Role Changes
Test Maria's permissions in each namespace:

# Production - should have full access
kubectl auth can-i create pods --as=maria -n production-db
kubectl auth can-i delete secrets --as=maria -n production-db

# Staging - should have read-only access to pods/services, no secrets
kubectl auth can-i get pods --as=maria -n staging-db
kubectl auth can-i create pods --as=maria -n staging-db
kubectl auth can-i get secrets --as=maria -n staging-db

# Backup - should only have read access to PVC
kubectl auth can-i get persistentvolumeclaims --as=maria -n backup-db
kubectl auth can-i create persistentvolumeclaims --as=maria -n backup-db
kubectl auth can-i get pods --as=maria -n backup-db


8. A deployment named log-aggregator in the secure-logging namespace is currently vulnerable to runtime tampering because its container uses a writable root filesystem. This could allow an attacker to modify application binaries or configuration files if the container is compromised.

The application requires write access only to the following specific directories for its normal operation:

/var/log/application (for application logs)
/tmp (for temporary processing files)
/var/cache (for cached data)
Your objective is to:

Harden the deployment by configuring its container to use a read-only root filesystem
Preserve application functionality by ensuring it can still write to the required directories listed above
Verify the solution by confirming the pod runs successfully and the logging service remains operational
Make the necessary changes to the deployment and validate that the application works correctly under the new security constraints.



Solution
Read-Only Root Filesystem Solution for Logging Service
Objective: Enable a read-only root filesystem while ensuring the logging service maintains full functionality.

Step 1: Patch the deployment to enable readOnlyRootFilesystem:

kubectl patch deployment log-aggregator -n secure-logging -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "log-container",
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

kubectl rollout status deployment/log-aggregator -n secure-logging

Step 3: Test writable directories:

# Expected to succeed - writing to mounted volumes
kubectl exec -n secure-logging <pod-name> -- sh -c "echo 'test' > /tmp/testfile"
kubectl exec -n secure-logging <pod-name> -- sh -c "echo 'log' > /var/log/application/test.log"
kubectl exec -n secure-logging <pod-name> -- sh -c "echo 'cache' > /var/cache/test.cache"

# Expected to fail - writing to root filesystem  
kubectl exec -n secure-logging <pod-name> -- sh -c "echo 'fail' > /etc/testfile"

Step 4: Test application functionality:

kubectl exec -n secure-logging <pod-name> -- sh -c "ls -la /var/log/application/"



9. A deployment named api-server is running in the namespace production. The deployment pods are failing to start.

Identify the issue causing the pods to fail, and then fix the deployment.



Solution
Solution: Troubleshooting Deployment Failure
Step-by-Step Troubleshooting
Step 1: Check Current Deployment Status
kubectl get deployment api-server -n production
kubectl get pods -n production -l app=api-server

Observation: No pods have been created, and the deployment shows 0/2 replicas.

Step 2: Check Deployment Events and Conditions
kubectl describe deployment api-server -n production

Critical Finding: Deployment events indicate Pod Security violations that are preventing pod creation.

Step 3: Analyze the Exact Security Violations
To identify the specific violations, review the deployment YAML:

kubectl get deployment api-server -n production -o yaml

Violations Identified:

privileged: true
runAsNonRoot: false
runAsUser: 0 and runAsGroup: 0 (root user)
Capabilities added: NET_ADMIN, SYS_TIME
Missing allowPrivilegeEscalation: false
Missing seccompProfile
Step 4: Apply the Complete Fix
The previous patch contained an issue. It is necessary to completely remove the capabilities.add section:

# First, check the current replica set
kubectl get replicaset -n production

# Apply a comprehensive fix that addresses all security violations
kubectl patch deployment api-server -n production --type='json' -p='
[
  {
    "op": "replace",
    "path": "/spec/template/spec/securityContext",
    "value": {
      "runAsNonRoot": true,
      "runAsUser": 101,
      "runAsGroup": 101,
      "seccompProfile": {
        "type": "RuntimeDefault"
      }
    }
  },
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/securityContext",
    "value": {
      "runAsNonRoot": true,
      "runAsUser": 101,
      "privileged": false,
      "allowPrivilegeEscalation": false,
      "capabilities": {
        "drop": ["ALL"]
      }
    }
  }
]
'

Step 5: Monitor the Rollout
# Observe the rollout progress
kubectl rollout status deployment/api-server -n production --watch

# Check if new pods are being created
kubectl get pods -n production -w

Step 6: Verify the Fix Worked
# Check deployment status
kubectl get deployment api-server -n production

# Check pod status
kubectl get pods -n production -l app=api-server

# Verify the security context is accurate
kubectl get pod -n production -l app=api-server -o jsonpath='{.items[0].spec.containers[0].securityContext}'

# Review pod logs to ensure nginx is running
kubectl logs -n production -l app=api-server --tail=10



10. In the galaxy namespace, a deployment starship-api is exposed by a service of the same name.

Create an ingress resource named starship-ingress to route incoming traffic to the workload on path /api.

Use the hostname starship.company.com for the Ingress rules.

Utilize the TLS certificate stored in the secret starship-tls in the galaxy namespace to enable TLS traffic on that ingress resource.

The ingress should redirect all HTTP traffic to HTTPS.



Solution
The following steps outline the correct procedure:

Verify the existence of the "galaxy" namespace:
   cluster1-controlplane ~ ➜   k get ns galaxy
   NAME     STATUS   AGE
   galaxy   Active   8m35s

Check the deployments within the "galaxy" namespace:
   cluster1-controlplane ~ ➜   k get deployments.apps -n galaxy
   NAME           READY   UP-TO-DATE   AVAILABLE   AGE
   starship-api   2/2     2            2           8m38s

Describe the deployment of "starship-api":
   cluster1-controlplane ~ ➜  k describe deployment starship-api -n galaxy
   Name:                   starship-api
   Namespace:              galaxy
   CreationTimestamp:      Mon, 06 Oct 2025 02:39:47 -0400
   Labels:                 <none>
   Annotations:            deployment.kubernetes.io/revision: 1
   Selector:               app=starship-api
   Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
   StrategyType:           RollingUpdate
   MinReadySeconds:        0
   RollingUpdateStrategy:  25% max unavailable, 25% max surge
   Pod Template:
     Labels:  app=starship-api
     Containers:
      webapp:
       Image:         nginx:alpine
       Port:          80/TCP
       Host Port:     0/TCP
       Environment:   <none>
       Mounts:        <none>
     Volumes:         <none>
     Node-Selectors:  <none>
     Tolerations:     <none>
   Conditions:
     Type           Status  Reason
     ----           ------  ------
     Available      True    MinimumReplicasAvailable
     Progressing    True    NewReplicaSetAvailable
   OldReplicaSets:  <none>
   NewReplicaSet:   starship-api-55b56dc767 (2/2 replicas created)
   Events:
     Type    Reason             Age    From                   Message
     ----    ------             ----   ----                   -------
     Normal  ScalingReplicaSet  8m44s  deployment-controller  Scaled up replica set starship-api-55b56dc767 from 0 to 2

4.Create the Ingress :

   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: starship-ingress
     namespace: galaxy
     annotations:
       nginx.ingress.kubernetes.io/ssl-redirect: "true"
       nginx.ingress.kubernetes.io/rewrite-target: /
   spec:
     ingressClassName: nginx
     tls:
       - hosts:
           - starship.company.com
         secretName: starship-tls
     rules:
       - host: starship.company.com
         http:
           paths:
             - path: /api
               pathType: Prefix
               backend:
                 service:
                   name: starship-api
                   port:
                     number: 80

Check the services in the "ingress-nginx" namespace:
   cluster1-controlplane ~ ➜  kubectl get svc -n ingress-nginx
   NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
   ingress-nginx-controller             LoadBalancer   172.20.96.68     <pending>     80:32746/TCP,443:31685/TCP   9m6s
   ingress-nginx-controller-admission   ClusterIP      172.20.232.147   <none>        443/TCP                      9m6s

Update the /etc/hosts file with the following line:
   cluster1-controlplane ~ ➜  echo "172.20.96.68 starship.company.com" | sudo tee -a /etc/hosts
   172.20.96.68 starship.company.com

Finally, test the Ingress by executing the following command:
   cluster1-controlplane ~ ➜  curl -k https://starship.company.com/api

The expected output should display a welcome page from nginx, indicating that the nginx web server is successfully installed and operational.



11. In the namespace vault, you need to implement secure secret management for the deployment secure-app.

Tasks:
Create a TLS secret named app-tls using:

Certificate: /root/app-cert.crt
Private key: /root/app-key.key
Mount the secret securely in the deployment secure-app:

Volume name: tls-secret-volume
Mount path: /etc/app/tls
Set file permissions to 0400 (read-only by owner)
Create a ServiceAccount named secure-sa for the deployment

Create RBAC resources to:

Allow the ServiceAccount to only get and list secrets in the vault namespace
Deny access to secrets in other namespaces
Apply the same label, other than the hash one, used on the secure-app deployment's pods to these resources
Update the deployment to use the ServiceAccount and security context:

Run as non-root user (UID 1001)
Set allowPrivilegeEscalation: false



Solution
Step-by-Step Solution

Create the TLS Secret
   # Create the TLS secret using the provided certificate and key
   kubectl create secret tls app-tls \
     --cert=/root/app-cert.crt \
     --key=/root/app-key.key \
     -n vault

Create the ServiceAccount
   # Create a dedicated ServiceAccount for the deployment
   kubectl create serviceaccount secure-sa -n vault

Create RBAC Resources
Create Role for secret access in the vault namespace:
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-access
  namespace: vault
  labels:
    app: secure-app
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
EOF

Create RoleBinding to link ServiceAccount to Role:
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-access-binding
  namespace: vault
  labels:
    app: secure-app
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: secret-access
subjects:
- kind: ServiceAccount
  name: secure-sa
  namespace: vault
EOF

Update the Deployment

Patch the deployment to include all security requirements:
 kubectl patch deployment secure-app -n vault --type='strategic' --patch='
 spec:
   template:
     spec:
       serviceAccountName: secure-sa
       securityContext:
         runAsNonRoot: true
         runAsUser: 1001
         fsGroup: 1001
       containers:
       - name: app
         securityContext:
           runAsNonRoot: true
           runAsUser: 1001
           allowPrivilegeEscalation: false
           capabilities:
             drop:
               - ALL
         volumeMounts:
         - name: tls-secret-volume
           mountPath: /etc/app/tls
           readOnly: true
       volumes:
       - name: tls-secret-volume
         secret:
           secretName: app-tls
           defaultMode: 0400
 '

Verification Commands

Verify the secret was created correctly:
 kubectl get secret app-tls -n vault

 # Verify certificate matches
 kubectl get secret app-tls -n vault -o jsonpath='{.data.tls\.crt}' | base64 -d | diff - /root/app-cert.crt

 # Verify key matches  
 kubectl get secret app-tls -n vault -o jsonpath='{.data.tls\.key}' | base64 -d | diff - /root/app-key.key

Verify the deployment configuration:
 # Check ServiceAccount
 kubectl get deployment secure-app -n vault -o jsonpath='{.spec.template.spec.serviceAccountName}'

 # Check volume mount
 kubectl get deployment secure-app -n vault -o jsonpath='{.spec.template.spec.volumes[?(@.secret.secretName=="app-tls")].name}'

 # Check security context
 kubectl get deployment secure-app -n vault -o jsonpath='{.spec.template.spec.containers[0].securityContext.runAsUser}'

Verify RBAC permissions:
 # Test if ServiceAccount can access secrets in vault namespace
 kubectl auth can-i get secrets --as=system:serviceaccount:vault:secure-sa -n vault

 # Test if ServiceAccount is restricted from other namespaces
 kubectl auth can-i get secrets --as=system:serviceaccount:vault:secure-sa -n default

Check pod status:
 kubectl get pods -n vault -l app=secure-app

 # Check pod logs for any errors
 kubectl logs -n vault -l app=secure-app --tail=10

Troubleshooting

If pods are failing to start:
 # Check pod events
 kubectl describe pods -n vault -l app=secure-app

 # Check if secrets are accessible
 kubectl exec -n vault -it $(kubectl get pods -n vault -l app=secure-app -o jsonpath='{.items[0].metadata.name}') -- ls -la /etc/app/tls

 # Check ServiceAccount exists
 kubectl get serviceaccount secure-sa -n vault

If RBAC permissions are incorrect:
 # Check Role and RoleBinding
 kubectl get role,rolebinding -n vault -l app=secure-app

 # Describe Role to see permissions
 kubectl describe role secret-access -n vault

 # Check what the ServiceAccount can do
 kubectl auth can-i --list --as=system:serviceaccount:vault:secure-sa -n vault



12. We want to deploy a PodSecurity admission controller to enforce security standards across the cluster.

Fix the error in /etc/kubernetes/pki/podsecurity_configuration.yaml which will be used by the PodSecurity admission controller.

Ensure that the restricted policy level is enforced across all namespaces by default, with audit and warn modes for baseline violations.

Enable the plugin on the API server with the correct configuration file.The PodSecurity admission controller should reject any pods that don't meet the restricted policy standards.

A copy of the kube-apiserver.yaml is available in /tmp/kube-apiserver-backup.yaml so you can revert if the configuration goes wrong. Ensure that the kube-apiserver is working correctly, as it will be required for grading the exam.



Solution
Update /etc/kubernetes/pki/podsecurity_configuration.yaml with the correct PodSecurity configuration:

apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "restricted"
      enforce-version: "latest"
      audit: "baseline"
      audit-version: "latest"
      warn: "baseline"
      warn-version: "latest"
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces: ["kube-system"]

Update /etc/kubernetes/manifests/kube-apiserver.yaml:

    - --enable-admission-plugins=NodeRestriction,PodSecurity
    - --admission-control-config-file=/etc/kubernetes/pki/podsecurity_configuration.yaml

The API server will automatically restart and pickup this configuration.



13. A vulnerable deployment has been identified in the security-scanning namespace. Your task is to utilize the pre-configured KubeLinter configuration located at /root/kube-linter-config.yaml to identify and rectify all security issues in this deployment.

Tasks:

Scan the vulnerable deployment using the provided KubeLinter configuration.
Identify all security violations present in the deployment.
Address and resolve the security issues identified in the deployment.
Verify that the revised deployment successfully passes all security checks.
The vulnerable deployment can be found at /tmp/.init/manifests/vulnerable-deployment.yaml, and KubeLinter is pre-installed with the configuration file already set up.



Solution
Step 1: Scan the Vulnerable Deployment

kubelinter lint /tmp/.init/manifests/vulnerable-deployment.yaml --config /root/kube-linter-config.yaml  

Step 2: Identify Security Issues
The scan will reveal the following security issues:

- Privileged container  
- No read-only root filesystem  
- Running as root user  
- Using latest image tag  
- Missing resource limits  

Step 3: Create a Fixed Deployment

cat > /tmp/.init/manifests/vulnerable-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: insecure-app
  namespace: security-scanning
spec:
  replicas: 2
  selector:
    matchLabels:
      app: insecure-app
  template:
    metadata:
      labels:
        app: insecure-app
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - insecure-app
              topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: nginx:1.25-alpine
        securityContext:
          privileged: false
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
EOF

Step 4: Apply the Fixed Deployment

kubectl apply -f /tmp/.init/manifests/vulnerable-deployment.yaml

Step 5: Verify Security Fixes

kubelinter lint /tmp/.init/manifests/vulnerable-deployment.yaml --config /root/kube-linter-config.yaml



14. The administrator has partially upgraded cluster1.

Complete the upgrade process by updating the worker node to the latest installed version available among the nodes.


Solution
Instructions for Upgrading Kubernetes on Worker Node

On the control plane node, execute the following command to check the current node status:

kubectl get nodes

You should see an output similar to the one below, indicating that the worker node node02 is running an older version of Kubernetes:

NAME                    STATUS   ROLES           AGE   VERSION
cluster1-controlplane   Ready    control-plane   49m   v1.34.0
node02                  Ready    worker          29m   v1.33.0

Next, SSH into node02:

ssh node02

On node02, use your preferred text editor to open the file that defines the Kubernetes APT repository:

vim /etc/apt/sources.list.d/kubernetes.list

On node02, update the version in the URL to the next available minor release, e.g., v1.34:

deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /

After making the changes, save the file and exit the text editor. Proceed with the next instruction.

On node02, run the following command to add the release key:

echo 'y' | curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --yes --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

Then, update the package list:

sudo apt-get update

Check the available versions of kubeadm:

apt-cache madison kubeadm

Switch back to the control plane node to drain node02:

kubectl drain node02 --ignore-daemonsets --delete-emptydir-data

Based on the version information displayed by apt-cache madison, the available package version for Kubernetes v1.34.0 should be noted. To install kubeadm for Kubernetes v1.34.0, execute the following command on node02:

sudo apt-get install -y kubeadm=1.34.0-1.1

Now, run the following command to upgrade the node:

sudo kubeadm upgrade node

If kubeadm is on hold during the upgrade, unhold it or follow the appropriate suggestions mentioned in the output.

Please note that the above steps may take several minutes to complete.

On node02, unhold and then upgrade the kubelet and kubectl versions:

sudo apt-mark unhold kubelet kubectl
sudo apt-get install --allow-change-held-packages -y kubelet=1.34.0-1.1 kubectl=1.34.0-1.1

Optionally, hold the versions again:

sudo apt-mark hold kubelet kubectl

On node02, refresh the systemd configuration and restart the Kubelet service with the following commands:

sudo systemctl daemon-reload
sudo systemctl restart kubelet

Switch back to the control plane node and uncordon node02:

kubectl uncordon node02

Finally, verify the version upgrade on the control plane node:

kubectl get nodes

You should now see both nodes at v1.34:

NAME                    STATUS   ROLES           AGE   VERSION
cluster1-controlplane   Ready    control-plane   70m   v1.34.0
node02                  Ready    worker          50m   v1.34.0


15. The kubectl commands executed on cluster2-controlplane are encountering TLS certificate errors.

Identify the issue within the kubeconfig file and take the necessary steps to resolve it.

If you are unable to execute the kubectl commands successfully, please refer to the kubeconfig backup file located at /root/cert-test/config.backup.


Solution
Solution: Kubeconfig Certificate Path Correction
Step 1: Verify the Certificate Error
First, confirm that kubectl commands are failing:

kubectl get nodes

You should see an error indicating certificate issues.

Step 2: Examine the Kubeconfig File
Check the current certificate configuration in the kubeconfig:

grep "certificate-authority" ~/.kube/config

You'll see the incorrect path:

certificate-authority: /etc/kubernetes/pki/ca.crt.moved

Step 3: Correct the Certificate Path
Edit the kubeconfig file to fix the path:

sed -i 's|/etc/kubernetes/pki/ca.crt.moved|/etc/kubernetes/pki/ca.crt|g' ~/.kube/config

Alternative method using a text editor:

sudo vi ~/.kube/config

Then find the line with certificate-authority: /etc/kubernetes/pki/ca.crt.moved and change it to certificate-authority: /etc/kubernetes/pki/ca.crt

Step 4: Verify the Correction
Confirm the path is now correct:

grep "certificate-authority" ~/.kube/config

Should show:

certificate-authority: /etc/kubernetes/pki/ca.crt

Step 5: Test Cluster Connectivity
Verify that kubectl commands now work:

kubectl get nodes

Step 6: Alternative Solution - Restore from Backup
If you prefer, you can restore the original kubeconfig:

cp /root/cert-test/config.backup ~/.kube/config

Verification Commands
Verify the CA certificate exists at the correct location:

ls -la /etc/kubernetes/pki/ca.crt

Check certificate details:

openssl x509 -in /etc/kubernetes/pki/ca.crt -text -noout | head -5

Key Points
Always verify certificate paths in kubeconfig files
The standard location for cluster CA certificates is /etc/kubernetes/pki/ca.crt
Certificate path errors are a common cause of kubectl connectivity issues
Maintain backups of working kubeconfig files for quick recovery


16. Please exit from cluster2-controlplane and ensure that you are in cluster1-controlplane for the subsequent question.

Run a CIS Benchmark scan using kube-bench and fix the etcd data directory permission issue.

Tasks:

Run kube-bench to scan the master components
Identify the etcd data directory permission violations
Requirements:

Use kube-bench with appropriate targets to find the issue
Restrict etcd directory permissions to the CIS recommended level
Verify that the fix resolves the violation
kube-bench is pre-installed. Focus on finding and fixing the etcd data directory permission issue specifically.


Solution
CIS Benchmark etcd Permission Fix
Step 1: Run CIS Benchmark Scan
kube-bench run --targets master

Step 2: Identify the etcd Permission Issue
Look for output like this in the scan results:

[FAIL] 1.1.11 Ensure that the etcd data directory permissions are set to 700 or more restrictive (Automated)
[FAIL] 1.1.12 Ensure that the etcd data directory ownership is set to etcd:etcd (Automated)

Step 3: Fix the etcd Directory Permissions
# Set restrictive permissions on etcd directory
chmod 700 /var/lib/etcd

# Set proper ownership (if etcd user exists)
chown etcd:etcd /var/lib/etcd 

if the user is not available you can create it :

sudo groupadd etcd 2>/dev/null || true
sudo useradd -r -g etcd -s /bin/false etcd 2>/dev/null || true

Step 4: Verify the Fix
# Check current permissions
ls -ld /var/lib/etcd
# Should show: drwx------

# Run kube-bench again to verify the fix
kube-bench run --targets master | grep "1.1.12"
# Should now show: [PASS]