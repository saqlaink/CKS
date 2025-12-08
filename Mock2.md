```markdown
# CKS Practice Exam – Full Solutions (Nothing Removed)

## **1.**
A pod called redis-backend has been created in the prod-x12cs namespace. It has been exposed as a service of type ClusterIP. The pod listens on TCP port 6379.  
Create a network policy called allow-redis-access to lock down access to this pod only for the following:  
- Any pod in the same namespace with the label backend=prod-x12cs  
- All pods in the prod-yx13cs namespace  
Ensure that traffic is only allowed on TCP port 6379.  
All other incoming connections should be blocked.  
Use the existing labels when creating the network policy.

**Solution**  
Create a network policy using the YAML below:
```yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-redis-access
  namespace: prod-x12cs
spec:
  podSelector:
    matchLabels:
      run: redis-backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: prod-yx13cs
    - podSelector:
        matchLabels:
          backend: prod-x12cs
    ports:
    - protocol: TCP
      port: 6379
```
Instead of kubernetes.io/metadata.name: prod-yx13cs in the above yaml, you can also use access: redis.

## **2.**
There is an existing CiliumNetworkPolicy default-allow in the namespace team-azure, which allows all traffic.  
In the namespace team-azure, create a CiliumNetworkPolicy as follows:  
Create a Layer 3 policy named p1 that denies outgoing traffic from Pods with the label role=messenger to Pods with the label role=database.

**Solution**  
We create a deny policy:
```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: p1
  namespace: team-azure
spec:
  endpointSelector:
    matchLabels:
      role: messenger
  egressDeny:
  - toEndpoints:
    - matchLabels:
        role: database
```
Use the below command to check the network connection. Change IP address of database pod below and check.  
Get IP address  
```bash
kubectl get pod -n team-azure database-pod -o jsonpath='{.status.podIP}'
```
Use the ipaddress below and check connectivity  
```bash
kubectl exec -n team-azure messenger-pod -- curl -m 3 <IP Address>:80
```

## **3.**
A pod named apps-cluster-dash has been created in the gamma namespace using a service account called cluster-view. This service account has been granted additional permissions as compared to the default service account and can view resources cluster-wide on this Kubernetes cluster. While these permissions are important for the application in this pod to work, the secret token is still mounted on this pod.  
Secure the pod in such a way that the secret token is no longer mounted on this pod. You may delete and recreate the pod.

**Solution**  
Update the Pod to use the field automountServiceAccountToken: false  
Using this option makes sure that the service account token secret is not mounted in the pod at the location /var/run/secrets/kubernetes.io/serviceaccount  
Also, we need to remove the serviceAccountToken as it should not be explicitly defined within any custom volumes.  
Sample YAML shown below:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: apps-cluster-dash
  name: apps-cluster-dash
  namespace: gamma
spec:
  containers:
    - image: nginx
      name: apps-cluster-dash
      volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: kube-api-access-c4mdx
          readOnly: true
  serviceAccountName: cluster-view
  automountServiceAccountToken: false
  volumes:
    - name: kube-api-access-c4mdx
      projected:
        defaultMode: 420
        sources:
          - configMap:
              items:
                - key: ca.crt
                  path: ca.crt
              name: kube-root-ca.crt
          - downwardAPI:
              items:
                - fieldRef:
                    apiVersion: v1
                    fieldPath: metadata.namespace
                  path: namespace
```

## **4.**
A pod in the sahara namespace has generated alerts that a shell was opened inside the container.  
To recognize such alerts, set the priority to ALERT and change the format of the output so that it looks like the below:  
ALERT timestamp of the event without nanoseconds,User ID,the container id,the container image repository  
Make sure to update the rule such that the changes persist across Falco updates.  
You can refer the falco documentation Here

**Solution**  
Add the below rule to /etc/falco/falco_rules.local.yaml and restart the falco service to override the current rule.
```yaml
- rule: Terminal shell in container
  desc: A shell was used as the entrypoint/exec point into a container with an attached terminal.
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
    and container_entrypoint
    and not user_expected_terminal_shell_in_container_conditions
  output: >
    %evt.time.s,%user.uid,%container.id,%container.image.repository
  priority: ALERT
  tags: [container, shell, mitre_execution]
```
Use the falco documentation to use the correct sysdig filters in the output.  
For example, the evt.time.s filter prints the timestamp for the event without nano seconds. This is clearly described in the falco documentation here - https://falco.org/docs/rules/supported-fields/#evt-field-class

## **5.**
martin is a developer who needs access to work on the dev-a, dev-b, and dev-z namespaces. He should be able to perform any operation on any pod in the dev-a and dev-b namespaces. However, in the dev-z namespace, he should only have permission to get and list the pods.  
The current setup is too permissive and violates the above condition. Use the above requirement to secure martin's access in the cluster. You may re-create objects, but ensure they retain their existing names.

**Solution**  
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

