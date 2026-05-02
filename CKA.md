Here’s your structured **README.md** with proper numbering, consistent “Question” sections, and clean formatting.

---

# 📘 CKA Practice README

> Structured set of Kubernetes (CKA-style) tasks with **Questions + Copy-Paste Answers** for quick revision and execution.
> Source: 

---

## 🧩 Question 1: Kubernetes HPA Configuration - apache-server

Create a new **HorizontalPodAutoscaler (HPA)** named `apache-server` in the `autoscale` namespace.

### Tasks:

* Target Deployment: `apache-server`
* Namespace: `autoscale`
* CPU utilization target: **50%**
* Minimum replicas: **1**
* Maximum replicas: **4**
* Downscale stabilization window: **30 seconds**

### ✅ Answer

```bash
kubectl autoscale deployment apache-server \
  -n autoscale \
  --cpu-percent=50 \
  --min=1 \
  --max=4 \
  --name=apache-server

kubectl patch hpa apache-server -n autoscale --type='merge' -p '{
  "spec": {
    "behavior": {
      "scaleDown": {
        "stabilizationWindowSeconds": 30
      }
    }
  }
}'

kubectl get hpa apache-server -n autoscale -o yaml
```

### Alternative (Single YAML)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: apache-server
  namespace: autoscale
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: apache-server
  minReplicas: 1
  maxReplicas: 4
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 30
```

Apply using:

```bash
kubectl apply -f hpa.yaml
```

### Notes / Troubleshooting

* Ensure **metrics-server** is installed, otherwise HPA will not function.
* CPU metrics will show `<unknown>` if resource requests are not defined in the Deployment.
* `behavior` field requires `autoscaling/v2` API version.
* Always verify HPA status after creation.

---

## 🧩 Question 2: Create Ingress Resource

Create a new Ingress resource with the following details:

### Tasks:

* Name: `echo`
* Namespace: `sound-repeater`
* Expose Service: `echoserver-service`
* Path: `http://example.org/echo`
* Service Port: `8080`
* Verify with curl returning HTTP 200

### ✅ Answer

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo
  namespace: sound-repeater
spec:
  rules:
  - host: example.org
    http:
      paths:
      - path: /echo
        pathType: Prefix
        backend:
          service:
            name: echoserver-service
            port:
              number: 8080
EOF
```

### Verification & Troubleshooting

```bash
# Ensure Ingress Controller exists
kubectl get pods -A | grep -i ingress || echo "⚠️ No ingress controller found"

# Add host entry if DNS not configured
echo "<INGRESS-IP> example.org" | sudo tee -a /etc/hosts

# Get Ingress IP
kubectl get ingress echo -n sound-repeater

# Test
curl -o /dev/null -s -w "%{http_code}\n" http://example.org/echo
```

---

## 🧩 Question 3: Kubernetes Node Preparation

Prepare a Linux system for Kubernetes. Docker is already installed, but it needs to be configured for `kubeadm`.

### Tasks:

1. Install `cri-dockerd` from `/home/candidate/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb`
2. Enable and start the `cri-docker` service
3. Configure system parameters (apply immediately & persist after reboot)
4. Load kernel module `br_netfilter`

### ✅ Answer

```bash
# Install cri-dockerd
sudo dpkg -i /home/candidate/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb

# Enable and start service
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable cri-docker
sudo systemctl start cri-docker

# Load required kernel module
sudo modprobe br_netfilter

# Set system parameters (use /sbin/sysctl if sysctl not found)
sudo /sbin/sysctl -w net.bridge.bridge-nf-call-iptables=1
sudo /sbin/sysctl -w net.ipv6.conf.all.forwarding=1
sudo /sbin/sysctl -w net.ipv4.ip_forward=1
sudo /sbin/sysctl -w net.netfilter.nf_conntrack_max=131072

# Make parameters persistent
sudo bash -c 'cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.ipv6.conf.all.forwarding=1
net.ipv4.ip_forward=1
net.netfilter.nf_conntrack_max=131072
EOF'

# Apply persistent settings
sudo sysctl --system

# Persist kernel module
echo br_netfilter | sudo tee /etc/modules-load.d/k8s.conf

# Verify
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward
sysctl net.ipv6.conf.all.forwarding
sysctl net.netfilter.nf_conntrack_max
```

---

## 🧩 Question 4: Fix Unschedulable WordPress Pods

Pods are not scheduled due to high resource requests in the `relative-fawn` namespace with 3 replicas.

### Tasks:

* Scale down Deployment to 0
* Adjust resource requests to allow scheduling
* Scale back to 3 replicas
* Verify all Pods are running

### ✅ Answer

```bash
# Scale down deployment
kubectl scale deploy wordpress -n relative-fawn --replicas=0

