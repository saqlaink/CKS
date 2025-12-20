# CKS Exam Solutions

## 1. Pod Security Issues in Omni Namespace

A pod has been created in the omni namespace, but it has a few issues that need to be addressed. The pod has been created with more permissions than it needs. It allows read access to the /usr/share/nginx/html/internal directory, making the Internal Site publicly accessible. To verify this, click the Site button (above the terminal) and add /internal/ to the end of the URL.

Use the below recommendations to resolve this. Use the AppArmor profile created at /etc/apparmor.d/frontend to restrict access to the internal site. The omni namespace has several service accounts. Apply the principle of least privilege and use the service account with the minimum privileges (excluding the default service account). Once the pod is recreated with the correct service account, delete the other unused service accounts in the omni namespace (excluding the default service account). Do not create a new service account or use the default service account.

### Solution

On the controlplane node, load the AppArmor profile:

```bash
root@controlplane:~# apparmor_parser -q /etc/apparmor.d/frontend
```

The profile name used by this file is `restricted-frontend` (open the `/etc/apparmor.d/frontend` file to check).

To verify that the profile was successfully loaded, use the `aa-status` command:

```bash
root@controlplane:~# aa-status | grep restricted-frontend
   restricted-frontend
```

The pod should only use the service account called `frontend-default` as it has the least privileges of all the service accounts in the omni namespace (excluding default). The other service accounts, `fe` and `frontend` have additional permissions (check the roles and rolebindings associated with these accounts).

Use the below YAML file to re-create the frontend-site pod after deleting the original frontend-site pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: frontend-site
  namespace: omni
spec:
  securityContext:
    appArmorProfile:
      type: Localhost
      localhostProfile: restricted-frontend
  serviceAccountName: frontend-default #Use the service account with least privileges
  containers:
- image: nginx:alpine
    name: nginx
    volumeMounts:
- mountPath: /usr/share/nginx/html
      name: test-volume
  volumes:
- name: test-volume
    hostPath:
       path: /data/pages
       type: Directory
```

Alternatively, you can also use the edit command to edit the running pod definition:

```bash
kubectl edit pod frontend-site -n omni
```

Then delete the frontend-site pod and apply the file saved in the /tmp folder after editing like the one shown here: A copy of your changes has been stored to "/tmp/kubectl-edit-3250548530.yaml"

Next, Delete the unused service accounts in the 'omni' namespace.

```bash
controlplane$ kubectl -n omni delete sa frontend
controlplane$ kubectl -n omni delete sa fe
```

## 2. Secret Handling in Orion Namespace

A pod has been created in the orion namespace. It uses secrets as environment variables. Extract the decoded secret for the CONNECTOR_PASSWORD and place it under /root/CKS/secrets/CONNECTOR_PASSWORD. You are not yet done; instead of using secrets as an environment variable, mount the secret as a read-only volume at the path /mnt/connector/password, which the application can then use.

### Answer

To extract the secret, run:

```bash
kubectl -n orion get secrets a-safe-secret -o jsonpath='{.data.CONNECTOR_PASSWORD}' | base64 --decode >/root/CKS/secrets/CONNECTOR_PASSWORD
```

One way that is more secure to distribute secrets is to mount it as a read-only volume. Use the following YAML file to recreate the POD with secret mounted as a volume:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-xyz
  namespace: orion
  labels:
    name: app-xyz
spec:
  containers:
    - name: app-xyz
      image: nginx:alpine
      ports:
        - containerPort: 3306
      volumeMounts:
        - name: secret-volume
          mountPath: /mnt/connector/password
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: a-safe-secret
```

## 3. Static Analysis of YAML and Dockerfiles

The Release Engineering Team has shared some YAML manifests and Dockerfiles with you for review. These files are located under /opt/course/. As a container security expert, your task is to perform a manual static analysis and identify possible security issues related to unwanted credential exposure. Note that running processes as root is not a concern in this task. Record the filenames containing issues in /opt/course/security-issues.txt. Note: Assume that all referenced files, folders, secrets, and volume mounts are present in the Dockerfiles and YAML manifests. You can ignore any syntax or logic errors.

