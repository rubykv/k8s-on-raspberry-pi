# Deploy Nginx Service

This section explains how to deploy a simple Nginx service to the Kubernetes cluster.

## 1. Create a Deployment

Create a file named `nginx-deployment.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

Apply the deployment:

```bash
kubectl apply -f nginx-deployment.yaml
```

```bash
kubectl get pods -o wide
```
output:
NAME                     READY   STATUS    RESTARTS   AGE   IP             NODE        NOMINATED NODE   READINESS GATES
nginx-6c68485969-4rlph   1/1     Running   0          35s   192.168.68.1   pi-worker   <none>           <none>


```bash
 kubectl expose deployment nginx \
  --type=NodePort \
  --port=80
```

```bash
kubectl get svc nginx
```
output:
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.109.133.75   <none>        80:31902/TCP   10s

Copy the port number (31902 in this case) and access the service from any device on the network using the IP address of your control plane node followed by that port.

```bash
kubectl get nodes -o wide
```
output:
NAME         STATUS   ROLES           AGE   VERSION   LABELS
pi-control   Ready    control-plane   118m  v1.29.15   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,kubernetes.io/arch=arm64,kubernetes.io/hostname=pi-control,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=master,node-role.kubernetes.io/master=master
pi-worker    Ready    <none>          10m   v1.29.15   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,kubernetes.io/arch=arm64,kubernetes.io/hostname=pi-worker,kubernetes.io/os=linux,node-role=worker


```bash
kubectl get endpoints nginx
```
output:
NAME    ENDPOINTS             AGE
nginx   192.168.68.1:80       2m 

Access the service at: http://pi-control:31902 (replace with your control plane IP and port)

If it doesn't work, check that the NodePort service is correctly mapped to the pod IP and port. You can also try accessing from a different browser or clear cache.