# Edit deployment
kubectl edit deploy wordpress -n relative-fawn

# --- Inside editor, update ONLY this section ---
# resources:
#   requests:
#     cpu: "200m"
#     memory: "512Mi"
# -----------------------------------------------

# Scale back to 3 replicas
kubectl scale deploy wordpress -n relative-fawn --replicas=3

# Verify pods are running
kubectl get pods -n relative-fawn

# (Optional) Check scheduling issues if still Pending
kubectl describe pod -n relative-fawn <pod-name> | grep -i "insufficient"
```

---

## 🧩 Question 5: ConfigMap Update & Immutability

Update ConfigMap with TLS settings, restart Deployment, verify with curl, and make ConfigMap immutable.

### Tasks:

* SSH to node
* Edit ConfigMap to allow TLSv1.2 and TLSv1.3
* Restart Deployment to apply changes
* Verify curl with `--tls-max 1.2`
* Make ConfigMap immutable

### ✅ Answer

```bash
# SSH to node
ssh cka000048b

# Edit ConfigMap (update nginx config)
kubectl edit configmap nginx-config -n nginx-static
# --> Set: ssl_protocols TLSv1.2 TLSv1.3;

# Restart deployment to apply changes
kubectl rollout restart deployment nginx-static -n nginx-static

# Verify TLSv1.2 works
curl --tls-max 1.2 https://web.k8s.local

# Make ConfigMap immutable
kubectl patch configmap nginx-config -n nginx-static -p '{"immutable": true}'

# Verify immutable
kubectl get cm nginx-config -n nginx-static -o yaml | grep immutable
```

### ⚠️ Common Fixes

* JSON error → ensure: `-p '{"immutable": true}'`
* Changes not applied → forgot `rollout restart`
* Wrong namespace → always use `-n nginx-static`
* Patch fails → fallback:
  ```bash
  kubectl get cm nginx-config -n nginx-static -o yaml > cm.yaml
  vi cm.yaml   # add: immutable: true
  kubectl apply -f cm.yaml
  ```

---

## 🧩 Question 6: Fix kubeadm Cluster (etcd IP issue)

Cluster broken after migration. API server cannot connect to external etcd due to outdated IP.

### Tasks:

* Identify old etcd endpoint
* Get new node IP
* Update kube-apiserver manifest
* Restart kubelet
* Verify cluster health

### ✅ Answer

```bash
# Verify kubeconfig
export KUBECONFIG=/etc/kubernetes/admin.conf

# Check kube-apiserver logs for etcd connection issues
crictl ps | grep kube-apiserver
crictl logs <apiserver-container-id>

# Check current etcd endpoint (shows OLD IP)
sudo grep etcd /etc/kubernetes/manifests/kube-apiserver.yaml

# Get correct node IP (NEW IP)
ip a | grep inet

# Edit API server manifest
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml

# Replace OLD IP with NEW IP (example below)
# --etcd-servers=https://192.168.15.222:2379   ❌
# --etcd-servers=https://10.0.2.2:2379         ✅

# Restart kubelet (recreates static pods)
sudo systemctl restart kubelet

# Wait and verify cluster
kubectl get nodes
kubectl get pods -A

# Validate etcd is reachable
curl -k https://<NEW-IP>:2379
```

### ⚠️ Common Fixes

* If still failing → try localhost: `--etcd-servers=https://127.0.0.1:2379`
* Check kubelet errors: `journalctl -u kubelet -f`
* Ensure etcd cert paths exist: `ls /etc/kubernetes/pki/`
* Fast retry: `sudo systemctl restart kubelet && watch kubectl get pods -A`

---

## 🧩 Question 7: cert-manager CRDs Extraction

Verify cert-manager and extract CRDs and field documentation.

### Tasks:

* List all cert-manager CRDs
* Save to `/home/candidate/resources.yaml` (default output format)
* Extract `subject` field documentation
* Save to `/home/candidate/subject.yaml`

### ✅ Answer

```bash
# Verify cert-manager (optional check)
kubectl get crds | grep cert-manager

# Save cert-manager CRDs (default output ONLY - no -o flag)
kubectl get crds | grep cert-manager > /home/candidate/resources.yaml

# Extract subject field documentation
kubectl explain certificate.spec.subject > /home/candidate/subject.yaml

# Verify files created
ls -la /home/candidate/resources.yaml /home/candidate/subject.yaml
head /home/candidate/resources.yaml
```