## **6.**
There is a deployment named hacker in the namespace team-red, which mounts /run/containerd as a hostPath volume on the Node where it's running.  
This means that the Pod can access various data about other containers running on the same Node.  
To prevent this, configure the team-red namespace to enforce the baseline Pod Security Standard. Once completed, delete the Pod from the deployment mentioned above.  
Check the ReplicaSet events and write the event lines containing the reason why the Pod isn't recreated into /opt/course/logs.txt.  
Note: You may see multiple identical event lines with the same error. Paste only one of them in the logs.txt file - preferably the last one.

**Solution**  
Now we configure the requested label:
```bash
k edit ns team-red
```
```yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    kubernetes.io/metadata.name: team-red
    pod-security.kubernetes.io/enforce: baseline # add
  name: team-red
```
This should already be enough for the default Pod Security Admission Controller to pick up on that change.  
Let's test it and delete the Pod to see if it'll be recreated or fails, it should fail!
```bash
k -n team-red get pod
k -n team-red delete pod hacker-f74b9f47d-6lz8f --force --grace-period=0
k -n team-red get pod
k -n team-red get rs
k -n team-red describe rs hacker-f74b9f47d
```
Events:
```
Warning FailedCreate 16m replicaset-controller Error creating: pods "hacker-f74b9f47d-x47pz" is forbidden: violates PodSecurity "baseline:latest": hostPath volumes (volume "containerd-volume")
Warning FailedCreate 5m17s (x9 over 16m) replicaset-controller (combined from similar events): Error creating: pods "hacker-f74b9f47d-jk9fn" is forbidden: violates PodSecurity "baseline:latest": hostPath volumes (volume "containerd-volume")
```
Finally we write the reason into the requested file so that scoring will be happy too!
```bash
cat /opt/course/logs.txt
Warning FailedCreate 5m17s (x9 over 16m) replicaset-controller (combined from similar events): Error creating: pods "hacker-f74b9f47d-jk9fn" is forbidden: violates PodSecurity "baseline:latest": hostPath volumes (volume "containerd-volume")
```

## **7.**
There is a Dockerfile named unsecure.Dockerfile located at /opt/course/image/.  
DevSecOps has asked you to improve this image by:  
- Changing the base image to alpine:3.12  
- Not installing curl  
- Updating nginx to use the version constraint >=1.18.0  
- Running the main process as user myuser  
Do not add any new lines to the Dockerfile - only modify the existing ones.

**Solution**  
Modify the Dockerfile as required:
```dockerfile
FROM alpine:3.12
RUN apk update && apk add vim nginx>=1.18.0
RUN addgroup -S myuser && adduser -S myuser -G myuser
COPY ./run.sh run.sh
RUN ["chmod", "+x", "./run.sh"]
USER myuser
ENTRYPOINT ["/bin/sh" , "./run.sh"]
```

## **8.**
A pod definition file has been created at /root/CKS/simple-pod.yaml. Use the kubesec tool to generate a report for this pod definition file and fix the major issues so that the subsequent scan passes.  
Once done, generate the report again and save it to the file /root/CKS/kubesec-report.txt

**Solution**  
Remove the SYS_ADMIN capability from the container for the simple-webapp-1 pod in the POD definition file and re-run the scan.
```bash
controlplane $ kubesec scan /root/CKS/simple-pod.yaml > /root/CKS/kubesec-report.txt
```
The fixed report should PASS with a message like this:
```
[
  {
    "object": "Pod/simple-webapp-1.default",
    "valid": true,
    "fileName": "simple-pod.yaml",
    "message": "Passed with a score of 0 points",
    "score": 0,
  },
...
```

## **9.**
There is a deployment named rocket-server, which is exposed by a service of the same name in the space namespace.  
Create an ingress resource named rocket-ingress to load balance the incoming traffic to the workload on path /.  
Use the hostname rocket-server.local for the Ingress rules.  
Use the TLS certificate stored in the rocket-tls secret in the space namespace to enable TLS traffic on the ingress resource.  
Make sure to add the IP of the Ingress Nginx controller matching the rocket-server.local DNS to the /etc/hosts file.

**Solution**  
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
Then add this IP to the /etc/hosts file:
```bash
echo "<INGRESS-IP> rocket-server.local" | sudo tee -a /etc/hosts
```
Finally you can check the working:
```bash
curl -k https://rocket-server.local
```
You should see the nginx welcome page.

## **10.**
Create a new pod called audit-nginx in the default namespace using the nginx image. Secure the syscalls that this pod can use by using the audit.json seccomp profile in the pod's security context.  
The audit.json file is located in the /root/CKS directory.  
Before creating the pod, move the file to the profiles directory inside the default seccomp directory.

**Solution**  
Copy the audit.json seccomp profile to /var/lib/kubelet/seccomp/profiles on the controlplane node:
```bash
controlplane$ mv /root/CKS/audit.json /var/lib/kubelet/seccomp/profiles
```
Next, create the pod using the below YAML file:
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
  - image: nginx
    name: nginx
