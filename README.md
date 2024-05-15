Rancher Kubernetes Engine for Kubernetes cluster installation and Rancher Management UI:

What is Rancher?

Rancher is an open-source software platform designed to manage and deploy Kubernetes clusters in production environments. It provides a user-friendly interface for managing containerized applications across different Kubernetes clusters, whether they are hosted on-premises, in the cloud, or in hybrid environments.

Why Rancher?

Rancher simplifies the management and operation of Kubernetes clusters, making it easier for organizations to adopt and scale containerized applications in production environments while maintaining security, compliance, and operational efficiency.

What are we doing here and Where are we doing this?
This document helps us to setup kubernetes cluster in on-prem server running on Ubuntu 22.04. 
In our setup process, we're utilizing Rancher Kubernetes Engine (RKE) version 1.4.xx to establish our Kubernetes cluster. To achieve this, we employ the 'rke config' command, where we input essential key values such as the number of nodes, their respective IP addresses, and their roles—whether they function as control plane, etcd, or worker nodes. Additionally, we specify the desired Kubernetes version for installation across these nodes.

Once the cluster is created, our next step involves installing the Rancher management UI within one of the clusters. Subsequently, we proceed to create an additional cluster, designated as an application downstream cluster. Importantly, we integrate this newly created cluster into the Rancher management UI. This integration facilitates streamlined management and oversight.

Finally, leveraging the Rancher management UI, we deploy applications onto the downstream cluster and initiate monitoring processes to ensure optimal performance and reliability

Creating rancher ui management cluster and adding a downstream application cluster to it.

THIS IS THE PRE-REQUISTE AND DOCKER 23.06 INSTALLATION PROCESS GUIDE:

1. Check these packages exit already or not:

       dpkg -l |grep -i containerd
   
       dpkg -l |grep -i docker
       dpkg -l |grep -i apparmor



3. Disable apparmor:

check the status:

    sudo systemctl status apparmor

    sudo systemctl stop apparmor

    sudo systemctl disable apparmor

check again.

    sudo systemctl status apparmor

    sudo apt update

3.Disable  swap:


    sudo swapoff -a

    sudo sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab

check now, 

    free -h
See the swap is commented out or not:

    cat /etc/fstab | grep swap


4. Configure network parameters / CP-W

Create k8s.conf file in /etc/sysctl.d

    sudo vim /etc/sysctl.d/k8s.conf

Add below content, save and close the file

    net.bridge.bridge-nf-call-ip6tables = 1 

    net.bridge.bridge-nf-call-iptables = 1 

    net.ipv4.ip_forward = 1


Apply newly added network params

    sudo sysctl --system


5. Installing Docker 23.0.6 version:
   This is how it starts;
     
a
    
    sudo apt-get update
    sudo apt-get install ca-certificates curl  

    sudo install -m 0755 -d /etc/apt/keyrings

    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

    sudo chmod a+r /etc/apt/keyrings/docker.asc

    echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  
    sudo apt-get update

    VERSION_STRING=5:23.0.6-1~ubuntu.22.04~jammy

    sudo apt-get install docker-ce=$VERSION_STRING docker-ce-cli=$VERSION_STRING containerd.io docker-buildx-plugin docker-compose-plugin

    systemctl enable docker
    systemctl start docker
    systemctl status docker

add the user with appropriate previlages

    sudo usermod -aG docker $USER

    id $USER


6. Configure containerd:

Create a directory to store containerd config file in /etc/

     sudo mkdir -p /etc/containerd

Generate the default config.toml file


    sudo containerd config default|sudo tee /etc/containerd/config.toml

Open the generated file in any text editor and verify whether the following settings are present. If not, you should set them as shown below

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
   runtime_type = "io.containerd.runc.v2"  # <- note this, this line might have been missed
  
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  
      SystemdCgroup = true # <- note this, this could be set as false in the default configuration, please make it true


﻿
restart the containerd:

    sudo systemctl restart containerd
  
    sudo systemctl enable containerd

    systemctl status containerd



THIS IS FOR RANCHER MANAGEMENT CLUSTER BY RKE:

1. Check the hostnames are unique on all VMs:

        cat /etc/hostnames

2. Check the sudo previlages permissions for sadmin;

        sudo visudo

3. Check the firewall is disabled: 

        sudo ufw status