---

## 🧩 Question 8: NetworkPolicy Configuration

Apply correct policy allowing communication from `frontend` (frontend namespace) to `backend` (backend namespace).

### Tasks:

* Inspect labels and ports
* Identify correct NetworkPolicy YAML
* Apply the policy (do NOT modify existing)
* Test connectivity

### ✅ Answer

```bash
# Inspect labels and ports
kubectl get pods -n frontend --show-labels
kubectl get pods -n backend --show-labels
kubectl get svc -n backend

# Check available NetworkPolicy YAMLs
ls /home/candidate/netpol
cat /home/candidate/netpol/*

# Identify correct policy (must match ALL below):
# - podSelector targets backend pods
# - ingress.from includes:
#     namespaceSelector for frontend namespace
#     AND podSelector for frontend pods
# - includes correct backend port
# - NOT open to all (no 0.0.0.0/0, no empty selectors)

# Apply the correct file
kubectl apply -f /home/candidate/netpol/<correct-file>.yaml

# Verify policy applied
kubectl get netpol -A

# Test connectivity
FRONTEND_POD=$(kubectl get pod -n frontend -o jsonpath='{.items[0].metadata.name}')
BACKEND_SVC=$(kubectl get svc -n backend -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n frontend $FRONTEND_POD -- curl -m 5 $BACKEND_SVC
```

### ⚠️ Fixes (if not working)

* ❌ Timeout → wrong policy selected OR missing port
* ❌ No route → namespaceSelector missing
* ❌ Still blocked → default deny exists → ensure ingress rule is correct
* ❌ Wrong labels → re-check pod labels
* ❌ Service not resolving → try ClusterIP instead:
  ```bash
  kubectl get svc -n backend
  kubectl exec -n frontend $FRONTEND_POD -- curl -m 5 <CLUSTER-IP>:<PORT>
  ```

---

## 🧩 Question 9: Ingress → Gateway API Migration

Migrate existing Ingress resource `web` to Gateway API while maintaining HTTPS access.

### Tasks:

* Extract TLS config and routing from Ingress
* Create Gateway `web-gateway` with TLS
* Create HTTPRoute `web-route` with same rules
* Test with curl
* Delete old Ingress

### ✅ Answer

```bash
# Extract required values from existing Ingress
TLS_SECRET=$(kubectl get ingress web -o jsonpath='{.spec.tls[0].secretName}')
SERVICE_NAME=$(kubectl get ingress web -o jsonpath='{.spec.rules[0].http.paths[0].backend.service.name}')
SERVICE_PORT=$(kubectl get ingress web -o jsonpath='{.spec.rules[0].http.paths[0].backend.service.port.number}')

# Create Gateway
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: gateway.web.k8s.local
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: ${TLS_SECRET}
EOF

# Create HTTPRoute
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
  - name: web-gateway
  hostnames:
  - gateway.web.k8s.local
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: ${SERVICE_NAME}
      port: ${SERVICE_PORT}
EOF

# Test (use -k for self-signed TLS)
curl -k https://gateway.web.k8s.local

# Delete old Ingress
kubectl delete ingress web
```

### ⚠️ Quick Fixes

* TLS error → ensure secret exists: `kubectl get secret ${TLS_SECRET}`
* 503 error → verify svc/port: `kubectl get svc ${SERVICE_NAME}`
* Route not working → check: `kubectl describe httproute web-route`
* Gateway not ready → `kubectl get gateway`

---

## 🧩 Question 10: Restore MariaDB with PVC

Restore MariaDB Deployment ensuring data persistence using existing retained PV.

### Tasks:

* Get existing PV
* Create PVC bound to existing PV
* Update Deployment to use PVC
* Apply and verify

### ✅ Answer

```bash
# Get existing PV (IMPORTANT: note PV_NAME)
kubectl get pv

# Create PVC bound to existing PV
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb
  namespace: mariadb
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 250Mi
  volumeName: <PV_NAME>
EOF

# Edit Deployment to use PVC
sed -i '/containers:/a\
        volumeMounts:\
        - mountPath: /var/lib/mysql\
          name: mariadb-storage' /home/candidate/mariadb-deployment.yaml

sed -i '/spec:/a\
      volumes:\
      - name: mariadb-storage\
        persistentVolumeClaim:\
          claimName: mariadb' /home/candidate/mariadb-deployment.yaml

# (If sed fails, manually add under spec.template.spec)

# Apply Deployment
kubectl apply -f /home/candidate/mariadb-deployment.yaml

# Verify
kubectl get pvc -n mariadb
kubectl get pods -n mariadb
kubectl describe pod -n mariadb | grep -i mount
```