### Answer

File `q3_file1.Dockerfile` copies a file `secret-token` over (`COPY secret-token .`), uses it (`RUN /etc/register.sh ./secret-token`) and deletes it afterwards (`RUN rm ./secret-token`). But because of the way Docker works, every RUN, COPY and ADD command creates a new layer and every layer is persisted in the image. This means that even if the file `secret-token` gets deleted, it's still included with the image.

File `q3_file2.yaml` contains plain text password under env section `value: P@sSw0rd`. Its secure practice to utilize k8s's secrets to store passwords.

```bash
echo -e "q3_file1.Dockerfile\nq3_file2.yaml" > /opt/course/security-issues.txt
```

## 4. Seccomp Profile for Pod

Create a new pod named audit-nginx in the default namespace using the nginx:alpine image. Secure the syscalls that this pod can use by using the audit.json seccomp profile in the pod's security context. The audit.json file is provided at the /root/CKS directory. Move it into the profiles directory inside the default seccomp directory before creating the pod.

### Answer

Copy the audit.json seccomp profile to `/var/lib/kubelet/seccomp/profiles` on the controlplane node:

```bash
controlplane$ mv /root/CKS/audit.json /var/lib/kubelet/seccomp/profiles
```

Next, create the pod using the below YAML File

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: audit-nginx
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
  containers:
- image: nginx:alpine
    name: nginx
```

## 5. Fixing Kube-Bench Issues

We have identified a few issues with our kubernetes setup and need your help in fixing them. Fix the following issues on kubelet: Kubelet service file permission issues, Kubelet config.yaml permission issues. Fix the following issues on etcd: Incorrect ownership of the etcd directory. Fix the following issues on the controlplane node: Incorrect value of the profiling argument for: kube-controller-manager, kube-scheduler. Kube-bench is installed, and its config files are available under /opt/kube-bench. Use the cis-1.10 benchmark with the current Kubernetes version. Note: Only fix issues that have the status FAIL, except issue number 1.2.5. Also, ignore the issues with policies.

### Answer

**Solution**  
The fixes will be mentioned when you run kube-bench and generate a report.

For kubelet issues, run:

```bash
kube-bench --benchmark cis-1.10 --config-dir /opt/kube-bench/cfg run --targets node
```

Update the kubelet service file `/usr/lib/systemd/system/kubelet.service` and kubelet config YAML `/var/lib/kubelet/config.yaml` with the correct permissions:

```bash
chmod 600 /usr/lib/systemd/system/kubelet.service
chmod 600 /var/lib/kubelet/config.yaml
```

For controlplane and etcd issues, run:

```bash
kube-bench --benchmark cis-1.10 --config-dir /opt/kube-bench/cfg run --targets master
```

For controlplane fix, update the Controller Manager and Scheduler static pod definition file to make sure that the `--profiling=false` parameter is set. For this use the vi editor to edit both files:

```bash
vi /etc/kubernetes/manifests/kube-controller-manager.yaml
vi /etc/kubernetes/manifests/kube-scheduler.yaml
```

and under the `spec.containers.command` section, add another parameter `--profiling=false`. Save and exit the files.

To fix the etcd issue, change the file ownership:

```bash
sudo chown -R etcd:etcd /var/lib/etcd
```

If the user and group are not present, add them:

```bash
sudo groupadd --system etcd
sudo useradd -s /sbin/nologin --system -g etcd etcd
```

## 6. Falco Rule Update

There is suspicious activity in the cluster involving one of the pods running the httpd:2.4-alpine image. Falco generates frequent alerts that start with: File below a known binary directory opened for writing. Identify the rule causing this alert and update it as per the below requirements: Output should be displayed as: CRITICAL File below a known binary directory opened for writing (user_id=user_id file_updated=file_name command=command_that_was_run). Alerts are logged to /opt/security_incidents/alerts.log. Do not update the default rules file directly. Instead, use the falco_rules.local.yaml file to override. Note: After updating the alert rule, it may take up to a minute for the alerts to appear in the new log location.

### Solution

Enable file_output in `/etc/falco/falco.yaml` on the controlplane node:

```yaml
file_output:
  enabled: true
  keep_alive: false
  filename: /opt/security_incidents/alerts.log
