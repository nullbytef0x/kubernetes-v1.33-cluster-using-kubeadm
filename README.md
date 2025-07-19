## Kubernetes 1.33 Cluster Setup on Ubuntu 22.04 Live Server and Deploying PHP Guestbook application with Redis

### Prerequisites

- Ubuntu 22.04 LTS installed on all nodes.
- Access to the internet.
- User with sudo privileges.

### Step-by-Step Installation

### Kubernetes 1.33 Cluster Setup

### Step 1: Disable Swap on All Nodes

```jsx
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Step 2: Enable IPv4 Packet Forwarding

sysctl params required by setup, params persist across reboots

```jsx
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
```

Apply sysctl params without reboot
```jsx
sudo sysctl --system
```

### Step 3: Verify IPv4 Packet Forwarding

```jsx
sysctl net.ipv4.ip_forward
```

### Step 4: Install containerd

```jsx
wget https://github.com/containerd/containerd/releases/download/v2.1.3/containerd-2.1.3-linux-amd64.tar.gz
tar Cxzvf /usr/local containerd-2.1.3-linux-amd64.tar.gz
```

### Step 5:  Modify containerd Configuration for systemd Support

```jsx
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir -p /usr/local/lib/systemd/system/
sudo mv containerd.service /usr/local/lib/systemd/system/
```

### Step 6:  Restart the daemon and enable containerd

```jsx
systemctl daemon-reload
systemctl enable --now containerd
```

### Step 7:  	Installing runc

```jsx
wget https://github.com/opencontainers/runc/releases/download/v1.3.0/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc
```

### Step 8:  Installing CNI plugins

```jsx
wget https://github.com/containernetworking/plugins/releases/download/v1.7.1/cni-plugins-linux-amd64-v1.7.1.tgz
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.7.1.tgz
```

### Step 9:  Install kubeadm, kubelet, and kubectl

```jsx
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

### Step 10:  Initialize the cluster (Master Node Only)

```jsx
sudo kubeadm config images pull
sudo kubeadm init --control-plane-endpoint=<master_node_IP> --pod-network-cidr=<pod_network>

Example: sudo kubeadm init --control-plane-endpoint=192.168.8.106 --pod-network-cidr=10.244.0.0/16
```

```yaml
After Initialzing the Cluster Connect to it and apply the CNI yaml (We're using Calico CNI in this guide):

#To start using your cluster, you need to run the following as a regular user:

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#Alternatively, if you are the root user, you can run:

export KUBECONFIG=/etc/kubernetes/admin.conf
```

```yaml
#Apply the CNI YAML

curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml -O

# Fix the pod CIDR to match your setup
sed -i 's|192.168.0.0/16|<your_CIDR> |g' calico.yaml

Example:
	sed -i 's|192.168.0.0/16|10.244.0.0/16|g' calico.yaml

# Apply the manifest
kubectl apply -f calico.yaml
```

### Step 11: Connect worker nodes to the master node

```jsx
Run the command generated after initializing the master node on each worker node. For example:

kubeadm join 192.168.8.106:6443 --token kv1x5t.k3oskrdeiya2cwv2 --discovery-token-ca-cert-hash sha256:688cc5fb0b54c1b6047d499eed6ac0081d4805e69d75745929ca23389269015c
```

### Deploy a Sample Three-Tier Web Application

** In Master Node**

### Step 1:  Deploy Redis Leader

```yaml
nano redis-leader-deployment.yaml

# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-leader
  labels:
    app: redis
    role: leader
    tier: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        role: leader
        tier: backend
    spec:
      containers:
      - name: leader
        image: "docker.io/redis:6.0.5"
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379
```

```yaml
kubectl apply -f redis-leader-deployment.yaml

kubectl get pods
```

### **Step 2: Creating the Redis leader Service**

```yaml
nano redis-leader-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: redis-leader
  labels:
    app: redis
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
    role: leader
```

```yaml
kubectl apply -f redis-leader-service.yaml
```

### Step 3: Deploy the Redis Followers

```yaml
nano redis-follower-deployment.yaml

# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-follower
  labels:
    app: redis
    role: follower
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        role: follower
        tier: backend
    spec:
      containers:
      - name: follower
        image: us-docker.pkg.dev/google-samples/containers/gke/gb-redis-follower:v2
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379
```

```yaml
kubectl apply -f redis-follower-deployment.yaml
```

### Step 4: Deploy Redis Follower Service

```yaml
nano redis-follower-service.yaml

# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: v1
kind: Service
metadata:
  name: redis-follower
  labels:
    app: redis
    role: follower
    tier: backend
spec:
  ports:
    # the port that this service should serve on
  - port: 6379
  selector:
    app: redis
    role: follower
    tier: backend
```

```yaml
kubectl apply -f redis-follower-service.yaml

kubectl get service 

The response should be similar to this: 
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP    3d19h
redis-follower   ClusterIP   10.110.162.42   <none>        6379/TCP   9s
redis-leader     ClusterIP   10.103.78.24    <none>        6379/TCP   6m10s
```

### Step 5: Set up and Expose the Guestbook Frontend

```yaml

nano frontend-deployment.yaml

# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
        app: guestbook
        tier: frontend
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: us-docker.pkg.dev/google-samples/containers/gke/gb-frontend:v5
        env:
        - name: GET_HOSTS_FROM
          value: "dns"
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 80

```

```yaml
kubectl apply -f frontend-deployment.yaml

Query the list of Pods to verify that the three frontend replicas are running:
kubectl get pods -l app=guestbook -l tier=frontend
```

### Step 6: Deploy Frontend Service

```yaml
nano frontend-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: NodePort
  selector:
    app: guestbook
    tier: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80

```

```yaml
kubectl apply -f frontend-service.yaml
```

### Step 7: **Viewing the Frontend Service**

```yaml
kubectl get service frontend

The response should be similar to this: 
NAME       TYPE           CLUSTER-IP      EXTERNAL-IP        PORT(S)        AGE
frontend   **NodePort**     10.51.242.136       <none>       80:32372/TCP       1m

```

Notice two things:

1. **TYPE** is now **`NodePort`**.
2. **PORT(S)** now shows a mapping like **`80:3XXXX/TCP`**.

That **`3XXXX`** number is the port you need.

Now you can finally open the browser on your main computer and go to:
**`http://<master_node_IP:3XXXX`** (replacing **`3XXXX`** with your actual port).