### ⚠️ Fixes

* PVC Pending → missing volumeName or wrong PV
* Pod Crash → wrong mountPath (/var/lib/mysql)
* Not scheduled → namespace mismatch
* Changes not applied → `kubectl rollout restart deploy -n mariadb`

---

## 🧩 Question 11: Install Argo CD via Helm

Install Argo CD without CRDs (already installed) in `argocd` namespace.

### Tasks:

* Add Helm repo
* Create namespace
* Generate template (save YAML)
* Install with Helm
* Verify

### ✅ Answer

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

kubectl create namespace argocd

helm template argocd argo/argo-cd \
  --version 7.7.3 \
  --namespace argocd \
  --set crds.install=false \
  > /home/candidate/argo-helm.yaml

helm install argocd argo/argo-cd \
  --version 7.7.3 \
  --namespace argocd \
  --set crds.install=false

kubectl get pods -n argocd
```

### ⚠️ Quick Fixes

* CRDs error → ensure `--set crds.install=false`
* Namespace missing → run `kubectl create ns argocd`
* Wrong chart → must use `argo/argo-cd` (not bitnami)
* File not created → check path `/home/candidate/argo-helm.yaml`

---

## 🧩 Question 12: PriorityClass & Preemption

Create PriorityClass `high-priority` with value one less than highest existing, then patch Deployment to use it.

### Tasks:

* Get highest existing PriorityClass value
* Create new PriorityClass
* Patch Deployment
* Verify rollout and priority applied

### ✅ Answer

```bash
# Get highest existing PriorityClass value
kubectl get priorityclass
HIGHEST=$(kubectl get priorityclass -o jsonpath='{range .items[*]}{.value}{"\n"}{end}' | sort -nr | head -1)
NEW_VALUE=$((HIGHEST-1))

# Create new PriorityClass
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: ${NEW_VALUE}
globalDefault: false
description: "High priority for user workloads"
EOF

# Patch deployment to use new PriorityClass
kubectl patch deployment busybox-logger -n priority \
  --type='merge' \
  -p '{"spec":{"template":{"spec":{"priorityClassName":"high-priority"}}}}'

# Verify rollout
kubectl rollout status deployment busybox-logger -n priority

# Verify priority applied
kubectl get pods -n priority
kubectl describe pod -n priority | grep -i "Priority Class Name"
```

### ⚠️ Common Fixes

* JSON error → ensure correct quotes in patch command
* No rollout → use: `kubectl rollout restart deployment busybox-logger -n priority`
* Priority not applied → check: `kubectl get pc high-priority -o yaml`

---

## 🧩 Question 13: Expose Deployment via NodePort

Reconfigure Deployment `front-end` in namespace `spline-reticulator` to expose port **80/tcp** and create NodePort service.

### Tasks:

* Add containerPort 80 to Deployment
* Create NodePort Service
* Verify endpoints

### ✅ Answer

```bash
# SSH into correct node
ssh cka000022

# Set namespace (optional shortcut)
kubectl config set-context --current --namespace=spline-reticulator

# Add containerPort 80 to deployment
kubectl patch deployment front-end \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/ports","value":[{"containerPort":80}]}]'

# Create NodePort service
kubectl expose deployment front-end \
  --name=front-end-svc \
  --port=80 \
  --target-port=80 \
  --type=NodePort

# Verify
kubectl get svc front-end-svc
kubectl get endpoints front-end-svc
```

### ⚠️ Quick Fixes

* If patch fails (ports already exist), use edit: `kubectl edit deploy front-end`
* If service exists already: `kubectl delete svc front-end-svc` && recreate
* If no endpoints: `kubectl get pods --show-labels` - ensure labels match selector in svc
* Restart rollout if needed: `kubectl rollout restart deploy front-end`

---

## 🧩 Question 14: Create StorageClass

Create StorageClass `local-path` using rancher provisioner with `WaitForFirstConsumer` binding mode, set as default.

### Tasks:

* Create StorageClass with correct provisioner and binding mode
* Set as default StorageClass
* Verify

### ✅ Answer

```bash
# Create StorageClass YAML
cat <<EOF > sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
EOF

# Apply it
kubectl apply -f sc.yaml