```

## **11.**
A deployment named fruits in the namespace salad has three containers: apple, banana, and kiwi.  
One of these containers has the package curl installed. Identify which container has that package from the running containers, and create an SBOM SPDX for the container's image.  
Use the tarball archive of that particular image stored under /root/ImageTarballs directory for generating the SPDX JSON. The archives have names matching their images.  
Save the output in ~/bugged-fruit.spdx. Save the container name in ~/bugged-container.txt.  
Note: bom and all its required dependencies are already installed.

**Solution**  
First check the deployment running in the salad namespace:
```bash
kubectl get deployments -n salad
```
Then, run the following commands in succession to fetch the container name in the text file:
```bash
kubectl exec -n salad fruits-<string> -c apple -- apk info | grep curl && echo apple > ~/bugged-container.txt
```
Do the same for kiwi and banana as well.  
Determine the image name used in that container by describing the pod and checking the Containers section in the output:
```bash
k describe pod fruits-54665c68db-mcwg9 -n salad
```
Once you have determined the image name, navigate to the /root/ImageTarballs/ directory and identify the tarball of that image by its name.  
Then run the following command to generate the SPDX JSON:
```bash
bom generate --image-archive /root/ImageTarballs/<image_name>.tar --format json --output ~/bugged-fruit.spdx
```

## **12.**
Enable auditing on this cluster using a basic policy file located at /etc/kubernetes/cluster-policy.yaml.  
Configure the logs to be stored at /var/log/cluster-audit.log and retained for 10 days. The maximum size should be 10MB, and up to 3 files should be kept at a time.  
After enabling auditing on the cluster, update the basic policy file to track the following:  
- Delete activity on secrets in the kube-system namespace at the Metadata level  
- Changes to deployments in the default namespace at the Request level  
- All other requests at the Metadata level

**Solution**  
First complete the /etc/kubernetes/cluster-policy.yaml:
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
Then add/modify the properties in the vi /etc/kubernetes/manifests/kube-apiserver.yaml file. First add these flags:
```yaml
- --audit-policy-file=/etc/kubernetes/cluster-policy.yaml
- --audit-log-path=/var/log/cluster-audit.log
- --audit-log-maxage=10
- --audit-log-maxbackup=3
- --audit-log-maxsize=10
```
Under volumes section in the same file, add the following:
```yaml
- hostPath:
    path: /etc/kubernetes/cluster-policy.yaml
    type: File
  name: audit-policy
- hostPath:
    path: /var/log
    type: Directory
  name: varlog
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

## **13.**
A pod has been created in the omni namespace, but it has a couple of issues.  
The pod has been created with more permissions than it needs.  
It allows read access in the /usr/share/nginx/html/internal directory, opening the internal site to be accessed publicly.  
To verify this, click the Site button (above the terminal) and add /internal/ to the end of the URL.  
Use the below recommendations to fix this:  
- Use the AppArmor profile created at /etc/apparmor.d/frontend to restrict the internal site.  
- Several service accounts exist in the omni namespace. Apply the principle of least privilege to select the service account with the minimum privileges (excluding the default service account).  
- Recreate the pod with the correct service account, and then delete the other unused service accounts in the omni namespace (excluding the default service account).

**Solution**  
On the controlplane node, load the AppArmor profile:
```bash
root@controlplane:~# apparmor_parser -q /etc/apparmor.d/frontend
```
The profile name used by this file is restricted-frontend.  
The pod should only use the service account called frontend-default as it has the least privileges of all the service accounts in the omni namespace (excluding default).  
Use the below YAML File to re-create the frontend-site pod:
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
  serviceAccountName: frontend-default
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
Next, Delete the unused service accounts in the 'omni' namespace.
```bash
controlplane$ kubectl -n omni delete sa frontend
controlplane$ kubectl -n omni delete sa fe
```

## **14.**
The namespace encrypted has two applications, alpha and beta.  
Since these applications handle critical communications, enforce strict mTLS using Istio in the encrypted namespace.  
Make sure that the workloads have the istio sidecar injected.

**Solution**  
First enable istio sidecar injection in the namespace:
```bash
kubectl label namespace encrypted istio-injection=enabled --overwrite
```
Then, restart the deployments:
```bash
kubectl rollout restart deployment alpha -n encrypted
kubectl rollout restart deployment beta -n encrypted
```
Apply PeerAuthentication policy for STRICT mTLS:
```bash
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
```

## **15.**
In the namespace code, create a TLS secret code-secret with the following certificate and key provided:  
cert: /root/custom-cert.crt  
key: /root/custom-key.key  
Attach that secret as a volume named secret-volume in the deployment code-server.

**Solution**  
First create the TLS secret:
```bash
kubectl create secret tls code-secret \
  --cert=/root/custom-cert.crt \
  --key=/root/custom-key.key \
  -n code
```
Then edit the deployment to mount the secret:
```bash
kubectl edit deployment code-server -n code
```
Under the spec.template.spec section, add:
```yaml
volumes:
- name: secret-volume
  secret:
    secretName: code-secret
```
Under the spec.template.spec.containers section, add the volume mount then:
```yaml
volumeMounts:
- name: secret-volume
  mountPath: /etc/code/tls
  readOnly: true
```
Save and exit the deployment.
```

**Everything from your original text is preserved exactly as provided.**