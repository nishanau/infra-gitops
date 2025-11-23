# Kubernetes Cluster Setup – Technical Guide and Log

## 1. Boilerplate Setup (All Nodes)
Prepare all nodes with consistent OS settings, disable swap, apply sysctl parameters, and install containerd.

**Reflection:** Swap must be disabled for accurate memory accounting. `ip_forward` and `br_netfilter` enable Pod networking across nodes.

### Commands and Purpose
| Command | Purpose | Why Necessary | Verification |
|----------|----------|----------------|---------------|
| `sudo apt update && sudo apt -y upgrade` | Update packages | Ensure OS is patched | `apt list --upgradable` |
| `sudo swapoff -a` | Disable swap temporarily | Scheduler needs no swap | `free -h` shows 0 swap |
| `sudo sed -i '/ swap / s/^/#/' /etc/fstab` | Disable swap permanently by commenting the swap line in /etc/fstab file | Prevent swap after reboot | `cat /etc/fstab` |
| `echo -e "overlay\nbr_netfilter" \| sudo tee /etc/modules-load.d/containerd.conf`| Load `overlay` & `br_netfilter` | Needed for networking | `lsmod | grep br_netfilter` |
| `sudo modprobe overlay && sudo modprobe br_netfilter` | Load kernel modules | Needed for container networking and filesystems | `lsmod | grep br_netfilter` |
| `echo -e "net.bridge.bridge-nf-call-iptables = 1\nnet.bridge.bridge-nf-call-ip6tables = 1\nnet.ipv4.ip_forward = 1" \| sudo tee /etc/sysctl.d/99-kubernetes-cri.conf` | Enable iptables, IPv6, and forwarding | Needed for services and routing | `sysctl net.ipv4.ip_forward` = 1 |
| `sudo sysctl --system ` | Apply sysctl settings | Enable ip_forward and bridge netfilter | `sysctl net.ipv4.ip_forward` = 1 |
| `sudo apt install -y containerd` | Install container runtime | Required for Pods | `systemctl status containerd` |
| `ssudo mkdir -p /etc/containerd && containerd config default \| sudo tee /etc/containerd/config.toml >/dev/null  ` | Create default config for containerd and put it in config.toml | Configuration for containerd to run | cat config.toml should have configurations sudo |
| Edit `/etc/containerd/config.toml` and set `SystemdCgroup=true` | Align cgroup drivers | Kubelet & containerd must use same cgroup driver | `grep SystemdCgroup /etc/containerd/config.toml` |
| `sudo systemctl restart/enable containerd` | Restart containderd and enable it so it runs automatically | Need containerd to run automatically for kubernetes to function properly | Systemctl status containerd shoud show enabled and active |

**Tip:** Use `grep –nF 'pattern' filename` to find line numbers.

---

## 2. Install Kubernetes Tools (All Nodes)
Install `kubelet`, `kubeadm`, and `kubectl`. These form the core of Kubernetes.

**Reflection:** Installing GPG keys and repositories reinforces package security principles.

| Command | Purpose | Why Necessary | Verification |
|----------|----------|----------------|---------------|
| `sudo apt install -y apt-transport-https ca-certificates curl gpg` | Install prereqs | Enable HTTPS repos and key management | `curl --version`, `gpg --version` |
| `sudo mkdir -p /etc/apt/keyrings`<br>`curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \| sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg` | Add Kubernetes repo key | Verify and trust Kubernetes packages | `ls /etc/apt/keyrings/` |
| `echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" \| sudo tee /etc/apt/sources.list.d/kubernetes.list` | Add Kubernetes apt repo | Source for kubeadm, kubelet, kubectl | `cat /etc/apt/sources.list.d/kubernetes.list` |
| `sudo apt update && sudo apt install -y kubelet kubeadm kubectl` | Install Kubernetes tools | Needed to bootstrap and manage cluster | `kubeadm version`, `kubectl version --client` |
| `sudo apt-mark hold kubelet kubeadm kubectl` | Prevent auto-updates | Avoid version mismatches across nodes | `apt-mark showhold` |


---

## 3. Control Plane Initialization
Initialize control-plane and configure networking.

**Reflection:** Similar to promoting a domain controller, defines the cluster brain.

| Command | Purpose | Why Necessary | Verification |
|----------|----------|----------------|---------------|
| `sudo kubeadm init --apiserver-advertise-address=192.168.101.89 --pod-network-cidr=10.244.0.0/16` | Initialize the control-plane node and set up cluster core components | Bootstraps the control plane by creating the API server, etcd, controller manager, scheduler, and certificates. It also installs core add-ons like DNS and kube-proxy. Without initialization, worker nodes cannot join and the cluster cannot function. | `kubectl get nodes` after setup should show the control-plane node in `NotReady` state until networking is applied. |
| `kubeadm join 192.168.101.89:6443 --token <token> --discovery-token-ca-cert-hash <hash>` | Join worker nodes to the cluster | Connects a node to the control-plane using a secure token and CA hash, allowing it to register as a worker and start running workloads. | `kubectl get nodes` should show the new node after a few minutes. Use `kubeadm token list` to check tokens or `kubeadm token create --print-join-command` to generate a new one if expired. |
| `mkdir -p $HOME/.kube`<br>`sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`<br>`sudo chown $(id -u):$(id -g) $HOME/.kube/config` | Configure `kubectl` for the current user | Copies the admin kubeconfig to the user’s home directory and adjusts ownership so `kubectl` commands can be executed without `sudo`. Without this, you’d need elevated privileges for every command. | `kubectl get nodes` should return the cluster’s nodes successfully without `sudo`. |
| `kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/refs/heads/master/Documentation/kube-flannel.yml` | Deploy the Flannel CNI plugin (Pod network) | Installs a Container Network Interface (CNI) to enable pod-to-pod communication across nodes. Without it, pods cannot communicate between nodes, keeping nodes in `NotReady` state. | `kubectl -n kube-system get pods -o wide` should show `flannel` pods running, and `kubectl get nodes` should show nodes as `Ready`. |

### At this point, all the nodes should be in the cluster connected. 

Verify node readiness:
```bash
kubectl get nodes
```

---

## 4. Horizontal Pod Autoscaler (HPA) Setup

### 4.1 Why HPA?
The **Horizontal Pod Autoscaler** automatically adjusts the number of pod replicas based on observed metrics (CPU, memory, or custom metrics). This ensures:
- **Cost efficiency**: Scale down during low traffic
- **Performance**: Scale up during high load
- **Availability**: Maintain service quality under varying demand

### 4.2 Prerequisites
HPA requires the **Metrics Server** to function. Install it:

```bash
# Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# For self-signed certificates (lab environments)
kubectl -n kube-system patch deployment metrics-server --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Verify installation
kubectl get deployment metrics-server -n kube-system
kubectl top nodes  # Should show CPU/Memory usage
```

### 4.3 HPA Configuration

**Base HPA (`manifests/base/hpa.yaml`)**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: next-portfolio-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: next-portfolio
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 80  # Scale when avg CPU > 80%
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100  # Can double pods instantly
          periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 60  # Wait 60s before scaling down
      policies:
        - type: Percent
          value: 50  # Reduce by 50% max per cycle
          periodSeconds: 15
```

### 4.4 Environment-Specific Patches

Using **Kustomize patches**, each environment can override HPA thresholds:

**Dev HPA Patch (`manifests/overlays/dev/hpa-patch.yaml`)**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: next-portfolio-hpa
spec:
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 30  # Lower threshold for dev testing
```

**Stage/Prod**: Similar patches with higher thresholds (70-80%)

### 4.5 How HPA Works

1. **Metrics Server** collects CPU/memory usage from kubelet every 15s
2. **HPA Controller** queries Metrics Server every 15s (default)
3. **Calculation**: 
   ```
   desiredReplicas = ceil[currentReplicas * (currentMetric / targetMetric)]
   ```
   Example: 2 pods @ 160% CPU → `ceil[2 * (160/80)] = 4 pods`
4. **Stabilization**: Waits for `stabilizationWindowSeconds` before scaling down (prevents flapping)
5. **Deployment** adjusts replica count

### 4.6 Verification & Monitoring

**Check HPA Status**
```bash
kubectl get hpa -n dev
# NAME                   REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# next-portfolio-hpa-dev Deployment/next-portfolio-dev 45%/30%   2         5         3          2d
```