```

Next, add the updated rule under the `/etc/falco/falco_rules.local.yaml` and hot reload the Falco service:

```yaml
- rule: Write below binary dir
  desc: an attempt to write to any file below a set of binary directories
  condition: >
    bin_dir and evt.dir = < and open_write
    and not package_mgmt_procs
    and not exe_running_docker_save
    and not python_running_get_pip
    and not python_running_ms_oms
    and not user_known_write_below_binary_dir_activities
  output: >
    File below a known binary directory opened for writing (user_id=%user.uid file_updated=%fd.name command=%proc.cmdline)
  priority: CRITICAL
  tags: [filesystem, mitre_persistence]
```

To perform hot-reload falco use 'kill -1 /SIGHUP':

```bash
kill -1 $(cat /var/run/falco.pid)
```

Alternatively, you can also restart the falco service by running:

```bash
systemctl restart falco
```

## 7. SBOM Generation for Container

A deployment named fruits in the namespace salad has three containers: apple, banana, and kiwi. One of these containers has the package curl installed. Identify which container has that package from the running containers, and create an SBOM SPDX for the container's image. Use the tarball archive of that particular image stored under /root/ImageTarballs directory for generating the SPDX JSON. The archives have names matching their images. Save the output in ~/bugged-fruit.spdx. Save the container name in ~/bugged-container.txt. Note: bom as well as all its required dependencies have already been installed.

### Solution

First check the deployment running in the salad namespace:

```bash
kubectl get deployments -n salad
```

Then, run the following commands in succession to fetch the container name in the text file:

```bash
kubectl exec -n salad fruits-<string> -c apple -- apk info | grep curl && echo apple > ~/bugged-container.txt
```

Replace `<string>` with the actual random string in your pod name. Do the same for kiwi and banana as well.

Determine the image name used in that container by describing the pod and checking the Containers section in the output:

```bash
k describe pod fruits-54665c68db-mcwg9 -n salad
```

Your pod name could be different.

Once you have determined the image name, navigate to the `/root/ImageTarballs/` directory and identify the tarball of that image by its name. Then run the following command to generate the SPDX JSON:

```bash
bom generate --image-archive /root/ImageTarballs/<image_name>.tar --format json --output ~/bugged-fruit.spdx
```

Replace the `<image_name>` with the actual name of the tar file.

## 8. ImagePolicyWebhook Integration

We need to ensure that when pods are created in this cluster, they cannot use the latest image tag, irrespective of the repository being used. To achieve this, a simple Admission Webhook Server has been developed and deployed. A service called image-bouncer-webhook is deployed in the cluster. This Webhook server ensures that the developers of the team cannot use the latest image tag. Use the following specs to integrate it with the cluster using an ImagePolicyWebhook: Create a new admission configuration file at /etc/admission-controllers/admission-configuration.yaml. The kubeconfig file with the credentials to connect to the webhook server is located at /root/CKS/ImagePolicy/admission-kubeconfig.yaml. Note: The /root/CKS/ImagePolicy/ directory is already mounted on the kube-apiserver at path /etc/admission-controllers, so reference that path in reference the admission configuration. Ensure that if the latest tag is used, the request must be rejected at all times. Enable the Admission Controller. Finally, delete the existing pod in the magnum namespace that violates the policy and recreate it, ensuring the same image but using tag 1.27. NOTE: If the kube-apiserver becomes unresponsive, this can affect the validation of this exam. In such a case, restore the kube-apiserver using the backup file created at: /root/backup/kube-apiserver.yaml. Wait for the API to be available again and proceed.

### Solution

Create the below `admission-configuration.yaml` inside `/root/CKS/ImagePolicy` directory in the controlplane node:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/admission-controllers/admission-kubeconfig.yaml
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: false
```

