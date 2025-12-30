# Control Plane Setup

On Raspberry Pi 1: Install containerd, kubelet, kubeadm, kubectl

```bash
sudo swapoff -a
```

```bash
sudo modprobe overlay
```
```bash
sudo modprobe br_netfilter
```

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-ip6tables=1
EOF
```

```bash
sudo sysctl --system
```

```bash
sudo apt update
```

```bash
sudo apt install -y containerd
```

```bash
sudo mkdir -p /etc/containerd
```

verify the values in file:
```bash
containerd config default | sudo tee /etc/containerd/config.toml
```

Update value "SystemdCgroup = true" in the config file

```bash
sudo nano /etc/containerd/config.toml
```

```bash
sudo systemctl restart containerd
```

```bash
sudo systemctl enable containerd
```


Verify containerd is running:
```bash
containerd --version
```

If containerd is installed and running correctly, you should see the version information.


```bash
sudo apt install -y apt-transport-https ca-certificates curl
```

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
| sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /
```

```bash
sudo apt update
```

```bash
sudo apt install -y kubelet kubeadm kubectl
```

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

```bash
kubeadm version
```

```bash
sudo hostnamectl set-hostname pi-control
```

```bash
sudo reboot
```

```bash
ping -c 1 pi-control
```

```bash
ls /proc/sys/net/bridge
```

```bash
mount | grep cgroup
```

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

```bash
sudo systemctl status kubelet --no-pager
```

```bash
sudo systemctl status containerd --no-pager
```

verify swap is off
```bash
swapon --show
```
output should be empty if swap is off

temporary fix
sudo swapoff -a

Verify swap is off:
```bash
swapon --show
```

```bash
sudo nano /boot/firmware/cmdline.txt
```
Add "cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 systemd.unified_cgroup_hierarchy=1" to the end of the line.
after editing, save and exit (Ctrl+X, then Y, then Enter)

Reboot the system to apply the changes:
```bash
sudo reboot
```

Verify cgroup type:
```bash
stat -fc %T /sys/fs/cgroup/
```
on new raspberry pi nodes, you should see "cgroup2fs" as the output.

```bash
cat /sys/fs/cgroup/cgroup.controllers
```

This should show the available cgroup controllers like: cpu io memory pids

```bash
cat /sys/fs/cgroup/cgroup.subtree_control
```

This should show the enabled cgroup controllers for the subtree.eg: "cpuset cpu io memory pids".

-----Clean Up if needed---------

```bash
sudo kubeadm reset -f
```

```bash
sudo rm -rf /etc/cni/net.d
```

```bash
sudo rm -rf $HOME/.kube
```

```bash
sudo systemctl restart containerd
```

```bash
sudo systemctl restart kubelet
```
----------------------------------

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

If all steps complete successfully, you should see a message indicating that the cluster has been initialized successfully.

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
Alternatively, if you are the root user, you can run:
  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/
Then you can join any number of worker nodes by running the following on each as root:
kubeadm join <ip:port> --token 1ihy9q.... \
	--discovery-token-ca-cert-hash sha256:6bc2a5.. 
```

Copy the join command from the output above for use on worker nodes.

```bash
mkdir -p $HOME/.kube
```

```bash
 sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
 ```

```bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
kubectl get nodes
```
sample output:
NAME         STATUS     ROLES           AGE     VERSION
pi-control   NotReady   control-plane   2m25s   v1.29.15

Note: The node shows "NotReady" status because the CNI (Container Network Interface) plugin is not yet installed.

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

```bash
kubectl get nodes
```

NAME         STATUS   ROLES           AGE     VERSION
pi-control   Ready    control-plane   4m53s   v1.29.15

The node should now show as "Ready" status after the CNI plugin is installed.

**Next, setup the worker nodes by running the kubeadm join command copied earlier on each worker node.** Refer to the [Worker Nodes Setup](#worker-nodes-setup) section for detailed instructions.

```bash
kubectl get nodes
```
NAME         STATUS   ROLES           AGE    VERSION
pi-control   Ready    control-plane   118m   v1.29.15
pi-worker    Ready    <none>          10m    v1.29.15

The worker node should now be listed with no specific roles assigned.

Next, create a label to designate the worker node role.

```bash
kubectl label node pi-worker node-role=worker
```

```bash
kubectl get nodes --show-labels
```

sample output:
NAME         STATUS   ROLES           AGE   VERSION   LABELS
pi-control   Ready    control-plane   118m  v1.29.15   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,kubernetes.io/arch=arm64,kubernetes.io/hostname=pi-control,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=master,node-role.kubernetes.io/master=master
pi-worker    Ready    <none>          10m   v1.29.15   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,kubernetes.io/arch=arm64,kubernetes.io/hostname=pi-worker,kubernetes.io/os=linux,node-role=worker

# Worker Nodes Setup

```bash
sudo swapoff -a
```

Make sure to run this command on each worker node before joining it to the cluster.

```bash
sudo modprobe overlay
```

```bash
sudo modprobe br_netfilter
```

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-ip6tables=1
EOF
```