**View Detailed HPA Events**
```bash
kubectl describe hpa next-portfolio-hpa-dev -n dev
```

**Monitor Real-Time Scaling**
```bash
watch -n 2 'kubectl get hpa -n dev && kubectl get pods -n dev'
```

### 4.7 HPA in Action

Below is a snapshot showing HPA responding to load:

![HPA Usage Example](./public/enterprise_cicd_k8s/hpa-usage.png)

**Observations:**
- Multiple HPA instances across `dev`, `stage`, and `prod` namespaces
- Each environment has different scaling thresholds
- Current CPU utilization vs. target clearly visible
- Replica counts adjust dynamically based on load

### 4.8 Best Practices

✅ **DO:**
- Set `requests` in Deployment (HPA needs this for percentage calculations)
- Use `behavior` field to control scaling speed
- Test with load testing tools (e.g., `kubectl run -it --rm load-generator --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://next-portfolio-service; done"`)
- Monitor for "flapping" (rapid scale up/down cycles)

❌ **DON'T:**
- Set `minReplicas` too low (breaks high availability)
- Set `maxReplicas` beyond node capacity
- Use HPA without resource requests
- Scale on memory alone (often leads to OOM kills)

### 4.9 Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| HPA shows `<unknown>/80%` | Metrics Server not installed | Install Metrics Server |
| Pods not scaling | No resource requests in Deployment | Add `resources.requests.cpu` |
| Rapid scaling oscillations | No stabilization window | Add `behavior.scaleDown.stabilizationWindowSeconds` |
| HPA not matching pods | `nameSuffix` applied | Use `name: next-portfolio-hpa` in patch (suffix auto-added) |

---

## 5. Cluster Health and Sanity Checks
Test inter-pod connectivity and DNS resolution.


| Command | Purpose | Why Necessary | Verification |
|----------|----------|----------------|---------------|
| `Pod_a=<pod_name> in node1`<br>`Pod_b_ip=<pod_ip> of pod in node2`<br>`kubectl exec -it $Pod_a -- ping -c3 $Pod_b_ip` | Test Pod-to-Pod cross-node connectivity | Verifies that the cluster network (CNI) correctly routes traffic between pods on different nodes. Confirms that the network overlay (e.g., Flannel, Calico) is functioning. | Ping should succeed with no packet loss, proving cross-node communication works. |
| `POD=$(kubectl get pod -l app=netshoot -o jsonpath='{.items[0].metadata.name}')`<br>`kubectl exec -it $POD -- nslookup kubernetes.default` | Verify DNS resolution inside the cluster | Confirms that CoreDNS is operational and that pods can resolve internal service names to ClusterIPs. Without functional DNS, services cannot communicate by name. | Should resolve `kubernetes.default` to a ClusterIP, confirming DNS health. |
| `kubectl create deploy echo --image=hashicorp/http-echo -- /http-echo -text="ok"`<br>`kubectl expose deploy echo --port=5678`<br>`kubectl exec -it $POD -- wget -qO- http://echo:5678` | Validate Service-to-Pod routing | Ensures that Kubernetes Services correctly route traffic to backend pods using cluster networking and kube-proxy. Validates internal load balancing. | Output should return `ok`, confirming service routing and pod reachability. |


All tests should succeed (ping, DNS, service routing).

---

## 6. Infrastructure Installation
Install basic infra tools like metrics server, storageCLass for PVCs (Persistent Volume Claims), Ingress Controller. 

| Command | Purpose | Why Necessary | Verification |
|----------|----------|---------------|---------------|
| `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`<br><br>If using in labs or self-signed certificates:<br>`kubectl -n kube-system patch deployment metrics-server --type='json' -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'` | Install **Metrics Server** | Provides usage metrics (CPU, memory, etc.) for nodes and pods in the cluster. Essential for `kubectl top` and Horizontal Pod Autoscaler. | `kubectl get pods -A` or `kubectl get deploy -A` should list **metrics-server**.<br><br>**Tip:** Alternatively, download the deployment file and add `"--kubelet-insecure-tls"` to `spec.template.spec.containers.args` manually. |
| `kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml`<br><br>`kubectl -n ingress-nginx get pods -A` | Install **Ingress Controller** | Exposes internal cluster services to external traffic using HTTP/HTTPS routing. Critical for making apps accessible publicly or via domain names. | `kubectl get pods -A` and `kubectl get deploy -A` should show **ingress-nginx** controller and related services. |



### Reset Cluster (if needed or you mess up something, for example the external ip of the nodes changed which I tried fixing and messed up whole lot of things)
```bash
sudo kubeadm reset -f
sudo systemctl stop kubelet containerd
sudo rm -rf /etc/cni/net.d /var/lib/cni/ /var/lib/kubelet /etc/kubernetes /var/lib/etcd
sudo ip link delete cni0
sudo ip link delete flannel.1
```

---

## 7. Containerizing the Apps

### 6.1 Writing Dockerfiles

A **Dockerfile** is a blueprint for creating a container image.  
For my portfolio app, I used two separate **Node:20-alpine** images — one for **build** and one for **runtime**.

1. In the **build stage**, I copied the `package*.json` files into `/app` and ran `npm ci` to perform a clean installation of dependencies.  
   Using `npm ci` instead of `npm install` ensures reproducible builds and faster installs when `package-lock.json` is present.

2. I then used `COPY . .` to copy the full source code (excluding files in `.dockerignore`) and ran `npm run build`, which created the production build of the app.

3. In the **runtime stage**, I used another **Node:20-alpine** image for a lightweight environment.  
   I copied only the compiled output — static, public, and standalone files — into the new image.  
   This minimizes image size and attack surface.

4. Permissions were set using:
   ```bash
   RUN mkdir -p .next/cache && chown -R node:node .next
   ```
5. I used the **USER node** directive for security (to avoid running as root) and exposed port 3000.
6. Finally, the container runs the app using:
   ```bash
   CMD ["node", "server.js"]
   ```

**Dockerfile for our NextPortfolio App**
```bash
# Build stage
FROM node:20-alpine AS build
WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci

# Copy all files and build
COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
ENV PORT=3000

# Copy only necessary files from build stage
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/public ./public

# Give permission to node user
RUN mkdir -p .next/cache && chown -R node:node .next

# Run as non-root user
USER node
EXPOSE 3000

# Start application
CMD ["node", "server.js"]

```

