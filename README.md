<---------------------------------- MULTI NODE KUBERNETES SETUP BY - aamir - 2024 <---------------------------------->
                        
# Please Refer this link  :------->
https://github.com/vimallinuxworld13
https://github.com/vimallinuxworld13/Kubernetes_MultiNode_Cluster_using_Kubeadm/blob/master/common-setup-master-slave.sh


Agenda :- Kubernets setup using 'kubeadm' in AWS Ec2 RedHat Linux Server 
###############################################################################
   1 - Master  [ Control Plane ]   ==  ( 4 GB , 2 Core ) ....   T2.Medium
   2 - Worker  [  Worker Node  ]   ==  ( 1 GB , 1 Core ) ....   T2.Micro
################################################################################

=======================--
S T A R T   S E T  UP:-
=======================--
NOTE : - Go to AWS , Ceate RedHat ec2 Instance with T2.Medium.
       - Go to Advance Option and Paste Below script and then Launch the instance. 
       
# >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
#!/bin/bash
sudo su - 
sudo yum update -y

sudo hostnamectl set-hostname MASTER
# sudo hostnamectl set-hostname worker1
# sudo hostnamectl set-hostname worker2

bash
yum install net-tools  vim -y

swapoff -a
dnf install -y iproute-tc

modprobe overlay
modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system

setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

KUBERNETES_VERSION=v1.29
PROJECT_PATH=prerelease:/main

cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/repodata/repomd.xml.key
EOF

cat <<EOF | tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://pkgs.k8s.io/addons:/cri-o:/$PROJECT_PATH/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/addons:/cri-o:/$PROJECT_PATH/rpm/repodata/repomd.xml.key
EOF

dnf install -y cri-o kubelet kubeadm kubectl

systemctl enable --now crio

systemctl enable --now kubelet
# >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

# Note :---- When Instance Launch Successfully , paste Below Commands


$ sudo su -

# This Command will give you power your system as a Client System :-
$ kubeadm init --pod-network-cidr=192.168.0.0/16


## PASTE IT ONLY MASTER :-
  
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config



## PASTE IT ONLY WORKER: -

kubeadm join 172.31.41.78:6443 --token vx4e4q.k0tcy5pk8cny41x2 \
        --discovery-token-ca-cert-hash sha256:c213fe848fdef2f9a596a52b963a307177f21e465da9609d18aebaccf7db302d

        
# For checking our system work as a client or not check below command:
$ kubectl get pods
$ kubectl get svc
$ kubectl get pods -n kube-system
$ hostname
$ systemctl status kubelet
$ kubectl get nodes
--> NAME     STATUS   ROLES           AGE   VERSION
--> master   Ready    control-plane   21m   v1.29.2

# By default, apps won’t get scheduled on the master node. If u want to use the master node for scheduling apps, taint the master node.
$ kubectl taint nodes --all node-role.kubernetes.io/control-plane-

# Now Create our own deploymnet :
$ kubectl create deployment mydep1 --image=vimal13/apache-webserver-php
$ kubectl create deployment mydep1 --image=vimal13/apache-webserver-php --replicas=5
$ kubectl expose deployment mydep1 --type=NodePort --port=80
$ kubectl scale deployment mydep1 --replicas=1
$ kubectl get svc
$ kubectl get deploy
$ kubectl get pods
$ kubectl describe  pods < pod-name >
$ kubectl get pods -o wide
$ kubectl exec -it < pod-name > -- bash

# Scaling the Deploymnet :-
$ kubectl scale deployment mydep1 --replicas=4

# NOTE:--  Now Our Master Node ready , now joint the wroke node :- 
## If you missed copying the join command, execute the following command in the master node to recreate the token with the join command :-
$ kubeadm token create --print-join-command


# NOTE:--
# Now check how many Node/Worker join our cluster 
$ kubectl get nodes
--> NAME      STATUS   ROLES           AGE   VERSION
--> master    Ready    control-plane   53m   v1.29.2
--> worker1   Ready    <none>          85s   v1.29.2


# NOTE:- For Labeld the worker node
$ kubectl label node  ip-172-31-42-227.ap-south-1.compute.internal   node-role.kubernetes.io/worker=worker
# IN OUR CASE :-                                                          
$ kubectl label node  worker1   node-role.kubernetes.io/worker=worker
$ kubectl describe  nodes worker1

$ kubectl get nodes
--> NAME      STATUS   ROLES           AGE   VERSION
--> master    Ready    control-plane   56m   v1.29.2
--> worker1   Ready    worker          5m    v1.29.2


## Now You verify all the cluster component health statuses :-
$ kubectl get --raw='/readyz?verbose'
$ kubectl cluster-info 
                                                          
                                        

$ kubectl get nodes
--> NAME      STATUS   ROLES           AGE     VERSION
--> master    Ready    control-plane   80m     v1.29.2
--> worker1   Ready    worker          29m     v1.29.2
--> worker2   Ready    worker          3m48s   v1.29.2



## Kubeadm doesn’t install metrics server component during its initialization.
$ kubectl describe  nodes worker1
$ kubectl apply -f https://raw.githubusercontent.com/techiescamp/kubeadm-scripts/main/manifests/metrics-server.yaml

$ kubectl top  nodes
--> NAME      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
--> master    97m          4%     1109Mi          31%
--> worker1   6m           0%     340Mi           52%
--> worker2   15m          1%     360Mi           55%

$ kubectl top  pods -n kube-system
$ kubectl top  nodes --sort-by=cpu


# CALICO PACKAGE :- 
$ curl -O https://raw.githubusercontent.com/projectcalico/calico/master/manifests/calico.yaml
---> calico.yaml
$ vi calico.yaml                    [ if Require ]

# Auto-detect the BGP IP address.
            - name: IP
              value: "autodetect"             #<<------ Paste Below line
            - name: IP_AUTODETECTION_METHOD 
              value: "interface=eth0"
              
$ kubectl apply -f calico.yaml
$ kubectl get pods  -n kube-system
$ kubectl get pods  -o wide -n kube-system

$ ping 8.8.8.8
$ route -n
$ kubectl -n kube-system exec -it calico-node-4tbbl -- bash
  $ calico-node -show-status
  $ calico-node -bird-live
  $ calico-node -bird-ready

$ kubectl -n kube-system exec -it calico-node-4tbbl -- calico-node -bird-ready

$ cd /etc/kubernetes/
[root@MASTER kubernetes]# ls
admin.conf  controller-manager.conf  kubelet.conf  manifests  pki  scheduler.conf  super-admin.conf

vi admin.conf

## On Windows GitBash
$ kubectl get pods --kubeconfig k8s-cleint-vd.txt --insecure-skip-tls-verify


