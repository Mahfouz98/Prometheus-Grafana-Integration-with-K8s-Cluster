# 🛠️ Integration Prometheus and Grafana With Kubernetes Cluster Using KUBEADM
![1594668243636](https://github.com/Mahfouz98/Prometheus-Grafana-Integration-with-K8s-Cluster/assets/145352617/b6d198be-d21a-4f07-a3c6-26deb222ad58)


## This Project using AWS EC2 Instances to make Cluster and Cri-o as a Container runtime 

## Steps to Make a Cluster ⚙️
- First you have to make 2 EC2 Instances ( Master & Worker Nodes ) Choose the Type ( t3.medium ) and i used ubuntu 22.04 as AMI 
  - Connect to Both instances with ssh and Do these steps
  - Enable iptables Bridged Traffic on all the Nodes
    ```bash
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF
    
    sudo modprobe overlay
    sudo modprobe br_netfilter
    
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF

    sudo sysctl --system
    ```
  - To Disable Swap `swapoff -a` or `sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab` 
  - Disable SElinux by this command `sudo setenforce 0`

- Then you have to install Container Runtime on both Instances I preferred Cri-o instead of docker
   - Steps to add the repo and gpgkey
     ```bash
     sudo apt-get update -y
     sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates

     curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key |
      gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
     echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" |
      tee /etc/apt/sources.list.d/cri-o.list

     sudo apt-get update -y
     sudo apt-get install -y cri-o

     sudo systemctl daemon-reload
     sudo systemctl enable crio --now
     sudo systemctl start crio.service
     ```
  - Install Crictl  ( copy it as it is )
    ```bash
    VERSION="v1.28.0"
    wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
    sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
    rm -f crictl-$VERSION-linux-amd64.tar.gz
    ```
    
  - Install Kubeadm & Kubelet & Kubectl on all Nodes
    ```bash
    KUBERNETES_VERSION=1.29
    sudo mkdir -p /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

    sudo apt-get install -y kubelet=1.29.0-1.1 kubectl=1.29.0-1.1 kubeadm=1.29.0-1.1
    sudo apt-mark hold kubelet kubeadm kubectl   #used to prevent Upgrade only
    ```
  - Add the node IP to KUBELET_EXTRA_ARGS
    ```bash
    sudo apt-get install -y jq
    local_ip="$(ip --json addr show eth0 | jq -r '.[0].addr_info[] | select(.family == "inet") | .local')"
    cat > /etc/default/kubelet << EOF
    KUBELET_EXTRA_ARGS=--node-ip=$local_ip
    EOF
    ```

## 🔥 Initialize Kubeadm On Master Node To Setup Control Plane

- To use Public IP on masternode
  ```bash
  IPADDR=$(curl ifconfig.me && echo "")
  NODENAME=$(hostname -s)
  POD_CIDR="192.168.0.0/16"
  ```
- Initialize The Cluster
  ```bash
  sudo kubeadm init --control-plane-endpoint=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap
  ```
- Now you have to make these 3 steps in the image and if you are using root user then don't forget to execute this command `export KUBECONFIG=/etc/kubernetes/admin.conf`
  
<img width="789" alt="kubeadm" src="https://github.com/Mahfouz98/Prometheus-Grafana-Integration-with-K8s-Cluster/assets/145352617/1ee57ee6-af42-4481-9770-58394598dd0c">

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

-  The Last Command that starts with Kubeadm join, you will use it from the worker Node to join the Cluster

  - After join the Worker Node try to Execute `kubectl get nodes` you will see both nodes but status not-ready

  - Then you have to Install Calico Network Plugin for Pod Networking
    `kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml`
  - Then check again, you will see them ready

  - You Can Setup Kubernetes Metrics Server ( But i didn't )

## Let's Install Helm ✅

![helm](https://github.com/Mahfouz98/Prometheus-Grafana-Integration-with-K8s-Cluster/assets/145352617/997052cf-9c01-47a1-bbee-c6b411917509)


- There is easy way to install via script
  ```bash
  curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
  chmod 700 get_helm.sh
  ./get_helm.sh
  ```
- Adding Helm Charts & Install Prometheus and Grafana 
  ```bash
  helm repo add stable https://charts.helm.sh/stable
  helm repo add prometheus-community https://prometheus-community.github.io/helm-chart
  helm search repo prometheus-community
  kubectl create namespace prometheus 
  helm install stable prometheus-community/kube-prometheus-stack -n prometheus
  ```
![Screenshot from 2024-05-04 18-02-36](https://github.com/Mahfouz98/Prometheus-Grafana-Integration-with-K8s-Cluster/assets/145352617/37ecee38-db8e-4336-a86f-19722d49b466)

  
- To Check the services created
  `kubectl get svc -n prometheus`

- The Final Step to change the service from ClusterIP into NodePort or LoadBalancer
  - To use LoadBalancer you must Identify an IAM role for the EC2 instance in creation process
  - So we will be using NodePort for the 2 services
    ```bash
    kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
    kubectl edit svc stable-grafana -n prometheus
    ```
  - Scroll down to the end you will find
    - ` type: ClusterIP `  Change it to be  `type: NodePort`
    - Do these for the 2 services
    - As you can see in this screenshot
      
      ![Screenshot from 2024-05-04 17-56-26](https://github.com/Mahfouz98/Prometheus-Grafana-Integration-with-K8s-Cluster/assets/145352617/c9c89f1e-d6bd-49b4-a025-e57ef20c0644)



  - Now you can Check the Services By Using `http://<your_public_ip>:NodePort`  and this NodePort is written in svc File ( Check the Above Image you will see NodePort: 30225 )
  - you can get it through
   ```bash
    kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
    kubectl edit svc stable-grafana -n prometheus
    ```
  - To get the Password of Grafana Login Page, and username is admin by default
    
    `kubectl get secret -n prometheus stable-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo`
    
    ![Screenshot from 2024-05-04 17-56-13](https://github.com/Mahfouz98/Prometheus-Grafana-Integration-with-K8s-Cluster/assets/145352617/998aae42-ec27-4af8-a20e-eda5692bfc74)
  - In mycase Grafana password was `prom-operator`

    - These are Screenshots from Grafana and Prometheus Dashboards
    ![Screenshot from 2024-05-04 17-56-04](https://github.com/Mahfouz98/Prometheus-Grafana-Integration-with-K8s-Cluster/assets/145352617/8e422500-8efc-4a5b-b992-d7a7fa26e33f)
    ![Screenshot from 2024-05-04 17-56-07](https://github.com/Mahfouz98/Prometheus-Grafana-Integration-with-K8s-Cluster/assets/145352617/6e514514-6a17-49da-a1de-3f0d921542dc)

## 🎉 Acknowledgements

Thank you for taking the time to explore this project. Your interest and feedback are greatly appreciated. If you have any questions or suggestions, please feel free to open an issue or submit a pull request. Happy coding!

  














  
    