# Set as default StorageClass
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Verify
kubectl get sc
kubectl describe sc local-path | grep -i default
```

### ⚠️ Common Fixes

* Error: must specify --patch → Fix quoting exactly as shown above
* If shell escaping issue (zsh): `kubectl patch storageclass local-path --type=merge -p "{\"metadata\":{\"annotations\":{\"storageclass.kubernetes.io/is-default-class\":\"true\"}}}"`
* If not showing default → `kubectl get sc local-path -o yaml | grep volumeBindingMode`

---

## 🧩 Question 15: Install CNI (Flannel)

Cluster CNI failed. Install new CNI (Flannel v0.26.1) ensuring Pods communicate and NetworkPolicies work. Fix if CIDR mismatch occurs.

### Tasks:

* Apply Flannel manifest
* Check for CIDR mismatch
* Fix if nodes remain NotReady
* Verify

### ✅ Answer

```bash
# SSH
ssh cka000054

# Apply Flannel CNI
kubectl apply -f https://github.com/flannel-io/flannel/releases/download/v0.26.1/kube-flannel.yml

# Check status
kubectl get pods -n kube-flannel
kubectl get nodes

# If Node NotReady / Flannel Error → Fix CIDR mismatch

# Get cluster PodCIDR
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'

# Download Flannel YAML
curl -O https://github.com/flannel-io/flannel/releases/download/v0.26.1/kube-flannel.yml

# Replace default CIDR (10.244.0.0/16) with cluster CIDR (e.g. 192.168.0.0/16)
sed -i 's#10.244.0.0/16#192.168.0.0/16#g' kube-flannel.yml

# Reapply Flannel
kubectl delete -f kube-flannel.yml
kubectl apply -f kube-flannel.yml

# Verify
kubectl get nodes
```

---

## 🧩 Question 16: Add Sidecar Container

Update Deployment `synergy-leverager` by adding sidecar container (`busybox:stable`) streaming logs with shared volume.

### Tasks:

* Extract existing Deployment
* Add emptyDir volume
* Add volumeMount to existing container
* Add sidecar container
* Apply and verify

### ✅ Answer

```bash
# Extract deployment
kubectl get deploy synergy-leverager -o yaml > /tmp/synergy.yaml

# Edit the file
vi /tmp/synergy.yaml

# Make these changes under: spec.template.spec

# 1. Add volume (if not present):
# volumes:
# - name: log-volume
#   emptyDir: {}

# 2. In EXISTING container → ONLY add volumeMounts:
# volumeMounts:
# - name: log-volume
#   mountPath: /var/log

# 3. Add NEW sidecar container (IMPORTANT: new "-" entry):
# - name: sidecar
#   image: busybox:stable
#   command: ["/bin/sh", "-c", "tail -n+1 -f /var/log/synergy-leverager.log"]
#   volumeMounts:
#   - name: log-volume
#     mountPath: /var/log

# Apply changes
kubectl apply -f /tmp/synergy.yaml

# Verify
kubectl get pods
kubectl logs <pod-name> -c sidecar
```

### ⚠️ Common Fixes

* Sidecar not working → forgot "-" before new container
* Logs empty → /var/log not mounted in BOTH containers
* Pod crash → wrong command syntax
* Changes not applied → forgot `kubectl apply`

---

## 🧩 Question 17: Pod + API Access

Create Pod using ServiceAccount and query Kubernetes API for Secrets, saving output to file.

### Tasks:

* Create Pod `api-contact` in `project-swan` namespace with ServiceAccount `secret-reader`
* Exec into Pod and query API
* Save output to `/opt/course/9/result.json`

### ✅ Answer

```bash
# SSH
ssh cka9412

# Create Pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: api-contact
  namespace: project-swan
spec:
  serviceAccountName: secret-reader
  containers:
  - name: nginx
    image: nginx:1-alpine
    command: ["sleep","3600"]
EOF

# Wait for Pod
kubectl wait --for=condition=Ready pod/api-contact -n project-swan --timeout=60s

# Exec + query API + save output
kubectl exec -n project-swan api-contact -- sh -c '
curl -s --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
-H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
https://kubernetes.default.svc/api/v1/namespaces/project-swan/secrets \
> /opt/course/9/result.json
'

# Verify
kubectl exec -n project-swan api-contact -- cat /opt/course/9/result.json
```

---

# ✅ Notes

* All tasks are **CKA exam-focused**
* Answers are **optimized for copy-paste**
* Includes **real troubleshooting patterns**

---

If you want, I can also:

* Convert this into a **one-page cheat sheet (ultra short)**
* Add **common mistakes per question (exam traps)**
* Or turn it into a **Notion / printable PDF format**