```bash
sudo sysctl --system
```

```bash
sudo apt update
```

```bash
sudo apt install -y containerd
```

```bash
sudo mkdir -p /etc/containerd
```

```bash
containerd config default | sudo tee /etc/containerd/config.toml
```

```bash
sudo systemctl restart containerd
```

```bash
sudo systemctl enable containerd
```

```bash
containerd --version
```

```bash
sudo apt install -y apt-transport-https ca-certificates curl
```

```bash
sudo rm /etc/apt/sources.list.d/kubernetes.list
```

```bash
sudo mkdir -p /etc/apt/keyrings
```

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /
```

```bash
sudo apt update
```

```bash
sudo apt install -y kubelet kubeadm kubectl
```

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

```bash
kubeadm version
```
Output: kubeadm version: &version.Info{Major:"1", Minor:"29", GitVersion:"v1.29.15", GitCommit:"0d0f172cdf9fd42d6feee3467374b58d3e168df0", GitTreeState:"clean", BuildDate:"2025-03-11T17:46:36Z", GoVersion:"go1.23.6", Compiler:"gc", Platform:"linux/arm64"}

This confirms that kubeadm has been successfully installed and is ready to be used for initializing the Kubernetes cluster.

```bash
sudo hostnamectl set-hostname pi-worker
```

```bash
sudo reboot
```

check hostname:
```bash
hostname
```

```bash
ping -c 1 pi-worker
```

```bash
mount | grep cgroup
```

sample output: cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate,memory_recursiveprot)

```bash
stat -fc %T /sys/fs/cgroup/
```

```bash
grep SystemdCgroup /etc/containerd/config.toml
```

If the output shows `SystemdCgroup = true`, then containerd is already configured to use systemd cgroup driver. If it shows `SystemdCgroup = false` or the line is missing, you need to enable it: 

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

```bash
sudo systemctl restart containerd
```

```bash
sudo swapoff -a
```

Ensure swap is disabled permanently by commenting out swap entries in /etc/fstab if present.

```bash
sudo nano /boot/firmware/cmdline.txt
```
Add `cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 systemd.unified_cgroup_hierarchy=1` to the end of the line in cmdline.txt if not already present.


--------If reset is needed, execute these commands-----------

```bash
sudo kubeadm reset -f
```

```bash
sudo rm -rf /etc/kubernetes
```

```bash
sudo rm -rf /etc/cni/net.d
```

```bash
sudo rm -rf /var/lib/kubelet
```

```bash
sudo systemctl restart containerd
```

```bash
sudo systemctl restart kubelet
```

Check the output
```bash
cat /sys/fs/cgroup/cgroup.controllers | grep memory
```
output: cpuset cpu io memory pids


--------------------------------------------------------------

Now you can join the worker node to the cluster.

```bash
sudo kubeadm join <CONTROL_PLANE_IP>:<port> --token 1ihy9q.2vi7...    --discovery-token-ca-cert-hash sha256:6bc2a...
```

Make sure to replace `<CONTROL_PLANE_IP>` with the actual IP address of your control plane node and `<port>` with the API server port (usually 6443).

If the join is successful you will see a message like:
[preflight] Running pre-flight checks
	[WARNING SystemVerification]: missing optional cgroups: hugetlb
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

Now, continue with the steps in control plane node to verify the worker node has joined successfully.