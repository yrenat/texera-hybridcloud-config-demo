# Guide: Setting Up a Hybrid Kubernetes Cluster (Local Linux PC + AWS EC2)

**Objective:** To create a single Kubernetes cluster where the control plane (master) runs on a local Linux machine and a worker node runs on a remote AWS EC2 instance. Secure communication between the two is achieved using a Tailscale overlay network.  

**Author: Yichen Ren (zaoduyuan@gmail.com), with the help of Mr. Ali Risheh (arisheh@uci.edu) and Professor Chen Li (chenli@gmail.com)** 

## Prerequisites

1.  **Two Machines:**
    *   **Control Plane:** A local PC running a Debian-based Linux distribution (e.g., Ubuntu 22.04).
    *   **Worker Node:** An AWS EC2 instance, also running Ubuntu 22.04.
2.  **Access:** `sudo` or root privileges on both machines.
3.  **Tailscale Account:** A free account from [tailscale.com](https://tailscale.com).
4.  **AWS Security Group:** For the EC2 instance, ensure the security group allows all inbound and outbound traffic *from your Tailscale IP range* (e.g., `100.64.0.0/10`). For simplicity during setup, I allow all traffic.
5. To start a tailscale service with kubetnetes at the main time, I strongly advice you choose a machine with at least 4 cores. The minimum requirement for kubernetes to run is 2 cores, but in that case tailscale will suffer and the whole traffic will be quite jammed and slow. 
5.  **Please do in the order of what the following guide shows, otherwise some random bugs may (must) occur!**

---

## Phase 1: Configure the Secure Overlay Network (Tailscale)

This must be done on **BOTH** the local PC and the AWS EC2 instance.

1.  **Install Tailscale:**
    This command downloads and runs the installer script for tailscale.
    ```bash
    curl -fsSL https://tailscale.com/install.sh | sh
    ```

2.  **Start Tailscale and Connect:**
    This `up` command brings the network interface "up" (to start the tailscale service). You will then be prompted with a website. Visit the websites on both local PC and AWS EC2 machines and follow the guideline to input your email address for authentication. **Remember to use the same email address for both local PC and AWS EC2 machines**.
    ```bash
    sudo tailscale up
    ```

3.  **Verify and Get Tailscale IPs:**
    After authenticating, check the status and find the IP address assigned to each machine by the following command. A correctly assigned IP address by tailscale will be something like `100.x.x.x`.
    ```bash
    tailscale ip -4
    ```
    ***keep that Tailscale IP of your local PC (Control Plane) somewhere. You will need it later.***

---

## Phase 2: Install Kubernetes Components

This must be done on **BOTH** the local PC and the AWS EC2 instance.

1.  **Disable Swap:**
    Kubernetes requires swap to be disabled to function correctly.
    ```bash
    sudo swapoff -a
    # Permanently disable swap by commenting it out in the fstab file.
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
    ```

2.  **Enable Kernel Modules and `sysctl` Settings:**
    This is required for the container runtime and virtual networking.
    ```bash
    sudo modprobe overlay
    sudo modprobe br_netfilter

    # Purpose: Configure sysctl settings for Kubernetes networking to persist across reboots (optional).
    sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    EOF

    # Purpose: Apply the new sysctl settings immediately (mandatory).
    sudo sysctl --system
    ```

3.  **Install `containerd` (Container Runtime):**
    ```bash
    # Purpose: Install dependencies.
    sudo apt-get update
    sudo apt-get install -y ca-certificates curl

    # Purpose: Add Docker's official GPG key and repository (which provides containerd).
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update

    # Purpose: Install the containerd runtime.
    sudo apt-get install -y containerd.io

    # Purpose: Generate a default containerd configuration and set SystemdCgroup to true.
    sudo mkdir -p /etc/containerd
    sudo containerd config default | sudo tee /etc/containerd/config.toml
    sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

    # Purpose: Restart and enable the containerd service.
    sudo systemctl restart containerd
    sudo systemctl enable containerd
    ```

4.  **Install Kubernetes Tools (`kubeadm`, `kubelet`, `kubectl`):**
    ```bash
    # Purpose: Add the official Kubernetes repository and GPG key.
    sudo apt-get install -y apt-transport-https ca-certificates curl gpg
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

    # Purpose: Install the Kubernetes tools.
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl

    # Purpose: Prevent the packages from being accidentally auto-updated (optional).
    sudo apt-mark hold kubelet kubeadm kubectl
    ```

---

## Phase 3: Initialize the Control Plane

This is done **ONLY** on the **local PC**.

1.  **Initialize the Cluster with `kubeadm`:**
    This is the most critical command. It sets up the entire control plane.
    ```bash
    # !!! IMPORTANT !!!
    # Replace <TAILSCALE_IP_OF_LOCAL_PC> with the actual Tailscale IP (e.g., 100.86.203.40).
    sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=<TAILSCALE_IP_OF_LOCAL_PC>
    ```

2.  **Configure `kubectl` for Your User:**
    After `kubeadm init` succeeds, it will print three commands (shown as below). Run them to configure `kubectl` access.
    ```bash
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

3.  **Perform and Save the Join Command:**
    The output of `kubeadm init` will include a `kubeadm join` command with a token and hash.
    ```bash
    # !!! IMPORTANT !!!
    kubeadm token create --print-join-command
    ```
    **Copy this entire command and save it. You will need it for the worker node to join master node**. If you lose it, you can regenerate it

---

## Phase 4: Join the Worker Node

This is done **ONLY** on the **AWS EC2 instance**.

1.  **Join the Cluster:**
    Run the `kubeadm join` command you saved from the previous phase. You must run it with `sudo`.
    ```bash
    # Purpose: Connects the worker node to the control plane.
    # Example command (your token and hash will be different):
    sudo kubeadm join 100.86.203.40:6443 --token <your_token> --discovery-token-ca-cert-hash sha256:<your_hash>
    ```

---

## Phase 5: Config IP routing in AWS EC2 machine
Do this whenever you cannot connect to the pods from local master node (e.g., you get a timeout when trying to log that remote pod in AWS EC2 node)
```bash
sudo nano /var/lib/kubelet/kubeadm-flags.env
# Then add ip-flag to the beginning of the file
KUBELET_KUBEADM_ARGS="--node-ip=100.90.108.98 --network-plugin=cni...
# when you run kubectl get nodes -o wide in your local PC control panel, you should see the tailscale IP for your remote AWS EC2 machine in the "internal IP" session, rather than the physical real one allocated by AWS. 
```
After that, remember to run
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
``` 
to restart the service, otherwise the changes cannot take effect. 

## Phase 6: Install the CNI Network Plugin 

This is done **ONLY** on the **local PC (Control Plane)**.

1.  **Apply the Calico Manifest:**
    A CNI plugin is required for pod networking. Without it, nodes will remain `NotReady` and `coredns` will be stuck in `ContainerCreating`.
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
    ```

---

## Phase 7: Run This Whenever You Add a New AWS EC2 Node only run locally

1. Re-generate the Calico CA certificate:

    ```bash
    kubectl delete -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml
    kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml
    ```

2. Re-visit Phase 3 step 3

## Phase 8: Create Pod(s) on AWS EC2
1. Generally speaking, you can directly use the ```helm``` command to directly install k8s images, but there can be cases (especially when you have restarted your local PC/remote AWS EC2 server) where you will continuously see ```containercreating``` status for one or few more pods. In this case, if you run ```describe pods``` command, you will probably see that there is a failure in Calico. The command to fix that is still
```bash
    kubectl delete -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml && \
    kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml
```
2. You are strongly advised to first start all your pods locally, in case of any networking issues that may hidden the underlying code bugs. Once you succeed, you can further move to remote servers by 
```bash
kubectl patch statefulset YOUR-POD-NAME -n YOUR-NAMESPACE -p '{"spec":{"template":{"spec":{"nodeSelector":{"node-type":"YOUR-NODE-TYPE"}}}}}'
```
Note that after patching (moving) a new pod, you have to perform a deletion for that specific pod that you are moving. Because Kubernetes cannot move a pod that is running. 

3. If you want to create a minio pod or other pods that need PersistVolumeClaim (PVC), you need to run the following command
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml

# should be after phase 6, only run locally
```
(You are strongly advised to run minio service locally, as AWS EC2 configurations have certain potential bugs regarding persistent volume)



## Phase 9: pod to pod connection
the protocal used by Kubernetes to handle pod to pod communication is Calico, who uses IP-IP mode by default (one IP address encapsulated in another IP address). But IP-IP packets is not supported by a great number of routers, so the pod-to-pod traffic is easily getting dropped off. To solve that problem, I recommend to use VXLAN mode for transmitting packets. You can modify by following the steps below:
```bash
nano ippool.yaml
```
Then set ipipMode to Never and vxlanmode to Always. 

## Phase 10: Change config of Calico from IPIP to VXLAN
```bash
kubectl patch ippool default-ipv4-ippool --type='merge' -p '{"spec":{"ipipMode":"Never","vxlanMode":"Always"}}'
kubectl rollout restart daemonset/calico-node -n kube-system
```

## Phase 11: Finally, Verification and Troubleshooting

Run these commands on the **local PC (Control Plane)**.

1.  **Check Node Status:**
    Initially, the worker node may show `NotReady`. After Calico is installed and running (1-2 minutes), nodes should become `Ready`.
    ```bash
    kubectl get nodes
    ```
    **Success Output:**
    ```
    NAME               STATUS   ROLES           AGE   VERSION
    ip-172-31-46-192   Ready    <none>          10m   v1.30.x
    local-pc-name      Ready    control-plane   12m   v1.30.x
    ```

2.  **Check System Pod Status:**
    This command shows the health of the core cluster components. After Calico is installed, all pods should eventually be `Running`.
    ```bash
    kubectl get pods -n kube-system -o wide
    ```

---

### only common issues are shown

*   **Symptom:** Worker node stays `NotReady` for more than 5 minutes.
    *   **Cause:** CNI (Calico) is not installed or failing.
    *   **Fix:** Check the logs of the `calico-node` pod on the worker node.
        ```bash
        # Find the name of the calico-node pod running on the NotReady node
        kubectl get pods -n kube-system -o wide
        
        # Check its logs for errors
        kubectl logs -n kube-system <calico-pod-name-on-worker>

        # Then jump to phase 5 and run all commands afterwards (including commands in phase 5) again.
        ```

*   **Symptom:** `kubectl` gives a `certificate signed by unknown authority` error.
    *   **Cause:** Your `$HOME/.kube/config` is stale, likely after running `kubeadm reset`.
    *   **Fix:** Re-copy the fresh admin config as shown in Phase 3, Step 2.

*   **Symptom:** `kubectl` gives an `Unable to connect to the server: dial tcp <IP>:6443: i/o timeout` error.
    *   **Cause:** A firewall (like an AWS Security Group) is blocking traffic between the nodes on port 6443, or Tailscale is not running.
    *   **Fix:** Ensure `sudo tailscale up` is active on both machines and that your AWS Security Group allows traffic from your Tailscale IPs.