The `/root/CKS/ImagePolicy` is mounted at the path `/etc/admission-controllers` directory in the kube-apiserver. So, you can directly place the files under `/root/CKS/ImagePolicy`.

Here is a snippet of the volume and volumeMounts (already added to apiserver config, you don't need to do anything here):

```yaml
  containers:
  .
  .
  .
  volumeMounts:
- mountPath: /etc/admission-controllers
      name: admission-controllers
      readOnly: true
  volumes:
- hostPath:
      path: /root/CKS/ImagePolicy/
      type: DirectoryOrCreate
    name: admission-controllers
```

Next, update the kube-apiserver command flags and add ImagePolicyWebhook to the enable-admission-plugins flag. Use the configuration file that was created in the previous step as the value of admission-control-config-file. Note: Remember, this command will be run inside the kube-apiserver container, so the path must be `/etc/admission-controllers/admission-configuration.yaml` (mounted from `/root/CKS/ImagePolicy` in controlplane).

Edit the kube-apiserver manifest file:

```bash
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

to add the following flags under `spec.containers.command`:

```yaml
    - --admission-control-config-file=/etc/admission-controllers/admission-configuration.yaml
    - --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
```

In case we mess up while solving the question, API server could become unresponsive. For example: The connection to the server controlplane:6443 was refused - did you specify the right host or port? In such case scenario restore kube-apiserver to it's default state using the backup provided at `/root/backup/kube-apiserver.yaml`

Run the below command to restore the kube-apiserver initial state:

```bash
cp -v /root/backup/kube-apiserver.yaml /etc/kubernetes/manifests
```

### Remediate the Non-Compliant Pod

After the ImagePolicyWebhook is active, you must fix the existing app-0403 pod in the magnum namespace, as it violates the new policy, and , Admission controllers only check new requests; they don't affect pods that are already running.

**Get the Pod's Definition**  
First, get the YAML definition of the running pod and save it to a file.

```bash
kubectl get pod app-0403 -n magnum -o yaml > app-0403-fix.yaml
```

**Clean and Edit the YAML**  
Now, edit the file. The YAML you just saved contains live, system-managed fields. You must clean it up before you can re-apply it.

```bash
vi app-0403-fix.yaml
```

Change:

```yaml
image: gcr.io/google-containers/busybox
```

to:

```yaml
image: gcr.io/google-containers/busybox:1.27
```

Save and exit the file.

**Delete the Old Pod**  
You must delete the running pod before you can create a new one with the same name.

```bash
kubectl delete pod app-0403 -n magnum
```

**Apply the Fixed Manifest**  
Finally, create the new, compliant pod from your corrected YAML file. The webhook will inspect this request and allow it.

```bash
kubectl apply -f app-0403-fix.yaml
```

## 9. Service Account Configuration

Create a service account named bot-sa in the namespace automated. Make sure that this service account does not get automatically mounted to workloads. A workload named sweeper is also in the automated namespace. Set the deployment's service account to the newly created service account, and mount the service account token as a projected volume. Do not change any other fields in the deployment.

### Solution

Create the ServiceAccount bot-sa without auto-mount:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bot-sa
  namespace: automated
automountServiceAccountToken: false
```

Then, update the existing sweeper deployment in the automated namespace to: Use the new bot-sa service account, Manually mount the service account token via a projected volume. Recreate the deployment or edit it with the following definition:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sweeper
  namespace: automated
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sweeper
  template:
    metadata:
      labels:
        app: sweeper
    spec:
      serviceAccountName: bot-sa # Added
      automountServiceAccountToken: false # Prevent auto-mount
      containers:
        - name: sweeper
          image: busybox:1.36
          command: ["sleep", "3600"]
          volumeMounts: # Section added
            - name: sa-token
              mountPath: /var/run/secrets/tokens
              readOnly: true
      volumes: # Section added
        - name: sa-token
          projected:
            sources:
              - serviceAccountToken:
                  path: bot-token
                  expirationSeconds: 3600
                  audience: default
```

The token is now manually mounted as a projected volume at `/var/run/secrets/tokens/bot-token`.

## 10. Fixing Deployment in Restricted Namespace

A deployment named web-server is running in the restricted namespace. Identify the reason why the deployment is not in a running state; fix the issue so that it can be in a running state. Do not change the namespace labels or container image.

First check the status of the web-server deployment:

```bash
kubectl get deployments.apps -n restricted
```

You should see the READY column showing 0/1 which means the pod is not running.

Then run the following command:

```bash
kubectl describe deployments.apps web-server -n restricted
```

In the output, check the Conditions section:

```
Conditions:
  Type Status Reason
  ---- ------ ------
  Progressing True NewReplicaSetCreated
  Available False MinimumReplicasUnavailable
  ReplicaFailure True FailedCreate
```

This shows that the replica creation failed. Lets find out the reason why:

```bash
kubectl describe replicaset -n restricted
```

The Events section will provide the exact error cause:

```
  Warning FailedCreate 25s (x6 over 105s) replicaset-controller (combined from similar events): Error creating: pods "web-server-7d8b84ccc8-9ppsm" is forbidden: violates PodSecurity "restricted:latest": allowPrivilegeEscalation != false (container "nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), runAsUser=0 (container "nginx" must not set runAsUser=0), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

This confirms the pod was blocked by Pod Security Admission (PSA) because it doesn't meet the restricted security policy.

Now, check that the namespace is labeled to enforce the restricted policy:

```bash
kubectl get namespace restricted --show-labels
```

You should see this label present: `pod-security.kubernetes.io/enforce=restricted` which means kubernetes is actively enforcing restricted-level pod Security in this namespace.

According to the Events section of the replicaset, we need to make some changes in our deployment's securityContext section. Open the deployment file to edit:

```bash
kubectl edit deployment web-server -n restricted
```

Enter insert mode by typing `i` and change the securityContext section as follows:

```yaml
securityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop:
      - ALL
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
```

Also, remove `runAsUser: 0` and then press `Esc` followed by `:wq!`. This will save the changes.

You should then see the pod in a running state:

```bash
kubectl get pods -n restricted
```

```
NAME READY STATUS RESTARTS AGE
web-server-55549f978f-lgp8w 1/1 Running 0 5m
```

## 11. Network Policy for Traffic Control

Deployment web-app is running in the products namespace. Database product-db is running in the database namespace. Create a network policy named allow-traffic-to-products that allows traffic from product-db to the web-app workload, as well as all traffic originating from the payments namespace. Utilize the labels applied on the relevant resources.

First, find out the labels on the web-app deployment pod in the products namespace:

```bash
kubectl get pods -n products --show-labels
```

```
NAME READY STATUS RESTARTS AGE LABELS
web-app-64cd7668-bp8nv 1/1 Running 0 133m app=web-app,pod-template-hash=64cd7668
```

Then, find out the same for the product-db deployment pod in database namespace:

```bash
kubectl get pods -n database --show-labels
```

```
NAME READY STATUS RESTARTS AGE LABELS
product-db-74b9dfd7cb-hr9vf 1/1 Running 0 20m app=database,pod-template-hash=74b9dfd7cb
```

Then, find the labels on the database and payments namespaces:

```bash
kubectl get ns database --show-labels
```

```
NAME STATUS AGE LABELS
database Active 137m kubernetes.io/metadata.name=database
```

```bash
kubectl get ns payments --show-labels
```

```
NAME STATUS AGE LABELS
payments Active 137m kubernetes.io/metadata.name=payments
```

Finally, create and apply the network policy yaml with the determined labels:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-traffic-to-products
  namespace: products
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
- Ingress
  ingress:
- from:
- namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: database
      podSelector:
        matchLabels:
          app: database
- namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: payments
```

## 12. Security Context for Deployment Containers

Edit the gamma deployment in the galaxy namespace to ensure that all containers meet the following requirements: Run as user 1001, Do not allow privilege escalation, Mount their file systems as read-only.

Run the following command to edit the gamma deployment in the galaxy namespace:

```bash
k edit deployment gamma -n galaxy
```

Then add the following to each container under `spec->template->spec->containers` section and add the below section in every pod as allowPrivilegeEscalation and readOnlyRootFilesystem are container level fields not pod level:

```yaml
securityContext:
    runAsUser: 1001
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
```

Save the file and exit.

## 13. Ingress Resource Creation

A deployment rocket-server is exposed using the service of the same name in the space namespace. Create an ingress resource named rocket-ingress to load balance the incoming traffic to the workload on path /. Use the hostname rocket-server.local in the Ingress rules. Utilize the TLS certificate stored in a secret named rocket-tls in the space namespace so that it enables TLS traffic on that ingress resource.

### Solution

Run the command:

```bash
k describe deployment rocket-server -n space
```

and check the Port field under Pod Template -> Containers.

Then create and apply the following ingress yaml file:

```yaml
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
```

Once that is done, check the IP address of the ingress-nginx-controller:

```bash
kubectl get svc -n ingress-nginx
```

Then add this IP to the `/etc/hosts` file:

```bash
echo "<INGRESS-IP> rocket-server.local" | sudo tee -a /etc/hosts
```

Finally you can check the working:

```bash
curl -k https://rocket-server.local
```

You should the nginx welcome page.

## 14. RBAC Role Adjustment

A developer named martin needs access to work on the dev-a, dev-b, and dev-z namespaces. He should have the ability to carry out any operation on any pod in the dev-a and dev-b namespaces. However, on the dev-z namespace, he should only have the permission to get and list the pods. The current setup is too permissive and violates the above condition. Use the above requirement and secure martin's access in the cluster. You may re-create objects; however, ensure to use the same names as the ones in currently in effect.

The role called dev-user-access has been created for all three namespaces: dev-a, dev-b and dev-z. However, the role in the dev-z namespace grants martin access to all operation on all pods. To fix this, delete and re-create the role using the following YAML:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
    name: dev-user-access
    namespace: dev-z
rules:
  - apiGroups:
    - ""
    resources:
    - pods
    verbs:
    - get
    - list
```

## 15. Auditing Configuration

You need to enable auditing on this cluster. A basic policy file is available at /etc/kubernetes/cluster-policy.yaml. The logs should be stored at /var/log/cluster-audit.log. The logs should be retained for 10 days and should not exceed 10MB. A maximum of 3 files should be kept at a time. After you enable auditing on the cluster, update the basic policy file to track the following: Delete activity on secrets in the kube-system namespace at the Metadata level, Changes to deployments in the default namespace at the Request level, All other requests at the Metadata level. Make sure your changes to the policy file are in effect. Note: A copy of the kube-apiserver.yaml is kept in ~/ so that you can revert if the configuration goes wrong. Make sure kube-apiserver is working fine for the sake of grading the exam.

### Solution

First complete the `/etc/kubernetes/cluster-policy.yaml`:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - "RequestReceived"
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

Then add/modify the properties in the `vi /etc/kubernetes/manifests/kube-apiserver.yaml` file. First add these flags:

```yaml
- --audit-policy-file=/etc/kubernetes/cluster-policy.yaml
- --audit-log-path=/var/log/cluster-audit.log
- --audit-log-maxage=10
- --audit-log-maxbackup=3
- --audit-log-maxsize=10
```

Under volumes section in the same file, add the following:

```yaml
volumes:
- name: audit-policy
    hostPath:
      path: /etc/kubernetes/cluster-policy.yaml
      type: File
- name: varlog
    hostPath:
      path: /var/log
      type: Directory
```

Under volumeMounts section in the same file, add the following as well:

```yaml
- mountPath: /etc/kubernetes/cluster-policy.yaml
  name: audit-policy
  readOnly: true
- mountPath: /var/log
  name: varlog
```

Save and exit the file. Then run the following command to check the status of the api server and other components:

```bash
kubectl get pods -n kube-system
```

Wait till all components come up in a running state.

Finally, check the status of the log file:

```bash
ls -l /var/log/cluster-audit.log
```

The file should be automatically created and should now start storing logs.