### 6.3 Example Deployment (next-portfolio)
ArgoCD is a Continuous Delivery tool that will use GitOps repository as the single source of truth. It will routinely check the repository for changes and applies the changes as it sees. We
Install via [official guide](https://argo-cd.readthedocs.io/en/stable/getting_started/). Convert `argocd-server` service to LoadBalancer (`kubectl –n argocd edit svc argocd-server`) and allocate some MetalLB IP range for example, `192.168.101.220–230`.

**MetalLB Configuration (metallb-pool.yaml)**
```yaml
# metallb-pool.yaml

apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lb-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.101.200-192.168.101.210  # make sure these are UNUSED

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2
  namespace: metallb-system
```

Extract admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d && echo
```

**Repo Structure**
```
manifests/
  base/
    deployment.yaml
    service.yaml
    kustomization.yaml
  overlays/
     dev/
       ingress.yaml
       kustomization.yaml
     stage/
       ingress.yaml
       kustomization.yaml
     prod/
       ingress.yaml
       kustomization.yaml
```

### 6.3 Example Deployment (next-portfolio)
All manifests are represented in yaml format. The first line has apiVersion: apps/v1 and networking.k8s.io/v1 for Ingress, the next is kind: Deployment, this is where I define if it is a Deployment or Service or Ingress. The next field is metadata, where I define the name and labels and annotations of the yaml.  The next field is spec where I define the specifications of the pod. Inside spec, I have replicas (defines how many copies of the pod to deploy), selector (used by the deployment or service to find the pod its serves), then I have template,  which defines the specifications of the container inside the pod. Inside template I have metadata which has labels, this label must match the spec.selector defined above. After metadata, I have spec which defines the specification of the container/s. Inside spec, I have containers and under this I have – name (name of container). Note: The ‘-’ sign means array, i.e if I have multiple containers then each ‘–name’ would signify a separate container insde template.spec.containers. After name I have image (image of container), ports.containerPort (ports the container listens to, 3000 in our case), env (env.name and env.value ). 
I can also add liveness, readiness probes. Liveness probe checks if the app inside the container are live and readiness probes checks if the app is ready to serve. I can also add resource limits using resources,  
 
I used strategy in spec.strategy to define how updates are rolled i.e. RollingUpdate(currently used) that runs the updated pod first before killing the old pod ensureing no downtime. The other option is Recreate which kills old pod before starting the new one. I used annotations so that other tools like prometheus can track the data for monitoring.

**Security:** 
I set a different serviceAccouintName (default if not set) than default to make sure the pod doesnt have priveleges more that necessary.  I also used securityContext inside the container to make sure the user inside the container is not root by default and operates on least priveleges. I deliberately give the UID, GID, not allow privelege escalation, and only read root files.  

I finally used lifecycle field for lifecycle events like run immediately after container starts (postStart) and run before container is stopped (preStop). 
 
Here is the final deployment.yaml of the next-portfolio app/pod/container. 
base/deployment.yaml:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: next-portfolio
  labels:
    app: next-portfolio
spec:
  replicas: 2  # 2 replicas for high availability
  revisionHistoryLimit: 5  # keep last 5 revisions for rollback
  # pod must be ready for at least 10 seconds before being considered
  # available
  minReadySeconds: 10
  strategy:
    # updates without downtime, other option is Recreate which kills
    # old pods first
    type: RollingUpdate
    rollingUpdate:  # only applicable if type is RollingUpdate
      maxUnavailable: 1  # at most 1 pod can be unavailable during update
      maxSurge: 1  # at most 1 extra pod can be created during update
  selector:
    matchLabels:
      app: next-portfolio  # must match labels in template below
  template:  # template for the pods
    metadata:
      labels:
        app: next-portfolio  # must match selector above
      # used for storing non-identifying information, here I use it for
      # tracking deployment time
      annotations:
        prometheus.io/scrape: "true"  # enables prometheus scraping
        prometheus.io/port: "3000"  # port on which prometheus will scrape
        prometheus.io/path: "/metrics"  # path for prometheus scraping
    spec:  # specification of the pod
      # service account for the pod, if I use default, it has more
      # privileges than needed
      serviceAccountName: next-portfolio-sa
      securityContext:
        fsGroup: 20001  # files created by containers will be owned by this GID
        runAsNonRoot: true  # ensure pod is run as non-root user
        runAsUser: 10001  # run pod as user with UID 10001
        runAsGroup: 30001  # run pod as group with GID 30001
      # automatically mount the service account token, since the app doesnt
      # need to interact with API server, I disable it for security
      automountServiceAccountToken: false
      # time to wait before forcefully killing the pod
      terminationGracePeriodSeconds: 20
      containers:
        # name of the first container, more options include
        # imagePullPolicy, command, args, workingDir, volumeMounts
        - name: next-portfolio
          image: docker.io/nishans0/next-portfolio:v0.0.1
          imagePullPolicy: Always  # pull image always
          ports:
            - containerPort: 3000  # port on which the container is listening
          env:
            # environment variable to set the environment
            - name: NODE_ENV
              value: "production"  # value of the environment variable
          # checks if the app has started, if not, it will be restarted
          startupProbe:
            # HTTP GET request to the root path on port 3000 more options
            # can be protocol, host, scheme
            httpGet:
              path: "/"
              port: 3000
            initialDelaySeconds: 5  # wait 5 seconds before starting probes
            periodSeconds: 10  # probe every 10 seconds
            # after 10 failures, the pod is marked as not started
            failureThreshold: 10
            successThreshold: 1  # after 1 success, the pod is marked as started
          # checks if the app is alive, if not, it will be restarted
          livenessProbe:
            # HTTP GET request to the root path on port 3000 more options
            # can be protocol, host, scheme
            httpGet:
              path: "/"
              port: 3000
            initialDelaySeconds: 15  # wait 15 seconds before starting probes
            periodSeconds: 20  # probe every 20 seconds
            # after 5 failures, the pod is marked as not alive
            failureThreshold: 5
            successThreshold: 1  # after 1 success, pod is marked as alive
          resources:  # resource requests and limits
            requests:  # minimum resources required
              memory: "256Mi"
              cpu: "250m"
              ephemeral-storage: "1Gi"
            limits:  # maximum resources allowed
              memory: "512Mi"
              cpu: "500m"
              ephemeral-storage: "2Gi"
          # security options for the container. I are giving UID and GID
          # to run the container as non-root user for security
          securityContext:
            allowPrivilegeEscalation: false  # do not allow privilege escalation
            capabilities:  # drop all capabilities for security
              drop: ["ALL"]
            # make the root filesystem read-only for security
            readOnlyRootFilesystem: true
          lifecycle:  # hooks for container lifecycle events
            postStart:  # hook to run after the container has started
              exec:  # execute a command
                # simple echo command, can be replaced with any script
                command: ["sh", "-c", "echo Container started"]
            preStop:  # hook to run before the container is stopped
              exec:  # execute a command
                # sleep for 10 seconds to allow in-flight requests to
                # complete
                command: ["sh", "-c", "sleep 10"]

```
**base/service.yaml**
I created a service for the app, it will find our app using spec.selector field which should match with spec.template.metadata.label of the pod in the deployment. The service creates a binding that will bind its 80 port to 3000 port of the pod. It is of type clusterIP which means it can only be accessed inside the cluster.
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: next-portfolio
spec:
  selector: {app: next-portfolio}
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP

```
**\base\pdg.yaml**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: next-portfolio-pdb
  namespace: dev
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: next-portfolio

```
/base/sa.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: next-portfolio-sa
  namespace: dev

```

**base/kustomization.yaml**
This kustomization tells argoCD to use the same deployment from base but use tthe ingress defined in this folder. It also mentions the image to be used which has tag specifically to be used for dev environment. 
```yaml
---
resources:
  - deployment.yaml
  - service.yaml
  - pdb.yaml
  - sa.yaml

```

**/dev/ingress.yaml**
ArgoCD uses this ingress declaration to create an ingress controller that would make external access to the pod hosted in the dev namespace of the cluster. It uses nginx to create the ingress controller. Its specifications contains the rules for the ingress controller to be triggered whcih are the host requested should be portfolio-dev.nishdevops.ord, path /. Once triggered it will forward the traffic to service named next-portfolio on port 80.

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: next-portfolio
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: portfolio-dev.nishdevops.org
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: next-portfolio
                port:
                  number: 80

```
**/dev/kustomization.yaml**
This kustomization tells argoCD to use the same deployment from base but use tthe ingress defined in this folder. It also mentions the image to be used which has tag specifically to be used for dev environment. 
```yaml
---
namespace: dev
resources:
  - ../base
  - ingress.yaml
images:
  - name: docker.io/nishans0/next-portfolio
    newTag: edge

```

**/stage/ingress.yaml and /prod/ingress.yaml**
```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: next-portfolio
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: portfolio-stage.nishdevops.org
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: next-portfolio
                port:
                  number: 80

```

**/prod/kustomization.yaml**
Similar to dev in the resources used, but uses a numbered tag. The newTag will signify that the app for a specific version has passed the dev environment and ready to be staged. We’ve also added patches to the deplyment as example. 
```yaml
---
namespace: prod
resources:
  - ../base
  - ingress.yaml
images:
  - name: docker.io/nishans0/next-portfolio
    newTag: v0.0.1
patches:
  - target:
      kind: Deployment
      name: next-portfolio
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 2
```
**/prod/kustomization.yaml**
```yaml
---
namespace: prod
resources:
  - ../base
  - ingress.yaml
images:
  - name: docker.io/nishans0/next-portfolio
    newTag: v0.0.1
patches:
  - target:
      kind: Deployment
      name: next-portfolio
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 2
```

### 6.4 Pre-Deploy Tests
At this point I have written the yaml files. Before I commit these, I performed the following tests.
#### YAML Lint
```bash
yamllint manifests/
```
**Sample Output**
manifests/overlays\base\deployment.yaml
  1:4       error    wrong new line character: expected \n  (new-lines)
  7:7       error    wrong indentation: expected 4 but found 6  (indentation)
  9:17      warning  missing starting space in comment  (comments)
  10:27     warning  too few spaces before comment: expected 2  (comments)
  19:17     error    trailing spaces  (trailing-spaces)
  56:81     error    line too long (85 > 80 characters)  (line-length)


#### Render Check
Check if the manifest yamls can be successfully rendered/compiled by the cluster later.  
```bash
kustomize build manifests/overlays/dev >/dev/null,  
kustomize build manifests/overlays/base >/dev/null 
kustomize build manifests/overlays/stage >/dev/null 
kustomize build manifests/overlays/prod >/dev/null
```
The above commands should output nothing for successful check.

#### Schema Validation
```bash
kustomize build manifests/overlays/dev | kubeconform --strict --ignore-missing-schemas
```

Output of the first test:

<img width="706" height="199" alt="Screenshot 2025-10-21 103722" src="https://github.com/user-attachments/assets/07f91273-4ef4-4e76-9c03-2198faee8ad8" />


After fixing the issues shown the command gave exit(0) or no output which means our test passed API Schema Validation. 

#### Kube Score
Kube-score checks for best practices /safety net checks. 
Command: `kube-score score deployment.yaml` OR `kustomize build overlays/dev | kube-score score -`


Output of first test:

<img width="804" height="639" alt="Screenshot 2025-10-21 103937" src="https://github.com/user-attachments/assets/8e1f69f1-8fcc-4906-bf37-fd4713b06441" />


#### Policy Tests
I used conftest to test the custom policies I created which are stored in policy folder. Some of the policies I have are as follows.
```bash
kustomize build manifests/overlays/dev | conftest test -
```
Policies enforced:
- No `latest` tag
- Must have `app` label
- No default namespace
- Resource limits required
- Non-privileged containers only

---


## 8. Argo CD Apps
Now that the pre-deploy tests have been completed, I created an ArgoCD app. I accessed the argoCD portal from its LoadBalancerIP (192.168.101.221 in our case).  I created a project named porfolio-apps first with following configurations:

### TIP
*This makes ingress controller update the latest ip addresses used by the load balancer and services in logs.*
```bash
kubectl -n ingress-nginx patch deploy ingress-nginx-controller \
  --type='json' \ 
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--publish-service=$(POD_NAMESPACE)/ingress-nginx-controller"}]'
```
SOURCE REPOSITORIES: https://github.com/nishanau/NextJSPortfolioSite.git

DESTINATIONS: I will only allow the apps in this projects to run in dev, stage and prod namespaces only for now.

After setting up the project, I created an app each for each namespace; dev, stage and prod.  
I can use the GUI or a manifest yaml to set up the apps. 
Here’s the manifest of the next-portfolio-dev app: 

Example app manifest:
```yaml
project: portfolio-apps
source:
  repoURL: https://github.com/nishanau/NextJSPortfolioSite.git
  path: manifests/overlays/dev
  targetRevision: HEAD
destination:
  server: https://kubernetes.default.svc
  namespace: dev
syncPolicy:
  automated:
    prune: true
    selfHeal: true
    enabled: true
  syncOptions:
    - ApplyOutOfSyncOnly=true
    - CreateNamespace=true
  retry:
    limit: 2
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m0s 

```

---

## 8. GitHub Actions (CI/CD)
Initially combined, later split into:
- **CI:** Lint, validate, build, and push Docker image.
- **CD:** Sync updated image tag via ArgoCD.

***Thought Process and Evolution:***
*Initially, during brainstorming and designing the CI-flows, I thought a single flow that would build and push the image would be enough. As I dug deep, I realized, the pre deployment tests for manifests like yamllint, conftest, kube-score, etc need to be tested on every manifest update as well. This meant I would need 2 sets of tests, one for the app code and the other to test the infra code. After this I realized that its better to have separate workflows for app updates and infra code updates. I have a monorepo i.e. the GitOps Repo and the App repo is in a single repo. First, I started creating pre-deploy test flows for infra. The first test flow would install all the test tools in the vm for every flow. This was inefficient so I decided to create and push a container (tools-container) that would install and host all the test tools. Then our infra CI flow will use this container and test the infra code in it. However, if I need to make changes to the container, I would need to manually edit the dockerfile, then build it and push it. So, I thought maybe I should create a flow that will run when the dockerfile is changed. This gave me another insight; this container only has the test tools installed which can be used to test any manifests in any repo and I have many apps that I want to host in the cluster later on. So, if I make this a reusable image then all the repos I want can call it. This prompted me to create a ci-cd-templates repository which would host all reusable workflows for CI, dockerfile and package building. The dockerfile for the tools container is in `ci-images/pre-deploy-test-tools` directory. Any commit that contains change to this dockerfile will call `build-pre-deploy-tools.yml` which will build the latest package and push it to ghcr.io.  With the ci-cd-template repo created, I wanted to standardize my CI for app and infra code as well. Hence, I created  `ci-app.yml`, `ci-manifests.yml` and `ci-gitops-bump.yml` workflows. These are reusable workflows which is used by my **Next Portfolio** app and will be used by apps I deploy in the future. These workflows are the backbone of my scalable CI/CD End-to-End deployment. There is clear separation of concerns between different branches, as the workflows will only work on the files of the specific branch mentioned in the input but the calling workflow.*

### nishanau/ci-cd-templates/.github/workflows/ci-app.yml
This workflow will be called when there is change in the app related code, for example, adding new features, resolving code, etc. Any app can call this flow for its CI and when calling, the calling flow must provide the required inputs as mentioned in this flow for it to function properly. This workflow will containerize, test (automated dev related test like unit tests,etc.), lint, build, push and test the build image for vulnerabilities.

````yaml
# =====================================================================
#  Reusable Workflow: CI - App (Build, Scan, Push to Docker Hub)
# =====================================================================

name: CI - App (Build + Scan + Push)

on:
  workflow_call:
    inputs:
      image_name:
        description: "Full Docker Hub image name (e.g. nishanau/my-app)"
        required: true
        type: string
      context:
        description: "Docker build context"
        default: "."
        type: string
      dockerfile:
        description: "Dockerfile path"
        default: "./Dockerfile"
        type: string
      push_image:
        description: "Whether to push image to Docker Hub"
        default: true
        type: boolean
      run_tests:
        description: "Whether to run app lint/tests"
        default: true
        type: boolean

    secrets:
      DOCKERHUB_USERNAME:
        required: true
      DOCKERHUB_TOKEN:
        required: true

permissions:
  contents: read
  packages: read
  security-events: write

# =====================================================================
# 1️⃣ Lint + Unit Tests
# =====================================================================
jobs:
  lint_test:
    runs-on: ubuntu-latest
    if: ${{ inputs.run_tests }}

    steps:
      - uses: actions/checkout@v4

      - name: Install dependencies
        run: |
          if [ -f package.json ]; then
            npm ci
          elif [ -f requirements.txt ]; then
            pip install -r requirements.txt
          else
            echo "No recognized dependency manifest found"
          fi

      - name: Run tests
        run: |
          if [ -f package.json ]; then
            npm test || true
          elif [ -f pytest.ini ] || [ -d tests ]; then
            pytest || true
          else
            echo "No test suite found; skipping"
          fi

# =====================================================================
# 2️⃣ Build, Push, and Scan
# =====================================================================
  build_scan:
    runs-on: ubuntu-latest
    needs: [lint_test]

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # ----------------------------------------------------------
      # Login to Docker Hub using your secrets
      # ----------------------------------------------------------
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # ----------------------------------------------------------
      # Build and push Docker image
      # ----------------------------------------------------------
      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: ${{ inputs.context }}
          file: ${{ inputs.dockerfile }}
          push: ${{ inputs.push_image }}
          tags: |
            ${{ inputs.image_name }}:sha-${{ github.sha }}
            ${{ inputs.image_name }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true
          labels: |
            org.opencontainers.image.source=${{ github.repository }}
            org.opencontainers.image.revision=${{ github.sha }}

      # ----------------------------------------------------------
      # Scan built image using Trivy
      # ----------------------------------------------------------
      - name: Scan image for vulnerabilities
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ${{ inputs.image_name }}:sha-${{ github.sha }}
          vuln-type: 'os,library'
          severity: 'HIGH,CRITICAL'
          ignore-unfixed: true
          format: 'sarif'
          output: 'trivy-results.sarif'

      # ----------------------------------------------------------
      # Upload scan results to GitHub Security tab
      # ----------------------------------------------------------
      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif
````

### nishanau/ci-cd-templates/.github/workflows/ci-manifests.yml
This reusable workflow will use the tools test container that I created to test the manifests genereated. It will then test the manifests in the container and outputs the results. Calling workflows can use inputs to specify different variants of tests that can be run. It also publishes the results to Code Scanning Alerts which can be viewed from Security/Code Scanner tab for security related issues.
````yaml
# Reusable workflow that validates Kubernetes manifests before deploy
name: ci-manifests

on:
  workflow_call:
    inputs:
      overlay_path:                     # Path to the overlay folder to validate
        description: "Path to overlay (e.g., manifests/overlays/dev)"
        required: true
        type: string
      tools_ref:                        # Tag or digest for the pre-deploy tools image
        description: "Tag or digest for tools image"
        required: false
        type: string
        default: "latest"               # Prefer pinning by digest in callers for immutability
      policies_path:                    # Where your Rego policies live (optional)
        description: "Path to conftest policies (rego)"
        required: false
        type: string
        default: "policies"
      kubeconform_flags:                # Extra flags for kubeconform (schema validation)
        description: "Extra kubeconform flags"
        required: false
        type: string
        default: "--strict --ignore-missing-schemas"
      fail_on_warn:                     # Make kube-score warnings fail the job
        description: "Fail on kube-score warnings"
        required: false
        type: boolean
        default: true

permissions:
  contents: read            # checkout needs this
  packages: read            # pull the tools image from GHCR if needed
  security-events: write    # upload SARIF to Code Scanning

# Prevent duplicate runs on the same ref; cancel older in-flight runs
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  validate:
    name: Validate manifests
    runs-on: ubuntu-latest

    # Run all steps inside the shared tools container (has kustomize, kubeconform, etc.)
    container:
      image: ghcr.io/nishanau/ci-cd-templates/pre-deploy-tools:${{ inputs.tools_ref }}
      credentials:
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    timeout-minutes: 20

    steps:
      # 1) Pull the repo contents (manifests and optional policies)
      - name: Checkout repo
        uses: actions/checkout@v4

      # 2) Print tool versions into the job summary for auditability
      - name: Tool versions
        run: toolbox-versions >> $GITHUB_STEP_SUMMARY || true

      # 3) Lint raw YAML for indentation, formatting, and obvious syntax errors
      - name: YAML lint
        run: yamllint -d relaxed "${{ inputs.overlay_path }}"

      # 4) Render the overlay to final deployable YAML and persist as build.yaml for later checks
      - name: Render manifests (kustomize)
        shell: bash
        run: |
          kustomize build "${{ inputs.overlay_path }}" | tee build.yaml

      # 5) Validate rendered YAML against Kubernetes/OpenAPI schemas
      #    --strict: fail on unknown/invalid fields
      #    --ignore-missing-schemas: skip resources without published schemas (tune per repo)
      - name: Schema validate (kubeconform)
        run: kubeconform ${{ inputs.kubeconform_flags }} < build.yaml

      # 6) Run org policy tests (Rego). Enforce labels, probes, resources, securityContext, etc.
      - name: Policy tests (conftest)
        shell: bash
        run: |
          if [ -d "${{ inputs.policies_path }}" ]; then
            conftest test "${{ inputs.overlay_path }}" --policy "${{ inputs.policies_path }}" --output table
          else
            echo "No policies directory '${{ inputs.policies_path }}' found. Skipping."
          fi

      # 7) Heuristic best-practice checks; optionally fail on warnings for stronger gates
      - name: kube-score best-practices
        shell: bash
        run: |
          if ${{ inputs.fail_on_warn }}; then
            kustomize build ${{ inputs.overlay_path }} | kube-score score --exit-one-on-warning -
          else
            kustomize build ${{ inputs.overlay_path }} | kube-score score -
          fi

      # 8) IaC security scan; export SARIF so findings show in GitHub Security tab
      - name: IaC scan (Checkov → SARIF)
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: ${{ inputs.overlay_path }}
          output_format: sarif
          output_file_path: checkov.sarif
          skip_check: CKV_K8S_43,CKV_K8S_16

      # 9) Publish SARIF results to repository “Code scanning alerts”
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: checkov.sarif

      # 10) Persist key artifacts (rendered YAML + SARIF) for 21 days for audit and debugging
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: manifests-ci-${{ github.run_id }}
          path: |
            build.yaml
            checkov.sarif
          retention-days: 21

      # 11) Human-readable run summary
      - name: Summary
        if: always()
        run: |
          {
            echo "### Manifests CI"
            echo "- Overlay: \`${{ inputs.overlay_path }}\`"
            echo "- kubeconform flags: \`${{ inputs.kubeconform_flags }}\`"
            echo "- kube-score fail on warn: \`${{ inputs.fail_on_warn }}\`"
          } >> $GITHUB_STEP_SUMMARY
````

### nishanau/ci-cd-templates/.github/workflows/ci-gitops-bump.yml
This worflow is called in our next-portfolio app when workflows calling both ci-app and ci-manifests pass or skip(for various scenarios handled by the calling workflow) but not fail. If any of the 2 flows fail then this flow wont be called. The main function of this flow is to change the image tag of the kustomization.yaml in the overlays of branch mentioned in input (dev, stage, prod/main). This is what triggers the ArgoCD to resync and deploy the latest change to the cluster. There are different scenarios that determine if a tag bump is necessary.
````yaml
# =====================================================================
#  Reusable Workflow: CI - GitOps Bump
#  ---------------------------------------------------------------------
#  Purpose:
#    - Updates image tag (newTag) in the GitOps repo's kustomization.yaml
#    - Commits and pushes the change automatically
#    - Used after a successful app image build in the main pipeline
# =====================================================================

name: CI - GitOps Bump

on:
  workflow_call:
    inputs:
      gitops_repo:
        description: "Target GitOps repository (e.g., nishanau/gitops-infra-k8s)"
        required: true
        type: string
      gitops_path:
        description: "Path to the overlay kustomization.yaml file"
        required: true
        type: string
      image_name:
        description: "Full image name (e.g., nishanau/nextjs-portfolio)"
        required: true
        type: string
      image_tag:
        description: "New image tag (e.g., sha-b29c372...)"
        required: true
        type: string
    secrets:
      gitops_pat:
        required: true   # Personal Access Token (PAT) for repo write access

permissions:
  contents: write       # Needed for committing and pushing changes

jobs:
  bump:
    runs-on: ubuntu-latest

    steps:
      # ------------------------------------------------------------
      # 1. Checkout the GitOps repository where manifests live
      # ------------------------------------------------------------
      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: ${{ inputs.gitops_repo }}     # e.g. nishanau/NextJSPortfolioSite
          token: ${{ secrets.gitops_pat }}          # PAT forwarded from caller workflow
          persist-credentials: true                 # keep credentials for git push
          fetch-depth: 0                            # full history to allow push
          ref: ${{ github.ref_name }}               # checkout the branch that triggered the workflow

      # ------------------------------------------------------------
      # 2. Install yq for YAML editing
      # ------------------------------------------------------------
      - name: Install yq
        run: |
          sudo wget -q https://github.com/mikefarah/yq/releases/download/v4.44.2/yq_linux_amd64 -O /usr/local/bin/yq
          sudo chmod +x /usr/local/bin/yq
          echo "yq version:"
          yq --version

      # ------------------------------------------------------------
      # 3. Update the image tag in kustomization.yaml
      # ------------------------------------------------------------
      - name: Update Image Tag in kustomization.yaml
        run: |
          echo "Updating image '${{ inputs.image_name }}' to tag '${{ inputs.image_tag }}'"
          # Replace .newTag value of matching image
          yq -i '
            (.images[] | select(.name == "'"${{ inputs.image_name }}"'") | .newTag)
              = "'"${{ inputs.image_tag }}"'"
          ' "${{ inputs.gitops_path }}"
          echo "Updated file contents:"
          cat "${{ inputs.gitops_path }}"

      # ------------------------------------------------------------
      # 4. Commit and push the change
      # ------------------------------------------------------------
      - name: Commit and push changes
        run: |
          # Configure bot identity for commit
          git config user.name "ci-bot"
          git config user.email "ci-bot@users.noreply.github.com"

          # Stage modified file
          git add "${{ inputs.gitops_path }}"

          # Commit only if there are changes; otherwise skip
          git commit -m "bump image ${{ inputs.image_name }} to ${{ inputs.image_tag }} [skip ci]" || echo "No changes to commit"

          # Push back to the same repo branch that triggered the flow
          git push origin HEAD:${{ github.ref_name }}
````

### nishanau/NextJSPortfolioSite/.github/workflows/dev-ci.yml
This is the app specific CI workflow for our next-portfolio app. This is where our reusable workflows will be called. Firstly, it detects the files that have changed and based on the output of this detection it will decide to run Job2 (ci-app) and/or Job3 (ci-manifests) or not. If any app related files changed, it will run Job2, which will use ci-app.yml and builds, tests, and publishes the latest image with the commit sha as the tag. If any files related to manifests or policies change then Job 2 (ci-manifests) will call ci-manifests.yml which will test the manifests and gives the output. If both of these tests pass then it will proceed with gitops version bump. There are some scenarios that decide whether this version bump gets triggered or not given in the table below.

### 🧩 Unified Tag Bump Scenario Table

| Branch | Changed Files | App CI Result | Manifests CI Result | Tag Source Used | Tag Bump Happens? | Outcome / Behavior |
|:-------|:---------------|:---------------|:--------------------|:----------------|:------------------|:--------------------|
| **dev** | App code only (`src/**`, `Dockerfile`, etc.) | ✅ success | ⚠️ skipped | Current commit SHA | ✅ Yes | New image built and pushed. `manifests/overlays/dev/kustomization.yaml` updated with new SHA tag. |
| **dev** | App + manifests | ✅ success | ✅ success | Current commit SHA | ✅ Yes | Both app and infra validated; tag updated to new image version. |
| **dev** | Only manifests | ⚠️ skipped | ✅ success | — | ❌ No | No rebuild; manifests validated but tag unchanged. |
| **dev** | App build failed | ❌ failed | — | — | ❌ No | Protects environment from broken builds. |
| **dev** | No changes (manual run or re-run) | ⚠️ skipped | ⚠️ skipped | — | ❌ No | Nothing to commit or push. |
| **dev** | Manual rollback (tag edited manually) | ⚠️ skipped | ✅ success | Older SHA | ⚠️ Yes (manual) | Argo CD automatically rolls back to previous image version. |
| **stage** | Manifest overlay updated | ⚠️ skipped | ✅ success | Tag from **dev** overlay | ✅ Yes | Copies latest verified tag from dev → updates `stage` overlay. |
| **stage** | No manifest changes (promotion commit only) | ⚠️ skipped | ⚠️ skipped | Tag from **dev** overlay | ✅ Yes | Uses last known good image from dev to redeploy. |
| **stage** | Invalid manifests/policy errors | ⚠️ skipped | ❌ failed | — | ❌ No | Tag bump blocked due to failed validation. |
| **stage** | App code changed (rare) | ✅ success | ✅ success | Current commit SHA | ✅ Yes | Both succeed → tag updated, though unusual for stage. |
| **stage** | App CI failed | ❌ failed | — | — | ❌ No | Prevents failed app builds from promoting. |
| **stage** | Manual rollback (tag edited manually) | ⚠️ skipped | ⚠️ skipped | Older SHA | ⚠️ Yes (manual) | Argo CD redeploys previous version automatically. |
| **prod** | Promotion from stage (manifest change) | ⚠️ skipped | ✅ success | Tag from **stage** overlay | ✅ Yes | Bumps `prod` overlay with last validated stage tag. |
| **prod** | No manifest change (manual redeploy) | ⚠️ skipped | ⚠️ skipped | Tag from **stage** overlay | ✅ Yes | Redeploys existing prod image. |
| **prod** | Invalid manifests/policy errors | ⚠️ skipped | ❌ failed | — | ❌ No | Fails validation, tag unchanged. |
| **prod** | App rebuilt on prod (not recommended) | ✅ success | ✅ success | Current commit SHA | ✅ Yes | Works but violates GitOps best practice. |
| **prod** | App CI failed | ❌ failed | — | — | ❌ No | No bump; deployment halted. |
| **prod** | Manual rollback (tag edited manually) | ⚠️ skipped | ⚠️ skipped | Older SHA | ⚠️ Yes (manual) | Argo CD rolls back to known good version. |

---

### 🧠 Summary

| Environment | Tag Source | Automatic Build? | Typical Trigger | Behavior |
|:-------------|:-----------|:-----------------|:----------------|:----------|
| **dev** | `sha-${{ github.sha }}` | ✅ Yes | Code push | Builds, pushes, updates `dev` overlay |
| **stage** | From `dev` overlay | ❌ No | Merge/promotion | Copies last known good tag |
| **prod** | From `stage` overlay | ❌ No | Merge/promotion | Copies last verified tag |
| **any** | Manual edit | ❌ No | Tag revert | Argo CD auto-syncs rollback |


````yaml
# =====================================================================
#  Workflow: Dev CI (App + Manifests + GitOps Bump)
#  ---------------------------------------------------------------------
#  Purpose:
#    - Run App CI when app code/config changes
#    - Run Manifests CI when infra YAML/policy changes
#    - Bump image tag in manifests when both succeed
# =====================================================================

name: Dev CI (App + Manifests + GitOps Bump)

on:
  push:
    branches: [dev, stage, master]
    paths:
      - 'src/**'
      - 'public/**'
      - 'package.json'
      - 'package-lock.json'
      - 'Dockerfile'
      - '.dockerignore'
      - 'next.config.mjs'
      - 'eslint.config.mjs'
      - 'jsconfig.json'
      - 'tsconfig.json'
      - 'manifests/**'
      - 'policy/**'
      - '.github/workflows/dev-ci.yml'

permissions:
  contents: write
  packages: write
  security-events: write

jobs:
  # =====================================================================
  #  JOB 0: DETECT CHANGED FILES
  # =====================================================================
  changes:
    runs-on: ubuntu-latest
    outputs:
      app: ${{ steps.filter.outputs.app }}
      manifests: ${{ steps.filter.outputs.manifests }}
    steps:
      - uses: actions/checkout@v4

      - uses: dorny/paths-filter@v3
        id: filter
        with:
          base: ${{ github.event.before }}
          ref: ${{ github.sha }}
          filters: |
            app:
              - 'src/**'
              - 'public/**'
              - 'package.json'
              - 'package-lock.json'
              - 'Dockerfile'
              - '.dockerignore'
              - 'next.config.mjs'
              - 'eslint.config.mjs'
              - 'jsconfig.json'
              - 'tsconfig.json'
            manifests:
              - 'manifests/**'
              - 'policy/**'

  # =====================================================================
  #  JOB 1: APP CI - Build, Test, Scan, Push Image
  # =====================================================================
  app-ci:
    needs: [changes]
    if: ${{ needs.changes.outputs.app == 'true' && github.ref_name == 'dev' }}
    uses: nishanau/ci-cd-templates/.github/workflows/ci-app.yml@main
    with:
      image_name: nishans0/next-portfolio
      context: .
      dockerfile: ./Dockerfile
      push_image: true
      run_tests: true
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}

  # =====================================================================
  #  JOB 2: MANIFESTS CI - Validate YAML & Policies
  # =====================================================================
  manifests-ci:
    needs: [changes]
    if: ${{ needs.changes.outputs.manifests == 'true' }}
    uses: nishanau/ci-cd-templates/.github/workflows/ci-manifests.yml@main
    with:
      overlay_path: manifests/overlays/${{ github.ref_name }}
      tools_ref: "latest"
      policies_path: policy
      kubeconform_flags: "--strict --ignore-missing-schemas"
      fail_on_warn: false
    secrets: inherit

  # =====================================================================
  #  JOB 3A: RESOLVE IMAGE TAG
  # =====================================================================
  resolve-tag:
    runs-on: ubuntu-latest
    needs: [app-ci, manifests-ci]
    if: ${{ always() && (
      (github.ref_name == 'dev' && needs.app-ci.result == 'success') ||
      (
        github.ref_name != 'dev' &&
        ( !contains(needs.*, 'app-ci') || needs.app-ci.result == 'success' || needs.app-ci.result == 'skipped' ) &&
        (needs.manifests-ci.result == 'success' || needs.manifests-ci.result == 'skipped')
      )) }}
    steps:
      - uses: actions/checkout@v4

      - id: resolve_tag
        run: |
          if [[ "${{ github.ref_name }}" == "dev" ]]; then
            TAG="sha-${{ github.sha }}"
          elif [[ "${{ github.ref_name }}" == "stage" ]]; then
            TAG=$(yq e '.images[] | select(.name=="docker.io/nishans0/next-portfolio") | .newTag' manifests/overlays/dev/kustomization.yaml)
          elif [[ "${{ github.ref_name }}" == "master" || "${{ github.ref_name }}" == "prod" ]]; then
            TAG=$(yq e '.images[] | select(.name=="docker.io/nishans0/next-portfolio") | .newTag' manifests/overlays/stage/kustomization.yaml)
          fi
          echo "tag=$TAG" >> $GITHUB_OUTPUT

    outputs:
      tag: ${{ steps.resolve_tag.outputs.tag }}

  # =====================================================================
  #  JOB 3B: GITOPS BUMP - Update Image Tag in Manifest
  # =====================================================================
  bump-gitops:
    needs: [app-ci, manifests-ci, resolve-tag]
    if: ${{ always() && (
      (github.ref_name == 'dev' && needs.app-ci.result == 'success') ||
      (
        github.ref_name != 'dev' &&
        ( !contains(needs.*, 'app-ci') || needs.app-ci.result == 'success' || needs.app-ci.result == 'skipped' ) &&
        (needs.manifests-ci.result == 'success' || needs.manifests-ci.result == 'skipped')
      )) }}
    uses: nishanau/ci-cd-templates/.github/workflows/ci-gitops-bump.yml@main
    with:
      gitops_repo: nishanau/NextJSPortfolioSite
      gitops_path: manifests/overlays/${{ github.ref_name }}/kustomization.yaml
      image_name: docker.io/nishans0/next-portfolio
      image_tag: ${{ needs.resolve-tag.outputs.tag }}
    secrets:
      gitops_pat: ${{ secrets.GITOPS_PAT }}
````

## Final CI/CD (End-to-End Flow)
![CI/CD + GitOps Architecture](./public/enterprise_cicd_k8s/cicd_flow.svg)

## 9. Secrets Management with Doppler + External Secrets Operator

### 9.1 Why Doppler + ESO?

I use **Doppler** (cloud secret manager) + **External Secrets Operator (ESO)** to securely manage Kubernetes secrets while preserving GitOps principles. This approach separates secret storage from deployment configuration.

**Previous Approach (Manual Secret Injection):**
- ❌ Secrets manually created with `kubectl`
- ❌ Not reproducible or auditable
- ❌ No GitOps compliance
- ❌ Difficult to rotate or update

**Doppler + ESO Benefits:**
- ✅ Secrets never stored in Git (not even encrypted)
- ✅ ArgoCD never sees secret values
- ✅ Centralized secret management with audit logs
- ✅ Easy rotation without code changes
- ✅ Separation of concerns: deployment vs. secrets access
- ✅ Follows ArgoCD best practices

---

### 9.2 How It Works

**Architecture Flow:**
```
1. Secrets stored in Doppler (cloud)
   ↓
2. ESO authenticates with Doppler using bootstrap token
   ↓
3. ESO watches ExternalSecret resources
   ↓
4. ESO fetches secret values from Doppler
   ↓
5. ESO creates Kubernetes Secrets in target namespace
   ↓
6. Application pods consume secrets (standard K8s pattern)
```

**Key Components:**

**CRDs (Custom Resource Definitions):**
- Extend Kubernetes to understand new resource types
- `ExternalSecret` and `SecretStore` are custom resources
- Schema definitions only—don't perform any actions
- Never should be pruned (would delete all resources of that type)

**ESO Operator (Controller):**
- Actual application Pod that acts on custom resources
- Connects to Doppler, fetches secrets, creates K8s Secrets
- Runs continuously in `external-secrets` namespace
- Can be safely upgraded/redeployed

**SecretStore:**
- Connection configuration for secret backend (Doppler)
- Defines authentication method and project/environment
- Namespace-scoped for security isolation

**ExternalSecret:**
- Declares what secret to fetch and where to put it
- Maps Doppler secret keys to Kubernetes secret keys
- Refreshes periodically (every 15 minutes) to stay in sync

---

### 9.3 Setup Process

#### **Step 1: Doppler Account Setup**

1. Created account at https://doppler.com (free tier)
2. Created project: `baremetal-k8s-project`
3. Selected environment: `prd` (production)
4. Added secret:
   - Name: `CLOUDFLARED_TUNNEL_TOKEN`
   - Value: Actual tunnel token from Cloudflare
5. Generated service token:
   - Name: `kubernetes-prod`
   - Environment: `prd`
   - Access: `Read` (least privilege)
   - Token format: `dp.st.prd.xxx...`

#### **Step 2: External Secrets Operator Installation**

**Created separate infra repository:** `nishanau/infra-gitops`

**File structure:**
```
infra-gitops/
├── argocd/
│   └── overlays/prod/
│       ├── infra-project.yaml         # AppProject for infrastructure apps
│       ├── external-secrets-app.yaml  # ESO CRDs + Operator
│       ├── cloudflared-app.yaml       # Cloudflared app
│       └── kustomization.yaml
│
├── external-secrets-operator/
│   └── base/
│       ├── namespace.yaml
│       └── kustomization.yaml         # Includes CRDs bundle
│
└── cloudflared/
    ├── base/
    │   ├── deployment.yaml
    │   ├── secret-store.yaml          # Doppler connection
    │   ├── external-secret.yaml       # Secret definition
    │   └── kustomization.yaml
    └── overlays/prod/
```

**Why separate repository?**
- Infrastructure code separated from application code
- Reusable across multiple applications
- Better access control (infra team vs. dev team)

#### **Step 3: ArgoCD Infrastructure Project**

**infra-gitops/argocd/overlays/prod/infra-project.yaml:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: infra
  namespace: argocd
spec:
  description: Infrastructure applications
  sourceRepos:
    - 'https://github.com/nishanau/infra-gitops.git'
    - 'https://charts.external-secrets.io'  # Helm chart repo
    - '*'
  destinations:
    - namespace: '*'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
```

**Purpose:**
- Groups infrastructure apps under one project
- Allows Helm chart repositories (needed for ESO)
- Better organization than using `default` project

#### **Step 4: ESO CRDs and Operator Apps**

**infra-gitops/argocd/overlays/prod/external-secrets-app.yaml:**
```yaml
---
# App 1: CRDs (never pruned)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets-crds
  namespace: argocd
spec:
  project: infra
  source:
    repoURL: https://github.com/nishanau/infra-gitops
    targetRevision: HEAD
    path: external-secrets-operator/base
  destination:
    server: https://kubernetes.default.svc
    namespace: external-secrets
  syncPolicy:
    automated:
      prune: false  # ✅ Never delete CRDs!
      selfHeal: true

---
# App 2: Operator (can be pruned)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets-operator
  namespace: argocd
spec:
  project: infra
  source:
    repoURL: https://charts.external-secrets.io
    chart: external-secrets
    targetRevision: 0.9.11
    helm:
      releaseName: external-secrets
  destination:
    server: https://kubernetes.default.svc
    namespace: external-secrets
  syncPolicy:
    automated:
      prune: true  # ✅ Safe to prune operator
      selfHeal: true
```

**Why two separate Applications?**
- **CRDs**: Protected from deletion (`prune: false`)
- **Operator**: Can be safely upgraded/redeployed
- Separation follows Kubernetes operator best practices

#### **Step 5: Cloudflared SecretStore**

**infra-gitops/cloudflared/base/secret-store.yaml:**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: doppler-secret-store
  namespace: cloudflared
spec:
  provider:
    doppler:
      project: baremetal-k8s-project  # Doppler project
      config: prd                      # Environment
      auth:
        secretRef:
          dopplerToken:
            name: doppler-token-auth   # Bootstrap secret
            key: dopplerToken
```

**Purpose:**
- Defines HOW to connect to Doppler
- References bootstrap token (created manually)
- Namespace-scoped for security isolation

#### **Step 6: Cloudflared ExternalSecret**

**infra-gitops/cloudflared/base/external-secret.yaml:**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: cloudflared-credentials
  namespace: cloudflared
spec:
  refreshInterval: 15m  # Sync every 15 minutes
  
  secretStoreRef:
    name: doppler-secret-store
    kind: SecretStore
  
  target:
    name: cloudflared-credentials
    creationPolicy: Owner  # ESO manages lifecycle
    template:
      type: Opaque
  
  data:
    - secretKey: tunnel-token       # K8s secret key
      remoteRef:
        key: CLOUDFLARED_TUNNEL_TOKEN  # Doppler key
```

**Purpose:**
- Declares WHAT secret to fetch
- Maps Doppler keys to Kubernetes keys
- ESO creates and manages the actual Secret

#### **Step 7: Bootstrap Token (Manual Step)**

```bash
# One-time manual secret creation
kubectl create secret generic doppler-token-auth \
  --from-literal=dopplerToken="dp.st.prd.xxx..." \
  -n cloudflared
```

**Why manual?**
- Bootstrap problem: ESO needs this to authenticate
- Not stored in Git (security)
- Created once during initial setup

---

### 9.4 Deployment Flow

**Order of operations:**
```
1. Apply ArgoCD infra project
   ↓
2. Deploy ESO CRDs (must be first)
   ↓
3. Deploy ESO Operator (needs CRDs)
   ↓
4. Create bootstrap token (kubectl)
   ↓
5. Deploy cloudflared app
   ↓
6. ESO fetches secret from Doppler
   ↓
7. ESO creates cloudflared-credentials Secret
   ↓
8. Cloudflared pods start with secret
```

**Verification:**
```bash
# Check ESO is running
kubectl get pods -n external-secrets

# Check SecretStore is valid
kubectl get secretstore -n cloudflared
# Output: doppler-secret-store   Valid   XX

# Check ExternalSecret synced
kubectl get externalsecret -n cloudflared
# Output: cloudflared-credentials   SecretSynced   XX

# Verify secret was created
kubectl get secret cloudflared-credentials -n cloudflared

# Check cloudflared pod
kubectl get pods -n cloudflared
```

---

### 9.5 Security Benefits

| Aspect | Traditional Approach | Doppler + ESO |
|--------|---------------------|---------------|
| **Secret Storage** | In Git (encrypted) | In Doppler (cloud) |
| **ArgoCD Access** | Sees encrypted values | Never sees secrets |
| **Rotation** | Update Git, rebuild | Update Doppler, auto-syncs |
| **Audit Trail** | Git commits only | Doppler logs all access |
| **Access Control** | Git permissions | Separate secret management |
| **Blast Radius** | Git compromise = all secrets | Token compromise = limited scope |

**Key Principles:**
- ✅ Secrets never in Git (not even encrypted)
- ✅ Principle of least privilege (read-only service token)
- ✅ Separation of concerns (deployment vs. secrets)
- ✅ Audit trail (who accessed what, when)
- ✅ Easy rotation (change in Doppler, auto-syncs)

---

## 10. Cloudflared Tunnel Configuration

I use **Cloudflare Tunnel** to securely expose cluster services to the internet without opening firewall ports. The tunnel establishes an outbound-only connection to Cloudflare's edge network.

**File Structure:**
```
cloudflared/
├── base/
│   ├── deployment.yaml
│   ├── secret-store.yaml
│   ├── external-secret.yaml
│   └── kustomization.yaml
└── overlays/
    └── prod/
```

### infra-gitops/cloudflared/base/deployment.yaml
````yaml
# ============================================================
# Deployment: cloudflared
# Purpose:
#   Runs a Cloudflare Tunnel connector inside the cluster.
#   Establishes a secure outbound-only connection to Cloudflare’s edge,
#   eliminating the need for public ingress or exposed ports.
# ============================================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: cloudflared
  labels:
    app: cloudflared

spec:
  # ------------------------------------------------------------
  # Only one replica is required; Cloudflare handles HA on its end.
  # You can scale to multiple replicas for redundancy if desired.
  # ------------------------------------------------------------
  replicas: 1

  selector:
    matchLabels:
      app: cloudflared

  template:
    metadata:
      labels:
        app: cloudflared

    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:latest
          imagePullPolicy: Always

          # ----------------------------------------------------
          # Run the connector in token-based mode.
          # "--no-autoupdate" prevents background updates, keeping
          # image immutability for CI/CD control.
          # ----------------------------------------------------
          args:
            - tunnel
            - --no-autoupdate
            - --metrics
            - 0.0.0.0:2000
            - --loglevel
            - info
            - run
            - --token
            - $(TUNNEL_TOKEN)

          # ----------------------------------------------------
          # Inject the Cloudflare-issued tunnel token
          # securely from Kubernetes Secret.
          # ----------------------------------------------------
          env:
            - name: TUNNEL_TOKEN
              valueFrom:
                secretKeyRef:
                  name: cloudflared-credentials   # must exist in same namespace
                  key: tunnel-token               # token field name inside secret

          # ----------------------------------------------------
          # Health checks for self-healing and readiness signaling.
          # Cloudflared exposes a /ready endpoint on port 2000.
          # ----------------------------------------------------
          livenessProbe:
            httpGet:
              path: /ready
              port: 2000
            initialDelaySeconds: 10
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /ready
              port: 2000
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3

          # ----------------------------------------------------
          # Resource allocation:
          #   Requests ensure minimal guaranteed CPU/memory.
          #   Limits prevent runaway resource consumption.
          # Cloudflared is lightweight and stable within these bounds.
          # ----------------------------------------------------
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 256Mi

          # ----------------------------------------------------
          # Optionally expose /metrics (port 2000) for Prometheus.
          # This line is harmless even if Prometheus is absent.
          # ----------------------------------------------------
          ports:
            - containerPort: 2000
              name: metrics

      # --------------------------------------------------------
      # Restart policy: always keep the tunnel alive.
      # --------------------------------------------------------
      restartPolicy: Always
````

### infra-gitops/cloudflared/base/configmap.yaml


````yaml
# ------------------------------------------------------------
# ConfigMap: cloudflared-config
# Purpose:
#   Holds the tunnel routing configuration for all apps.
#   Each hostname entry maps a public domain to the internal
#   Kubernetes service (typically the ingress controller).
# ------------------------------------------------------------
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudflared-config
  namespace: cloudflared
data:
  # File name inside container (cloudflared expects config.yaml)
  config.yaml: |
    # Unique identifier for the tunnel (shown in Cloudflare dashboard)
    tunnel: shared-tunnel

    # Path where credentials will be mounted in the container
    credentials-file: /etc/cloudflared/creds/credentials.json

    # --------------------------------------------------------
    # Ingress rules — top to bottom evaluation order.
    # Each rule defines a Cloudflare hostname and the local
    # service that receives that traffic.
    # --------------------------------------------------------
    ingress:
      - hostname: portfolio.nishdevops.org
        service: http://ingress-nginx-controller.ingress-nginx.svc.cluster.local:80
      - hostname: portfolio-stage.nishdevops.org
        service: http://ingress-nginx-controller.ingress-nginx.svc.cluster.local:80

      # Fallback route — returns 404 for unknown hostnames
      - service: http_status:404
````

### infra-gitops/cloudflared/base/secret-store.yaml
````yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: doppler-secret-store
  namespace: cloudflared
spec:
  provider:
    doppler:
      project: baremetal-k8s-project
      config: prd
      auth:
        secretRef:
          dopplerToken:
            name: doppler-token-auth
            key: dopplerToken

````

### infra-gitops/cloudflared/base/external-secret.yaml
````yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: cloudflared-credentials
  namespace: cloudflared
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: doppler-secret-store
    kind: SecretStore
  target:
    name: cloudflared-credentials
    creationPolicy: Owner
    template:
      type: Opaque
  data:
    - secretKey: tunnel-token
      remoteRef:
        key: CLOUDFLARED_TUNNEL_TOKEN
````


---

**Outcome:**
A reproducible, security-conscious, and automation-ready Kubernetes deployment pipeline aligning with industry standards.
