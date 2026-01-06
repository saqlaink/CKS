API Server issues: https://github.com/kodekloudhub/community-faq/blob/main/docs/diagnose-crashed-apiserver.md


<details>
    <summary>While trying to use the crictl, I got bunch of docker.sock authentication and authorization issues. </summary>
The issue is with configuring the /etc/crictl.yaml for --runtime-endpoint and --image-endpoint.
It can be done by referring to the official docs 31:

```bash
cat <<EOF | sudo tee -a /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
EOF
```
Or
```bash
crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock # Set the containerd sock as runtime endpoint

crictl config --set image-endpoint=unix:///run/containerd/containerd.sock # Set the containerd sock as image endpoint
```
</details>


<details>
<summary>There was one question about tls that you need to delete some tls protocol between kube API server and etcd.</summary>
Add below in API server file

```bash
--tls-min-version=VersionTLS12
```
</details>

<details>
<summary>Passing Parameters to kubelet through systemd unit file.</summary>

```bash
sudo systemctl edit kubelet
```

Scenario 1️⃣: Add security flags

```
[Service]
Environment="KUBELET_EXTRA_ARGS=--anonymous-auth=false --authorization-mode=Webhook"
```

Scenario 2️⃣: Override ExecStart (more control)
```
[Service]
ExecStart=
ExecStart=/usr/bin/kubelet \
  --config=/var/lib/kubelet/config.yaml \
  --anonymous-auth=false \
  --authorization-mode=Webhook
```

⚠️ ExecStart= must be cleared first (empty line) — exam favorite mistake.
</details>