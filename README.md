# k8s-cluster-setup

1. Update apt and install required packages

  ```
  sudo apt-get update
  sudo apt-get install -y ca-certificates curl
  ```

2. Download signing key
  ```
  curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
  ```
3. Add kubernetes apt repository
  ```
  echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
  ```
4. Update apt package and install kubernetes tools
  ```
  sudo apt-get update
  sudo apt-get install -y kubelet kubeadm kubectl
  ```
5. Enable kubelet
  ```
  sudo systemctl enable kubelet
  ```
6. Disable swap for current session
  ```
  sudo -i  #for root user
  swapoff -a
  exit
  ```
  Also do this permanently by modifying `fstab` file:
  ```
  sudo nano /etc/fstab
  
  #Comment line file system line having type swap, ignore if this line does not exist in your fstab file
  # UUID=XXXXX    none   swap   sw    0   0
  ```
7. Enable Kernel Modules for current session 
  ```
  sudo modprobe overlay
  sudo modprobe br_netfilter
  ```
  Also do this permanently
  ```
  sudo tee /etc/modules-load.d/containerd.conf <<EOF
  overlay
  br_netfilter
  EOF
  ```
8. Add `sysctl` settings
  ```
  sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  net.ipv4.ip_forward = 1
  EOF
  ```
  Reload `sysctl`
  ```
  sudo sysctl --system
  ```
9. Install Container Runtime
  ```
  sudo apt-get install -y wget gnupg-agent software-properties-common
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
  sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  sudo apt update
  sudo apt install -y containerd.io
  sudo systemctl enable containerd
  ```
  Add default configuration
  ```
  sudo -i
  containerd config default > /etc/containerd/config.toml
  exit
  ```
10. Set `systemCgroup` to true in `containerd` config
  ```
  sudo nano /etc/containerd/config.toml
  ```
  Find and set `SystemdCgroup = true` under `plugins` on config file, then restart `containerd`.
  ```
  sudo systemctl restart containerd
  ```

11. On master node, initialize kubernetes cluster by running the following command
  ```
  sudo kubeadm init \
  --pod-network-cidr=172.16.0.0/16 \
  --cri-socket unix:///run/containerd/containerd.sock
  ```
  This command will take some time to execute, and will output a `kubeadm join` command that can be used by worker nodes to connect to the master. `kudeadm join` command looks like this:
  ```
  kubeadm join {MASTER_NODE_IP}:6443 --token [KUBE_TOKEN] \
    --discovery-token-ca-cert-hash sha256:c692fb049u15883b575bd6710779db5c5af8073f7cab460lod181yr3ddb29a18
  ```
   
12. To regenerate the `kubeadm join` command, run this on master node:
  
  ```
  kubeadm token create --print-join-command
  ```
13. Run the following on master node after `kubeadm init` has successfully ran to be able to use the cluster.
  ```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```