4. Check the swap is turned off or not;


        free -h

if not do this

    sudo swapoff -a
    sudo sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab

5. Check for apparmor is disabled or not:

        sudo systemctl status apparmor


6. Copy the SSH keys  into the VMs:

        ssh-keygen
(Considering for 3 nodes HA racnher mgmt cluster)

        for server in X.X.X.X X.X.X.X X.X.X.X; do ssh-copy-id sadmin@"$server"; done

7. Download the RKE(v1.4.17) binary file 

        wget https://github.com/rancher/rke/releases/download/v1.4.17/rke_linux-amd64
        chmod +x rke_linux-amd64
        sudo mv rke_linux-amd64 /usr/local/bin/rke
        rke --version


8.Create an RKE cluster.yaml file according to your need and assign the roles respectivley:
  choose  kuberentes version as 1.26.15

9. Install kubectl 

       curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
       sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
 

10. install Helm:

        wget https://get.helm.sh/helm-v3.7.1-linux-amd64.tar.gz || curl -LO https://get.helm.sh/helm-v3.7.1-linux-amd64.tar.gz
        tar -zxvf helm-v3.7.1-linux-amd64.tar.gz
        sudo mv linux-amd64/helm /usr/local/bin/helm
        helm version

10a. Add helm rancher repo
 
      helm repo add rancher-stable https://releases.rancher.com/server-charts/stable

11. create namespace

         kubectl create namespace cattle-system


        helm install rancher rancher-stable/rancher --version 2.7.9   --namespace cattle-system --set hostname=<your custom website name> --set ingress.tls.source=secret

         kubectl -n cattle-system rollout status deploy/rancher --kubeconfig kube_config_cluster.yml


12. apply the secrets and certificates 

         kubectl -n cattle-system create secret tls tls-rancher-ingress   --cert=<certifate_file>.crt   --key=<keyfile>.key





THIS IS FOR DOWNSTREAM APPLICATION CLUSTER BY RKE:



1. Check the hostnames are unique on all VMs:

        cat /etc/hostname

1a. Enter the ip and domain name of the rancher mgmt cluster in hosts file so that local network can resolve.

          vi /etc/hosts

          <ip address of rancher mgmt node> <your custom rancher mgmt website name>

2. Check the sudo previlages permissions for sadmin;

        sudo visudo

3. Check the firewall is disabled: 

        sudo ufw status


4. Check the swap is turned off or not;

        free -h

if not do this

      sudo swapoff -a
      sudo sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab

5. Check for apparmor is disabled or not 

        sudo systemctl status apparmor


6. Copy the SSH keys  into the VMs:

        ssh-keygen (from the vm where you are going to do rke )

(Considering for 4 nodes with 1 control plane and etcd and other 3 with worker roles assigned for downstream application cluster)

        for server in X.X.X.X X.X.X.X X.X.X.X; do ssh-copy-id sadmin@"$server"; done

7. Download the RKE(v1.4.17) binary file 

        wget https://github.com/rancher/rke/releases/download/v1.4.17/rke_linux-amd64
        chmod +x rke_linux-amd64
        sudo mv rke_linux-amd64 /usr/local/bin/rke
        rke --version

8. RKE config command and choosing the right kubernetes version while configuring cluster.yml file:
 
        kubernetes: rancher/hyperkube:v1.26.15-rancher1
 
 
8a. RKE up command:

run rke up command and your cluster will be up.
              
              rke up

9. To communicate with your kubernetes cluster install kubectl tool:
 

       curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
       sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
 
10. creating the kubeconfig file in in its default directory

        mkdir .kube
        mv kube_config_cluster.yaml .kube/config
        chmod 770 .kube/config

11. Apply the patch at the end.

              kubectl -n cattle-system patch  deployments cattle-cluster-agent --patch '{
                  "spec": {
                      "template": {
                          "spec": {
                              "hostAliases": [
                                  {
                                    "hostnames":
                                    [
                                      "<your custom rancher mgmt website name>"
                                    ],
                                    "ip": "<ip address of rancher mgmt node>"
                                  },
                                  {
                                    "hostnames":
                                    [
                                      "<your custom rancher mgmt website name>"
                                    ],
                                    "ip": "<ip address of rancher mgmt node>"
                                  }
 
                              ]
                          }
                      }
                  }
              }'
